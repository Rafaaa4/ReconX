# ReconX — Fiche projet et architecture

Oct 28, 2026 · @rafa3

## Définition du projet

ReconX est une plateforme distribuée qui prend l'URL d'un site que l'utilisateur a le droit de tester, puis analyse ses technologies exposées, ses sous-domaines et sa configuration de sécurité HTTP/TLS pour produire un rapport de risque.

**Titre :** ReconX — Distributed Web Security Assessment Platform

**Problématique :** comment centraliser et corréler des analyses de sécurité web dans une architecture orientée services, sans que la plateforme devienne elle-même un outil d'attaque ou un point d'entrée vers des réseaux internes ?

**Objectifs**

- Collecter les informations publiques d'une cible autorisée : technologies, sous-domaines, DNS, en-têtes HTTP.
- Évaluer la configuration HTTP et TLS et classer les résultats par gravité, avec un niveau de confiance et une preuve.
- Générer un rapport de sécurité, consultable aussi par un système tiers via SOAP.
- Démontrer une architecture SOA : 3 services métier et une API Gateway, avec REST, gRPC et SOAP, une base par service, Docker Compose et une résilience démontrée.

**Pile technique :** Spring Boot 3.3.x, Spring Cloud Gateway, MySQL, Docker Compose, Resilience4j.

## Workflow et périmètre

La première version se limite à la reconnaissance passive et à l'analyse de configuration ; les tests actifs viennent ensuite, uniquement dans un laboratoire dédié ou sur des cibles dont le test est explicitement autorisé.

Pour chaque scan, le backend exécute ce workflow :

1. Vérification du domaine et du périmètre autorisé.
2. Reconnaissance des technologies et des métadonnées publiques.
3. Découverte de sous-domaines à partir de sources publiques autorisées.
4. Analyse des configurations de sécurité HTTP et TLS.
5. Corrélation des résultats et évaluation du risque.
6. Génération d'un rapport détaillé.

Un résultat n'est pas forcément une vulnérabilité. ReconX distingue trois niveaux :

| Niveau | Définition | Exemple |
| --- | --- | --- |
| Observation | Information constatée, sans jugement | Un en-tête révèle le serveur utilisé |
| Finding | Problème potentiel à revoir | Un en-tête de sécurité est absent |
| Vulnérabilité confirmée | Faille prouvée par un test adapté et autorisé | Résultat d'un test actif en laboratoire |

La gravité d'un finding dépend du type de problème, du contexte et des preuves disponibles, pas du nombre d'alertes. Une technologie détectée est stockée avec une version `null` tant qu'aucune preuve fiable n'existe, plus un niveau de confiance et le type de preuve.

Une URL accessible sur Internet ne donne pas le droit de la tester : chaque scan exige une cible dont l'utilisateur est propriétaire ou pour laquelle il dispose d'une autorisation claire.

## Architecture globale

&#91;embedded content: architecture ReconX · gateway, 3 services, 3 bases\]

Le client ne parle qu'au Gateway ; Reconnaissance appelle Security Assessment en gRPC puis Security Reporting en SOAP, et chaque service écrit dans sa propre base.

## Services et choix des protocoles

Chaque service métier est une application Spring Boot 3 qui expose un seul style de contrat, choisi pour une raison technique précise.

| Service | Protocole | Rôle | Pourquoi ce protocole | Base |
| --- | --- | --- | --- | --- |
| Reconnaissance | REST | Valide la cible, collecte DNS, sous-domaines et en-têtes HTTP, détecte les technologies, orchestre le workflow | API simple pour le frontend : ressources `/api/v1/scans`, codes HTTP, OpenAPI | `reconx_recon_db` |
| Security Assessment | gRPC | Analyse les en-têtes de sécurité, cookies, HTTPS/TLS et indices CSRF, classe les findings par gravité | Contrat typé en `.proto`, appel interne efficace, streaming des findings | `reconx_security_db` |
| Security Reporting | SOAP | Produit le rapport, garde l'historique, résume le risque par cible | Interopérabilité avec un système tiers qui exige un contrat XML/WSDL | `reconx_reporting_db` |
| API Gateway | REST (Spring Cloud Gateway) | Point d'entrée unique : routage, JWT, rate limiting, correlation ID, masque les adresses internes | Un seul point d'entrée pour le frontend | — |

Les contrats à livrer :

- **REST :** `POST /api/v1/scans` répond `202 Accepted` avec un `scanId` et le statut `QUEUED`, car le scan tourne en arrière-plan. Les erreurs renvoient 400, 403 ou 404.
- **gRPC :** `security_assessment.proto` définit `AnalyzeTarget` (unaire) et `StreamFindings` (server streaming), avec des deadlines côté client et les statuts `INVALID_ARGUMENT` et `DEADLINE_EXCEEDED`.
- **SOAP :** contrat écrit d'abord en XSD, WSDL généré avec Spring Web Services, opérations `GetSecurityReport` et `GetRiskSummary`, et un SOAP Fault si le `scanId` est inconnu ou l'accès refusé.

## Communication entre services

Trois appels inter-services réels, sur trois contrats différents, et un seul service (Reconnaissance) qui orchestre le workflow, ce qui évite toute dépendance circulaire.

| Flux | Chemin | Protocole | Objet |
| --- | --- | --- | --- |
| 1 | Client → Gateway → Reconnaissance → Security Assessment | REST puis gRPC `AnalyzeTarget()` | Analyser la configuration de sécurité après la collecte initiale |
| 2 | Reconnaissance → Security Reporting | SOAP via `WebServiceTemplate` | Obtenir le résumé du rapport au format XML |
| 3 | Security Assessment → Reconnaissance | gRPC server streaming `StreamFindings()` | Envoyer les findings un par un pendant l'analyse |

Security Assessment et Security Reporting n'appellent aucun autre service.

À trancher avant le code : comment Reporting reçoit les findings, et comment le Gateway expose en REST les routes `/api/v1/security/**` et `/api/v1/reports/**` alors que ces services parlent gRPC et SOAP.

## Données, déploiement, résilience et sécurité

- **Données :** une base MySQL par service (`reconx_recon_db`, `reconx_security_db`, `reconx_reporting_db`), avec des identifiants distincts et des droits limités. Aucun service ne lit la base d'un autre ; `scan_id` est une référence logique, pas une clé étrangère.
- **Déploiement :** `docker compose up --build` démarre le gateway, les 3 services et MySQL sur un réseau interne, avec des volumes persistants. Dockerfiles multi-stage, utilisateur non-root, healthchecks, secrets hors des images (variables d'environnement, `.env.example` sans valeurs).
- **Résilience :** Resilience4j sur les clients gRPC et SOAP : timeout par appel réseau, retry limité avec backoff, circuit breaker, et un fallback qui enregistre le scan en `PARTIAL` si Security Assessment est indisponible. Démonstration : arrêter ce service, lancer un scan, montrer que le Gateway ne reste pas bloqué, puis redémarrer le service.
- **Sécurité de la plateforme :** avant tout scan, validation de l'URL et du périmètre autorisé. Blocage de localhost et des IP privées, revalidation DNS et de chaque redirection (protection SSRF), ports limités, scanner dans un réseau isolé. En bonus, JWT vérifié au Gateway et rôles ADMIN, ANALYST et VIEWER ; chaque scan est lié à son créateur, donc connaître un `scanId` ne suffit pas pour lire un rapport.
- **Tests :** JUnit 5 et Mockito pour l'unitaire ; tests d'intégration REST, gRPC et SOAP ; Testcontainers si le temps le permet.

## Déroulement d'un scan

&#91;embedded content: déroulement d'un scan · 7 étapes, 1 décision\]

Le client reçoit 202 dès que la cible est validée ; si Security Assessment ne répond pas dans le délai, le scan se termine en PARTIAL au lieu d'échouer.

## Endpoints REST

Le Reconnaissance Service expose six endpoints sous `/api/v1/scans` ; le Gateway route chaque préfixe vers le service concerné.

| Méthode | Endpoint | Fonction |
| --- | --- | --- |
| POST | `/api/v1/scans` | Créer un scan |
| GET | `/api/v1/scans/{id}` | Afficher l'état du scan |
| GET | `/api/v1/scans/{id}/assets` | Lister les assets découverts |
| GET | `/api/v1/scans/{id}/technologies` | Lister les technologies détectées |
| GET | `/api/v1/scans/{id}/subdomains` | Lister les sous-domaines |
| GET | `/api/v1/scans/{id}/headers` | Afficher les en-têtes HTTP |

Le `POST` répond `202 Accepted`, puisque le scan continue en arrière-plan. Autres codes à tester : `400` pour une URL invalide, `404` pour un scan inconnu, `403` pour un utilisateur non autorisé. L'OpenAPI doit refléter ces endpoints.

Format de la requête et de la réponse (exemple de contrat, pas un scan exécuté) :

```http
POST /api/v1/scans
Content-Type: application/json

{
  "target": "https://example.com",
  "scanProfile": "PASSIVE"
}
```

```json
{
  "scanId": "scan-001",
  "status": "QUEUED",
  "target": "example.com",
  "profile": "PASSIVE"
}
```

Routage du Gateway :

- `/api/v1/scans/**` vers Reconnaissance Service
- `/api/v1/security/**` vers Security Assessment Service
- `/api/v1/reports/**` vers Security Reporting Service

## Modèle de données

Chaque base contient les tables d'un seul service ; `scan_id` relie les bases par une référence logique, sans clé étrangère.

| Service | Table | Champs clés |
| --- | --- | --- |
| Reconnaissance | `scans` | `id` (UUID), `target_url`, `status`, `scan_profile`, `created_at`, `completed_at` |
| Reconnaissance | `assets` | `id`, `scan_id`, `asset_type`, `asset_value`, `source`, `confidence` |
| Security Assessment | `security_findings` | `id`, `scan_id`, `category`, `severity`, `title`, `description`, `evidence_summary`, `status`, `created_at` |
| Security Reporting | `security_reports` | `id`, `scan_id`, `overall_risk`, `findings_count`, `report_format`, `created_at` |

`asset_type` vaut `SUBDOMAIN`, `TECHNOLOGY`, `DNS_RECORD` ou `HTTP_ENDPOINT`. Une technologie sans preuve fiable garde une version `null`. Chaque service se connecte avec son propre compte MySQL, limité à sa base, jamais avec root.

## Structure du dépôt et des packages

Un dépôt Git unique contient les quatre applications Spring Boot, les contrats partagés et le fichier Compose.

```text
reconx/
├── api-gateway/
├── reconnaissance-service/
├── security-assessment-service/
├── security-reporting-service/
├── contracts/
│   ├── grpc/security_assessment.proto
│   └── soap/security-report.xsd
├── tests/integration/
├── docker-compose.yml
├── .env.example
└── README.md
```

Chaque service a son `pom.xml`, son `Dockerfile` et son dossier `src`. Dans un service, les packages (exemple : `com.reconx.reconnaissance`) séparent les responsabilités :

| Package | Rôle |
| --- | --- |
| `controller` | Endpoints REST, sans logique métier |
| `service` | Logique métier |
| `domain` | Entités |
| `repository` | Accès à la base |
| `dto` | Objets d'entrée et de sortie |
| `client` | Clients gRPC et SOAP |
| `config` | Configuration technique |
| `exception` | Gestion centralisée des erreurs |
| `security` | Validation du périmètre et contrôle d'accès |

## Déploiement Docker Compose

`docker compose up --build` démarre cinq conteneurs sur un réseau interne ; l'ordre ci-dessous est proposé, avec des conditions de readiness et non un simple ordre de lancement.

| Conteneur | Rôle | Démarre quand | Données |
| --- | --- | --- | --- |
| `mysql` | Une instance, trois bases | En premier | Volume persistant |
| `security-assessment-service` | gRPC | `mysql` est sain | `reconx_security_db` |
| `security-reporting-service` | SOAP | `mysql` est sain | `reconx_reporting_db` |
| `reconnaissance-service` | REST et orchestration | `mysql` est sain | `reconx_recon_db` |
| `api-gateway` | Point d'entrée unique | Les trois services sont sains | Aucune |

Reconnaissance n'attend pas Security Assessment pour démarrer : il doit survivre à son absence, ce que la démonstration de résilience montre.

- **Images :** Dockerfile multi-stage, une étape Maven pour construire le JAR puis une image d'exécution légère, avec Java lancé par un utilisateur non-root.
- **Configuration :** ports et identifiants lus depuis des variables d'environnement, avec des comptes MySQL distincts par service. `.env.example` ne contient que les noms (`MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `JWT_SECRET`), jamais de vraies valeurs.
- **Santé et logs :** healthchecks sur chaque service, Actuator, logs sur stdout et stderr.
- **Réseau :** les requêtes externes entrent par le Gateway ; les services et MySQL restent sur le réseau interne.

## Résilience et sécurité en détail

La résilience protège les appels gRPC et SOAP quand un service tombe ; la sécurité empêche d'utiliser ReconX contre une cible non autorisée ou un réseau interne.

**Résilience (Resilience4j, sur les clients gRPC et SOAP)**

| Mécanisme | Règle |
| --- | --- |
| Timeout | Chaque appel réseau a un délai maximal ; côté gRPC, un deadline qui donne `DEADLINE_EXCEEDED` |
| Retry | Limité, avec backoff, et seulement pour les opérations sans effet de bord |
| Circuit breaker | Coupe les appels répétés tant que le service est indisponible |
| Fallback | Le scan passe en `PARTIAL`, avec la mention que l'analyse de sécurité n'est pas terminée |

**Sécurité de la plateforme**

| Mesure | Détail |
| --- | --- |
| Autorisation de la cible | URL valide, cible dans le périmètre autorisé, limites sur le nombre de requêtes et la durée du scan |
| Adresses interdites | localhost, IP privées et réseaux internes refusés |
| Revalidation DNS | Vérification avant la connexion, sur chaque IP réellement contactée et pas seulement sur le hostname |
| Redirections | Désactivées, ou destination revalidée à chaque redirection |
| Ports | Liste de ports autorisés |
| Isolation réseau | Services de scan dans un réseau isolé, sans accès aux bases ni aux services internes inutiles |
| Accès | JWT validé au Gateway (bonus), rôles ADMIN, ANALYST et VIEWER ; chaque service revérifie les droits sur les opérations sensibles |
