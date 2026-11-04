# Inventaire initial — Otheloo Job Research Sandbox

Date : 4 novembre 2026. Les commits amont sont épinglés dans les sous-modules.

## Dépôt hôte

- Dépôt : https://github.com/digitaleflex/job_ob
- Branche de recherche : `sandbox/othello-job-research-2026-11-04`
- Branche `main` non modifiée.
- État initial observé : un seul `README.md` de 8 octets.

## 1. freehire — priorité architecture globale

- Dépôt : https://github.com/strelov1/freehire
- Commit : `3f0c2015de2bf800bb8ed6e3d56d9276e1413aa0`
- Licence MIT présente.
- Stack déclarée : Go, Fiber, PostgreSQL, pgvector, sqlc, Meilisearch, Redis, SvelteKit, Docker Compose.
- Structure : `cmd/`, `internal/`, `migrations/`, `services/`, `web/`, `extension/`, `design-system/`, `docs/`.
- Intérêt : catalogue d'entreprises/boards, registre de providers, pipeline fetch → normalize → deduplicate → upsert, workers séparés, recherche et API.
- Arborescence API : 11 313 blobs suivis, environ 96,9 MB de contenu ; dépôt lourd.
- Le README annonce 4 785 488 offres ouvertes, 701 972 entreprises et 421 sources au 30 octobre 2026. Ces chiffres sont déclarés par le projet, pas audités indépendamment.
- Pour Otheloo : étudier les interfaces de sources, le catalogue, l'idempotence, la déduplication, le cycle de vie des offres et les contrats API. Ne pas reprendre tout le produit.

## 2. JobScraper — référence Nigeria et multi-sources

- Dépôt : https://github.com/femakin/JobScraper
- Commit : `c08e3ddd4ce18f30e2cd65d11f1360dc728d2d40`
- Aucun fichier `LICENSE` détecté dans l'arborescence consultée : sous-module pour consultation ; pas de copie/réutilisation de son code avant confirmation des droits.
- Stack déclarée : TypeScript/Next.js, AWS Lambda/EventBridge, Twilio/WhatsApp et scoring OpenAI.
- README : 15 sources orientées remote et développeurs frontend compatibles avec le Nigeria, dont Jobberman, MyJobMag, Adzuna, Remotive, RemoteOK, Jobicy et d'autres.
- Intérêt : collecteurs hétérogènes, filtres géographiques, fraîcheur, scoring et notifications.
- Limites : cible spécialisée ; 15 sources annoncées ne signifient pas une couverture générale des pays et métiers africains.
- Pour Otheloo : étudier les interfaces et filtres ; ne pas reprendre les règles de ciblage Nigeria/frontend telles quelles.

## 3. Kenyan Job Finder — adaptateurs de job boards kényans

- Dépôt : https://github.com/markbosire/kenyan-job-finder
- Branche amont : `master`
- Commit : `4754080f753346153e2b1c60805771600fe397d6`
- Aucun fichier `LICENSE` détecté : sous-module de consultation ; pas de réutilisation du code sans autorisation.
- Stack observée : Python, scrapers dédiés, scripts de consolidation et documentation par source.
- Intérêt : granularité des adaptateurs locaux et méthodes propres aux job boards kényans.
- Limites : couverture Kenya ; dépendances navigateur et sélecteurs susceptibles de casser.
- Pour Otheloo : modèle de connecteur local à étudier pour l'adapter aux portails du Bénin, Togo, Sénégal et Côte d'Ivoire.

## 4. Job-Sense — pipeline data et matching

- Dépôt : https://github.com/declerke/Job-Sense
- Commit : `9400efec63efbb5f39cdd55aec97b24e7f01c501`
- Licence MIT présente.
- Stack : Python, FastAPI, Airflow, dbt, PostgreSQL, pgvector, scrapers et interface de démonstration.
- Structure : `api/`, `scrapers/`, `pipeline/`, `dags/`, `dbt/`, `tests/`, `frontend/`.
- Intérêt : séparation collecte/traitement/API, orchestration, données analytiques, recherche sémantique et matching CV/offre.
- Limites : maturité et volume réel à mesurer ; ne pas ajouter toute sa stack si Spring Boot et PostgreSQL couvrent déjà le besoin.
- Pour Otheloo : reprendre des composants de pipeline et de matching de manière ciblée.

## 5. Nairobi Data Jobs Tracker — pipeline opérationnel compact

- Dépôt : https://github.com/declerke/Nairobi-Data-Jobs-Tracker
- Commit : `8189f0b546924a6754888abadd34500342487441`
- Licence MIT présente.
- Stack : Python 3.11, Playwright, requests, BeautifulSoup, PostgreSQL, Airflow, Docker et Metabase.
- Sources documentées : BrighterMonday Kenya, MyJobMag Kenya et Fuzu Kenya.
- Intérêt : stratégie HTTP puis navigateur, normalisation, scoring, logs de scraping, orchestration, SQL et notifications.
- Le README annonce environ 235 offres par exécution et un pipeline quotidien ; chiffres déclarés par l'auteur.
- Pour Otheloo : bon exemple pour étudier qualité des données, logs, idempotence et planification. Pas un scraper de production garanti.

## Comparaison initiale

| Projet | Couverture | Valeur principale | Risque/limite |
|---|---|---|---|
| freehire | Internationale, ATS et boards | Architecture d'agrégation | Gros dépôt, complexité élevée |
| JobScraper | Nigeria + remote | Collecteurs et scoring | Licence absente, niche frontend |
| Kenyan Job Finder | Kenya | Adaptateurs locaux | Licence absente, fragilité des sélecteurs |
| Job-Sense | Kenya / démonstrateur | Pipeline et matching | Maturité/volume à vérifier |
| Nairobi Tracker | Nairobi / data jobs | Pipeline compact et monitoring | Trois sources, spécialisation métier |

## Hypothèses à tester pour Otheloo

1. Spring Boot demeure le cœur métier : API, auth, autorisations, offres, entreprises, candidats et publication.
2. Les collecteurs peuvent être des workers Python autonomes derrière un contrat d'ingestion stable ; ne pas créer un microservice par source.
3. Un registre doit décrire la source, le pays, la méthode autorisée, la fréquence, les limites, le statut, le dernier succès et les droits d'usage.
4. Modèle canonique : titre, entreprise, lieu, pays, description, type de contrat, salaire/devise, dates, URL originale, source, identifiants externes, statut et horodatages.
5. Déduplication en plusieurs étapes : URL canonique, identifiant externe, puis empreinte entreprise/titre/localisation/description.
6. Gérer explicitement la fraîcheur et l'expiration : une offre disparue ne doit pas rester active indéfiniment.
7. Ajouter OpenSearch/Meilisearch, pgvector, Airflow ou RabbitMQ uniquement si les mesures le justifient.

## Protocole d'audit approfondi

Pour chaque projet et connecteur : lire README/licence/dépendances ; cartographier les points d'entrée ; relever domaines et endpoints contactés ; analyser pagination, retries, backoff, délais et gestion des erreurs ; suivre brut → parsing → normalisation → validation → déduplication → stockage ; inspecter secrets/cookies/logs ; exécuter seulement isolé, sans secrets réels, après revue ; mesurer taux de succès, couverture, doublons, fraîcheur, coût et charge ; produire une décision reprendre/adapter/réécrire/écarter.

## Licences et conformité

- Conserver les licences et avis de copyright.
- Pour JobScraper et Kenyan Job Finder, aucune licence n'a été trouvée à l'inventaire : consultation possible, mais pas de copie/réutilisation sans autorisation du titulaire.
- La licence du scraper ne donne pas le droit d'extraire les données d'un site tiers.
- Respecter conditions d'utilisation, règles d'accès, limites de requêtes et obligations de protection des données.
