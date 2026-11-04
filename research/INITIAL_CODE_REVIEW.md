# Première revue de code — Otheloo Job Research

Date : 4 novembre 2026  
Périmètre : lecture statique ciblée des points d'entrée et composants structurants des cinq sous-modules.  
Statut : **première passe, non exhaustive**. Aucun scraper n'a été exécuté et aucune mesure de rendement n'a été reproduite.

## Synthèse exécutive

La meilleure source d'inspiration pour l'architecture d'ingestion demeure **freehire**. Les deux projets kényans sont surtout utiles pour comprendre les structures réelles des sites régionaux et les différences entre extraction HTTP et navigateur. **JobScraper** illustre un registre simple et un pipeline de filtrage/scoring, mais sa logique métier est très orientée développeurs frontend au Nigeria. **Job-Sense** apporte un modèle de données typed et un upsert batch intéressant.

Il ne faut pas fusionner ces cinq applications. Pour Otheloo, l'approche recommandée est de concevoir **un contrat canonique de connecteur et un pipeline d'ingestion propre**, puis d'implémenter les sources prioritaires une par une. Spring Boot reste le propriétaire du domaine Emploi ; des collecteurs Python isolés peuvent alimenter cette API si l'écosystème Python apporte un avantage concret. N'introduire RabbitMQ/Airflow/pgvector qu'après mesure.

---

## 1. freehire — architecture la plus robuste, mais complexe

### Ce qui est réellement intéressant dans le code

- `internal/ingest/sources/source.go` définit un contrat minimal `Source` avec `Provider()` et `Fetch(ctx, CompanyEntry)`.
- `CompanyEntry` distingue l'entreprise, le provider, l'identifiant de board, la région, les hubs et les correspondances de tenants. C'est utile lorsqu'une plateforme ATS expose plusieurs entreprises ou régions.
- Le type `Job` sépare les données brutes des signaux structurés. Les champs de seniority, catégorie, type d'emploi, skills, expérience et mode de travail ne doivent être remplis que lorsque la plateforme fournit une donnée structurée. Le pipeline garde ainsi la provenance et évite de transformer une intuition en fait.
- `internal/ingest/pipeline/pipeline.go` ne dépend ni d'une base précise ni d'un provider : le pipeline reçoit un registre d'adaptateurs et un contrat `Store`.
- Le stockage expose des capacités optionnelles étroites (`FormSaver`, `Closer`, `Toucher`, `SeenLookup`, `CoveredRefresher`) au lieu d'imposer une interface gigantesque à tous les stores.
- Le runner fixe une concurrence par défaut de 8 boards et un heartbeat de progression de 60 secondes.
- `internal/ingest/sources/registry.go` formalise plusieurs comportements d'adaptateurs : catalogue complet, flux auto-closing, listings partiels, grâce de fermeture et liveness probing. C'est une bonne réponse au problème des offres qui disparaissent temporairement lorsqu'un crawl est incomplet.

### Ce qu'Otheloo devrait reprendre comme principes

1. Un contrat d'adaptateur stable, indépendant du stockage.
2. Un registre déclaratif des sources et des boards.
3. Une distinction entre champs extraits, champs structurés fournis par la source et champs inférés.
4. Des règles de fermeture différentes selon la couverture garantie par chaque source.
5. Des écritures idempotentes et un historique de runs.

### Ce qu'il ne faut pas reprendre tel quel

- L'ensemble de freehire : il s'agit d'une plateforme de recrutement très complète, pas simplement d'une bibliothèque de scraping.
- Son volume de packages, ses migrations et ses composants de recherche ne justifient pas automatiquement la même complexité pour Otheloo.
- Le dépôt est important (environ 96,9 MB de blobs et 11 313 fichiers selon l'arbre GitHub interrogé). Il faut lire les sous-systèmes utiles au lieu de tout transposer.

**Décision provisoire :** référence architecturale prioritaire ; adapter les concepts, ne pas porter le code entier.

---

## 2. JobScraper — modèle multi-sources simple, mais trop spécialisé

### Observations du code

- `src/lib/scrapers/registry.ts` définit une liste explicite de 15 sources, avec `id`, nom, site, fonction de scraping, état activé et option de bypass du filtre.
- `src/lib/scrapers/index.ts` exécute les sources séquentiellement, ajoute une pause d'une seconde entre elles, consigne les résultats, filtre les nouveautés, puis regroupe les nouvelles offres avant scoring IA et notifications.
- `src/lib/dedup.ts` calcule un SHA-256 sur `title + company + url`.
- Ce hash est trop dépendant de l'URL pour une vraie déduplication inter-sites : deux copies de la même annonce avec deux URLs différentes ne seront probablement pas fusionnées. Il manque une seconde étape de similarité basée sur titre, entreprise, localisation et description.
- `runScraper` supprime les lignes de `scrape_runs` d'une source avant de recréer la ligne du run. Cela donne un état courant, mais ne constitue pas un historique fiable des exécutions.
- `filter.ts` impose des règles strictes de métier, de pays et de fraîcheur, avec rejet des offres sans date par défaut. C'est cohérent pour son cas d'usage, pas pour un agrégateur panafricain généraliste.
- Le scraper Jobberman s'appuie sur une longue liste de requêtes de mots-clés centrées sur le développement frontend et déduplique localement par slug.
- Le README mentionne Supabase, OpenAI, Twilio/WhatsApp et un déclenchement planifié AWS.

### Risques et limites pour Otheloo

- Le périmètre est ciblé : développeurs frontend et offres compatibles avec le Nigeria/remote.
- Les filtres de localisation ne doivent pas être repris tels quels pour d'autres pays.
- L'IA n'est pas nécessaire dans le chemin critique d'ingestion ; scoring et alertes peuvent devenir coûteux et créer une dépendance externe.
- Le registre et l'isolation des sources sont intéressants, mais il faut ajouter timeout, retry/backoff borné, métriques et historique durable.

**Décision provisoire :** reprendre l'idée du registre et de la séparation `scrape → filter → dedup → enrich → notify`, mais corriger l'identité canonique et retirer les hypothèses Nigeria/frontend.

**Licence :** aucun fichier `LICENSE` n'a été trouvé dans l'arborescence consultée. Sous-module conservé comme référence ; ne pas copier/réutiliser son code avant clarification des droits.

---

## 3. Kenyan Job Finder — source utile pour les adaptateurs locaux, avec précautions

### Observations du code

- `scrapers/base.py` propose une classe commune `ScraperBase`, des filtres par mots-clés, une fusion de résultats par `job_id`, et la persistance CSV/pickle.
- Le pipeline assemble plusieurs scrapers kényans et JobSpy.
- `run.py` modifie à l'exécution l'énumération `Country` de la dépendance JobSpy pour ajouter Kenya. Cette monkey-patch est fragile : une mise à jour de JobSpy peut casser le comportement.
- `scrapers/_playwright_common.py` s'appuie sur un navigateur local via CDP sur le port 9222 et contient des routines de gestion de processus. Ce couplage à un navigateur préexistant rend les exécutions difficiles à isoler dans un worker de production.
- La classe de base utilise `pickle.load` pour restaurer des données. Le pickle ne doit jamais être chargé depuis une source non fiable : la désérialisation peut exécuter du code.

### Ce qu'Otheloo devrait en retenir

- Les adaptations sont spécifiques à chaque portail : il faut un connecteur par famille de site et des tests de parsing avec fixtures HTML.
- Une base abstraite de scraper doit gérer timeouts, retries contrôlés, pagination, validation et arrêt propre du navigateur.
- Préférer JSON ou stockage DB à la persistance pickle.
- Éviter de patcher une dépendance externe à l'exécution ; encapsuler l'adaptation pays dans notre propre code.

**Décision provisoire :** référence pour découvrir les particularités des job boards d'Afrique de l'Est ; ne pas reprendre son runner ni sa gestion navigateur telle quelle.

**Licence :** aucun fichier `LICENSE` n'a été trouvé dans l'arborescence consultée. Sous-module conservé comme référence ; ne pas copier/réutiliser son code avant clarification des droits.

---

## 4. Job-Sense — modèle de données et upsert batch intéressants

### Observations du code

- `scrapers/base_scraper.py` fournit un `JobData` typé : source, external_id, entreprise, localisation, description, bornes de salaire, devise, contrat, expérience, remote, URL, tags et dates.
- `BaseScraper` partage une session HTTP et une méthode de récupération avec timeout et retries exponentiels.
- `scrapers/runner.py` possède un registre de scrapers, déduplique le lot par `(source, external_id)` et réalise un upsert batch PostgreSQL via `execute_values`.
- La contrainte `ON CONFLICT (source, external_id)` rend l'import idempotent à l'intérieur d'une source.
- Le pipeline de nettoyage applique des catégories de contrat et d'expérience, détecte les signaux remote et extrait des tags d'un vocabulaire fixe.
- Le module CV matching calcule des similarités vectorielles et peut envoyer un extrait du CV à Anthropic pour une analyse complémentaire.

### Limites pour Otheloo

- La clé `(source, external_id)` n'élimine pas les doublons entre deux sources différentes : un passage de déduplication global est nécessaire.
- La devise par défaut KES et les valeurs de localisation par défaut orientent le projet vers le Kenya. Otheloo doit utiliser le pays/devise de la source et ne jamais inventer une devise.
- Le scoring lexical/remote par mots-clés peut créer des faux positifs ; il faut conserver les valeurs brutes et la provenance.
- L'envoi de CV à un fournisseur LLM impose consentement explicite, minimisation, durée de conservation, sécurité et information claire du candidat.

**Décision provisoire :** bon modèle pour les DTO canoniques, le registre et l'upsert batch ; renforcer la déduplication inter-sources et la normalisation géographique.

---

## 5. Nairobi Data Jobs Tracker — bon laboratoire pipeline, pas une base panafricaine

### Observations du code

- Trois connecteurs sont réellement câblés dans `run.py` : BrighterMonday, MyJobMag et Fuzu, tous dans le contexte kényan.
- Le README décrit une stratégie mixte : Playwright pour les pages SPA, requests/BeautifulSoup pour les pages statiques.
- `setup_database.sql` crée une table d'offres, des logs de scraping, des logs email, des index, triggers et vues analytiques.
- La contrainte d'unicité principale est sur `posting_url`, utile contre les répétitions exactes, mais insuffisante pour les URLs avec paramètres de suivi ou copies inter-sites.
- La colonne `source` est limitée par un CHECK à trois noms et `Other`. Cette contrainte ne passe pas à l'échelle d'un registre de sources extensible.
- La normalisation de localisation retombe sur `Kenya` si la source ne fournit rien ; l'extraction de salaire est spécifique au KES. Ces valeurs ne sont pas transposables à l'Afrique entière.
- `posted_date` est stocké comme DATE, ce qui perd l'heure et la précision de fuseau.
- Dans `utils/helpers.py`, `kwargs.pop('headers', self.get_headers())` se trouve dans la boucle de retry. Si des headers sont fournis au départ et que la première tentative échoue, ils sont retirés de `kwargs` ; les tentatives suivantes peuvent utiliser les headers par défaut. C'est un défaut concret à corriger avant réutilisation.
- `run.py` initialise la base, crée les tables et écrit des logs de run avant le contrôle `dry_run`. Le mode annoncé comme dry-run n'est donc pas totalement sans effet sur la base.
- Le code enregistre `jobs_new=len(jobs)` au niveau du log source avant de connaître les vrais nouveaux enregistrements ; cette métrique peut surestimer les nouveautés.

### Décision provisoire

Très bon exemple de petit pipeline complet et de suivi opérationnel. À utiliser pour apprendre la structure d'un collecteur, la journalisation, les tests et les vues de statistiques ; réécrire le modèle multi-pays et les cycles de vie des annonces.

---

## Architecture cible suggérée pour Otheloo

### Contrat d'adaptateur

Chaque connecteur devrait exposer une entrée uniforme :

- `sourceCode`, `countryCode`, `sourceUrl`, `accessMethod`, `schedule`, `rateLimit`, `termsReviewedAt`, `licenseStatus`, `enabled`.
- `fetchPage/crawl` retourne des annonces brutes et des métadonnées de pagination.
- Le collecteur ne décide pas de la publication finale ; il transmet le brut à un pipeline commun.

### Modèle canonique minimal

| Champ | Règle |
|---|---|
| sourceId + externalId | Identité stable dans une source |
| canonicalUrl | URL sans paramètres de tracking |
| title, company | Texte nettoyé, sans perte du brut |
| countryCode, city, locationRaw | Géographie structurée + valeur d'origine |
| workplaceType | remote / hybrid / onsite / unknown |
| employmentType | full-time / part-time / contract / internship / unknown |
| salaryMin, salaryMax, currency, period | Valeurs séparées ; null si inconnu |
| publishedAt, deadlineAt, collectedAt | Instants UTC quand connus |
| descriptionRaw, descriptionText | Conservation contrôlée du brut et du texte |
| contentFingerprint | Aide à la déduplication inter-sources |
| status, lastSeenAt, closedAt | Cycle de vie de l'annonce |

### Pipeline recommandé

1. **Collecte** : connecteur isolé, limites, timeout, retry avec backoff et respect des règles du site.
2. **Validation** : schéma, URL, titre, entreprise, localisation, dates et longueur.
3. **Normalisation** : pays/devise/fuseau issus de la source ; pas de fallback arbitraire.
4. **Déduplication intra-source** : external ID puis URL canonique.
5. **Déduplication inter-sources** : empreinte normalisée puis similarité titre/entreprise/localisation/description ; garder les sources de provenance.
6. **Persistance idempotente** : upsert en lot et stockage de la source d'origine.
7. **Cycle de vie** : ne fermer une offre que lorsque la source a fourni une preuve de disparition fiable (crawl complet ou plusieurs absences consécutives selon la source).
8. **Observabilité** : compteurs found/parsed/valid/duplicate/upserted/closed/throttled/failed, temps de réponse, taux de succès et dernier succès par source.

### Intégration avec Spring Boot

- Spring Boot gère le domaine Emploi, les permissions, les entreprises, les candidats, les API et les candidatures.
- Le worker de scraping peut rester en Python lorsqu'un connecteur exige Playwright ou des bibliothèques Python.
- Commencer avec un endpoint d'ingestion authentifié et idempotent ou une file simple déjà présente dans Otheloo. Ne pas installer Airflow, RabbitMQ, Redis et OpenSearch en une seule fois sans métriques qui le justifient.
- Prévoir un mode `dry-run` réellement sans écriture, des fixtures HTML anonymisées et des tests par connecteur.

---

## Ordre de travail conseillé

1. Auditer d'abord le sous-système ingestion de freehire : contrat Source, registry, board catalog, pipeline, store, sweep, tests.
2. Comparer les adaptateurs BrighterMonday/Fuzu/MyJobMag de Job-Sense et Nairobi Tracker.
3. Analyser JobScraper comme exemple de registre, filtres et notifications, en isolant les problèmes de hash et de journalisation.
4. Examiner les adaptateurs du Kenyan Job Finder pour les sources régionales et documenter ceux qui restent juridiquement et techniquement exploitables.
5. Concevoir ensuite le contrat Otheloo, puis un premier connecteur pilote pour le Bénin.

## Fichiers examinés

- freehire : [source.go](https://github.com/strelov1/freehire/blob/3f0c2015de2bf800bb8ed6e3d56d9276e1413aa0/internal/ingest/sources/source.go), [registry.go](https://github.com/strelov1/freehire/blob/3f0c2015de2bf800bb8ed6e3d56d9276e1413aa0/internal/ingest/sources/registry.go), [pipeline.go](https://github.com/strelov1/freehire/blob/3f0c2015de2bf800bb8ed6e3d56d9276e1413aa0/internal/ingest/pipeline/pipeline.go), [store.go](https://github.com/strelov1/freehire/blob/3f0c2015de2bf800bb8ed6e3d56d9276e1413aa0/cmd/ingest/store.go).
- JobScraper : [registry.ts](https://github.com/femakin/JobScraper/blob/c08e3ddd4ce18f30e2cd65d11f1360dc728d2d40/src/lib/scrapers/registry.ts), [index.ts](https://github.com/femakin/JobScraper/blob/c08e3ddd4ce18f30e2cd65d11f1360dc728d2d40/src/lib/scrapers/index.ts), [dedup.ts](https://github.com/femakin/JobScraper/blob/c08e3ddd4ce18f30e2cd65d11f1360dc728d2d40/src/lib/dedup.ts), [filter.ts](https://github.com/femakin/JobScraper/blob/c08e3ddd4ce18f30e2cd65d11f1360dc728d2d40/src/lib/filter.ts).
- Kenyan Job Finder : [base.py](https://github.com/markbosire/kenyan-job-finder/blob/4754080f753346153e2b1c60805771600fe397d6/scrapers/base.py), [run.py](https://github.com/markbosire/kenyan-job-finder/blob/4754080f753346153e2b1c60805771600fe397d6/run.py), [_playwright_common.py](https://github.com/markbosire/kenyan-job-finder/blob/4754080f753346153e2b1c60805771600fe397d6/scrapers/_playwright_common.py).
- Job-Sense : [base_scraper.py](https://github.com/declerke/Job-Sense/blob/9400efec63efbb5f39cdd55aec97b24e7f01c501/scrapers/base_scraper.py), [runner.py](https://github.com/declerke/Job-Sense/blob/9400efec63efbb5f39cdd55aec97b24e7f01c501/scrapers/runner.py), [cleaner.py](https://github.com/declerke/Job-Sense/blob/9400efec63efbb5f39cdd55aec97b24e7f01c501/pipeline/cleaner.py), [cv_match.py](https://github.com/declerke/Job-Sense/blob/9400efec63efbb5f39cdd55aec97b24e7f01c501/api/routers/cv_match.py).
- Nairobi Tracker : [helpers.py](https://github.com/declerke/Nairobi-Data-Jobs-Tracker/blob/8189f0b546924a6754888abadd34500342487441/utils/helpers.py), [run.py](https://github.com/declerke/Nairobi-Data-Jobs-Tracker/blob/8189f0b546924a6754888abadd34500342487441/run.py), [setup_database.sql](https://github.com/declerke/Nairobi-Data-Jobs-Tracker/blob/8189f0b546924a6754888abadd34500342487441/setup_database.sql).

**Limitation de cette passe :** inspection statique ciblée des fichiers indiqués ; ni exécution, ni audit exhaustif de sécurité, ni vérification du comportement actuel des sites externes. Les constats doivent être confirmés par tests isolés avant toute réutilisation.
