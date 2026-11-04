# Otheloo — Job Research Sandbox

Ce bac à sable isole les projets de référence étudiés pour le moteur **Otheloo Emploi**.

## Contenu

Les cinq projets sont référencés comme **sous-modules Git**, chacun épinglé sur un commit précis. Leurs sources restent dans leurs dépôts d'origine : cela évite de mélanger les licences, de dupliquer les historiques et de modifier les projets amont.

| Dossier | Projet | Utilité principale |
|---|---|---|
| `vendor/freehire` | strelov1/freehire | Agrégation ATS, pipeline d'ingestion, normalisation, déduplication, recherche |
| `vendor/JobScraper` | femakin/JobScraper | Collecteurs multi-sources, ciblage Nigeria, scoring et alertes |
| `vendor/kenyan-job-finder` | markbosire/kenyan-job-finder | Adaptateurs de job boards kényans |
| `vendor/Job-Sense` | declerke/Job-Sense | Pipeline data, recherche et matching CV/offres |
| `vendor/Nairobi-Data-Jobs-Tracker` | declerke/Nairobi-Data-Jobs-Tracker | Scraping navigateur/HTTP, scoring, PostgreSQL et orchestration |

L'inventaire initial, les limites connues et le protocole d'analyse sont dans [research/REPOSITORY_INVENTORY.md](research/REPOSITORY_INVENTORY.md).

## Cloner le bac à sable avec tous les codes

Depuis un terminal avec Git :

```bash
git clone --branch sandbox/othello-job-research-2026-11-04 --recurse-submodules https://github.com/digitaleflex/job_ob.git otheloo-job-research
cd otheloo-job-research
git submodule status
```

Si le dépôt est déjà cloné sans les sous-modules :

```bash
git submodule update --init --recursive
```

## Règles du bac à sable

- Cette branche est une zone de recherche séparée ; elle ne modifie pas `main`.
- Les sous-modules sont épinglés sur des commits reproductibles. Ne pas les mettre à jour sans consigner le nouveau SHA.
- Ne jamais versionner de fichier `.env`, token, cookie, session navigateur ou donnée personnelle.
- Ne pas exécuter aveuglément les scrapers : vérifier dépendances, scripts d'installation, appels réseau, stockage et effets de bord.
- Respecter les licences, les conditions d'utilisation des sites, leurs règles d'accès et les limites de requêtes.
- Aucune technique de contournement d'authentification ou de protection anti-bot ne doit être considérée comme acceptable par défaut.

## Objectif

Identifier les composants réutilisables pour Otheloo : registre des sources, adaptateurs, planification, collecte, parsing, normalisation, déduplication, géolocalisation, détection d'expiration, contrôle qualité, observabilité, stockage et API d'intégration Spring Boot.
