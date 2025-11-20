# Traitement RNE de bout en bout

## 1. Résumé exécutif (1 page maximum)
Le pipeline RNE industrialise l’acquisition du stock initial et des flux quotidiens fournis par l’INPI, les valide via Pydantic, les normalise dans une base SQLite versionnée et les publie sur MinIO. La base RNE alimente ensuite l’ETL SIRENE : elle est attachée à la base SIRENE, puis les données dirigeant, immatriculation, unité légale et siège sont nettoyées (pandas), fusionnées (`UPDATE`/`INSERT OR IGNORE`) et indexées pour l’annuaire. Des contrôles volumétriques, du dédoublonnage, l’ignorance du dernier flux potentiellement incomplet et des notifications Mattermost garantissent que seules des versions cohérentes sont diffusées.

## 2. Vue d’ensemble du pipeline de traitement RNE
- Collecte du stock RNE via FTP (`get_stock.sh` + `RneStockProcessor`) et dépôt immédiat dans MinIO (`rne/stock`).
- Collecte quotidienne des flux `companies/diff` via `ApiRNEClient` orchestrée par le DAG Airflow `get_flux_rne`, avec pagination `searchAfter`, gestion des tokens et sauvegarde journalière en `.json.gz` sur MinIO (`rne/flux`).
- Constitution ou reprise d’une base SQLite locale `rne_<date>.db` : lecture du stock si reconstruction, ingestion ordonnée des flux (en ignorant le dernier fichier), validation Pydantic, mapping vers les tables normalisées puis dédoublonnage et contrôles de volumétrie.
- Versionnement et publication sur MinIO (`rne/database/` + `latest_rne_date.json`), suppression des artefacts locaux.
- Intégration dans l’ETL SIRENE : attachement de la base RNE, prétraitements pandas par blocs, mises à jour/copies SQL, régénération des tables dérivées puis indexation pour l’annuaire.
- Supervision continue par scheduling quotidien (1h/2h), nettoyage des répertoires temporaires et notifications Mattermost de succès/échec.

## 3. Architecture des composants
- **Airflow** : deux DAGs principaux (`get_flux_rne` pour les flux, `fill_rne_database` pour la base) avec nettoyage des dossiers temporaires, limites de runs actifs, `dagrun_timeout`, retries et notifications.
- **MinIO** : stockage objet unique pour le stock (`rne/stock`), les flux (`rne/flux`), les bases compressées (`rne/database`) et les métadonnées (`latest_rne_date.json`), piloté par des utilitaires (`get_files`, `send_files`, `get_latest_file`).
- **SQLite** : base locale `rne.db` puis attachement à `sirene.db`; création de tables/index, `INSERT OR IGNORE`, `VACUUM` et `SELECT DISTINCT` pour le dédoublonnage et la performance.
- **Pydantic** : modèles `RNECompany` et imbriqués pour valider chaque enregistrement JSON avant mapping interne (`UniteLegale`, `Siege`, etc.).
- **pandas** : prétraitements dirigeant/établissement (tri, déduplication, homogénéisation de la casse, agrégation des rôles) pour limiter les doublons avant insertion SIRENE.
- **Scripts shell** : `get_stock.sh` encapsule `lftp` pour rapatrier le stock RNE dans un répertoire temporaire avant envoi MinIO.
- **API RNE** : client `ApiRNEClient` gérant tokens SSO, `requests.Session` et adaptation dynamique de `pageSize` (100→5→1) selon les erreurs HTTP, avec pagination `searchAfter`.

## 4. Flux de données détaillé (end-to-end)
### Collecte stock + flux
- Stock initial récupéré par `RneStockProcessor.download_stock` via `get_stock.sh`, extraction des JSON, upload sur MinIO `rne/stock`, puis suppression locale.
- Flux quotidien via DAG `get_flux_rne` (1h) : nettoyage préalable, boucle de dates (dernière version → J-1), appel à `get_and_save_daily_flux_rne` qui journalise les pages JSON ND, compresse en `.gz`, stocke sur MinIO `rne/flux`, supprime le local et notifie Mattermost. En cas d’erreur, sauvegarde partielle puis suppression des fichiers corrompus pour forcer la reprise.

### Stockage MinIO
- Stock : préfixe `rne/stock/` (ou `ae/<ENV>/rne/stock/`).
- Flux : préfixe `rne/flux/` (`ae/<ENV>/rne/flux/`).
- Bases compressées et métadonnées : `rne/database/` pour `rne_<date>.db.gz` et `latest_rne_date.json`.

### Création / mise à jour de la base SQLite RNE
- `get_start_date_minio` lit `latest_rne_date.json`; si absent, reconstruction complète depuis le stock.
- `create_db` instancie `rne_<start_date>.db` (ou `get_latest_db` restaure la base J-1) puis `create_tables` crée les tables cibles.
- `process_stock_json_files` injecte le stock uniquement en cas de reconstruction. `process_flux_json_files` traite tous les flux disponibles sauf le dernier fichier, charge chaque `.json.gz` dans l’ordre chronologique et supprime les fichiers locaux.
- `insert_unites_legales_into_db` insère les données mappées (UL, sièges, dirigeants, immatriculations, établissements, activités) après validation Pydantic et upsert par SIREN.
- `remove_duplicates` rejoue des `SELECT DISTINCT` table par table, reconstruit les index puis lance `VACUUM`. `check_db_count` impose des seuils minimaux (UL/immatriculations, dirigeants PP/PM) et lève une exception si franchis.

### Versioning
- `upload_db_to_minio` recompresse la base en `.gz`, la publie sous `rne_<last_date_processed>.db.gz`, puis supprime les artefacts locaux.
- `upload_latest_date_rne_minio` écrit la nouvelle date de référence (date traitée +1) dans `latest_rne_date.json` pour la reprise future.

### Intégration dans SIRENE
- `get_rne_database` télécharge la dernière base RNE depuis MinIO, la décompresse et l’attache à la base SIRENE locale.
- `create_dirig_tables`, `copy_immatriculation_table`, `create_unite_legale_tables`, `create_etablissements_tables` orchestrent la copie/ mise à jour par blocs (chunks de 100 000 SIREN) avec prétraitements pandas et requêtes SQL (`UPDATE`, `INSERT OR IGNORE`, `SELECT DISTINCT`).
- Régénération des tables dérivées (ancien_siege, historique_unite_legale/etablissement, dates de fermeture) après enrichissement.

### Nettoyage / règles de gestion
- Upsert par SIREN : `find_and_delete_same_siren` supprime les anciennes lignes avant insertion.
- Dédoublonnage global post-injection + `VACUUM`.
- Contrôles volumétriques via `check_db_count` (planchers sur UL, dirigeants PP/PM, immatriculations).
- Ignorance du dernier fichier de flux et suppression des fichiers partiels en cas d’erreur.
- Sélection de la meilleure valeur (forme juridique, nature d’entreprise, adresses, rôles) via les helpers de mapping.

### Supervision
- Scheduling quotidien (1h pour flux, 2h pour base), `max_active_runs=1` et `dagrun_timeout`.
- Notifications Mattermost vertes (plages traitées) ou rouges (échecs), avec nettoyage systématique des répertoires temporaires.

## 5. Schéma fonctionnel du pipeline
```mermaid
flowchart TD
    A[FTP INPI - Stock ZIP] -->|get_stock.sh / RneStockProcessor| B[MinIO rne/stock/]
    A2[RNE API companies/diff - Flux J] -->|ApiRNEClient / get_and_save_daily_flux_rne| C[MinIO rne/flux/ (.json.gz)]
    B --> D[fill_rne_database]
    C --> D
    D -->|Validation Pydantic + mapping + dédup| E[SQLite rne_<date>.db]
    E -->|Compression + versioning| F[MinIO rne/database/ + latest_rne_date.json]
    F -->|get_rne_database| G[ETL SIRENE SQLite]
    G -->|pandas + SQL (UPDATE/INSERT)| H[Tables SIRENE enrichies]
    H --> I[Mattermost notifications]
```

## 6. Inputs & Outputs unifiés
### Inputs
- Stock historique INPI via FTP (`stock_rne.zip` → JSON) téléchargé dans `/tmp/rne/stock` puis MinIO `rne/stock/`.
- Flux quotidien INPI (`companies/diff`) en JSON ND compressé par jour (`rne_flux_YYYY-MM-DD.json.gz`) stocké dans `RNE_FLUX_DATADIR` puis MinIO `rne/flux/`.
- Métadonnées MinIO : `latest_rne_date.json` et base SQLite précédente `rne_<date>.db.gz`.

### Outputs
- Base SQLite RNE versionnée (`rne_<date>.db.gz`) publiée sur MinIO `rne/database/` avec archivage quotidien.
- Tables RNE normalisées : `unite_legale`, `siege`, `dirigeant_pp`, `dirigeant_pm`, `immatriculation`, `etablissement`, `activite` avec index.
- Tables SIRENE enrichies (dirigeants PP/PM, immatriculation, unite_legale, siege, historiques) après attachement de la base RNE.
- Notifications Mattermost de succès/échec.

## 7. Mapping Source → Cible
### Tables RNE
- **unite_legale** : siren, identité (dénomination ou nom/prénom/usage), nom commercial, dates de création/mise à jour, activité principale, tranche d’effectif, forme/nature juridique, état administratif, forme d’exercice, statut de diffusion, adresse formatée, provenance (`file_name`).
- **siege** : siren, siret du siège, enseigne/nom commercial, adresse complète (pays, codes, voie, complément), date de mise à jour, fichier source.
- **dirigeant_pp** : siren, date de mise à jour, identité (nom, nom usage, prénoms concaténés, genre, date de naissance, nationalité, situation matrimoniale), rôle, fichier source.
- **dirigeant_pm** : siren entreprise, siren du dirigeant, dénomination, rôle, pays, forme juridique, date de mise à jour, fichier source.
- **immatriculation** : siren, dates (mise à jour, immatriculation, radiation/effet/cessation priorisées), indicateur associé unique, capital social/variable, devise, durée, date de début d’activité, nature d’entreprise sérialisée, fichier source.
- **etablissement** : siren, siret, date de mise à jour, fichier source.
- **activite** : siren/siret, catégorie, indicateurs principal/prolongement, date de début, forme d’exercice, catégorisations, indicateur/code APE, rattachement EIRL, date de mise à jour, fichier source.

### Tables SIRENE enrichies
- **Dirigeants** : `create_dirig_pp_table`/`create_dirig_pm_table` lisent les tables RNE, appliquent `preprocess_personne_physique` ou `preprocess_dirigeant_pm` (tri, déduplication, uppercase, agrégation de rôles via `map_roles`) puis reconstruisent les tables SIRENE avec index `siren`.
- **Immatriculation** : `copy_immatriculation_table` clone la structure RNE, insère un `SELECT DISTINCT` et crée un index sur `siren`.
- **Unité légale & siège** : `add_rne_siren_data_to_unite_legale_table` met à jour `from_rne` et `date_mise_a_jour_rne`, insère les SIREN absents ; `add_rne_data_to_siege_table` met à jour les dates puis insère les sièges manquants avec les attributs RNE (adresse, enseigne). Tables dérivées (ancien_siege, historiques, dates de fermeture) sont régénérées à partir des flux INSEE puis complétées par RNE.

### Règles de transformation / normalisation
- Validation systématique par `RNECompany` avant mapping.
- Choix de la meilleure valeur pour forme juridique, nature d’entreprise, adresses et rôles via helpers (`get_forme_juridique`, `get_nature_entreprise_list`, etc.).
- Upsert par SIREN via suppression préalable (`find_and_delete_same_siren`) puis insertion.
- Dédoublonnage global (`SELECT DISTINCT` + `VACUUM`) après injection.
- Sérialisation JSON de `nature_entreprise` issue des formes d’exercice et activités principales.
- Concaténation des prénoms dirigeants avec espace, nettoyage du SIREN dirigeant personne morale (suppression des espaces), agrégation des rôles via `map_roles` dans les préprocessings pandas.

## 8. Artefacts générés
- Fichiers stock JSON extraits du ZIP initial, puis supprimés après upload.
- Fichiers flux quotidiens `rne_flux_YYYY-MM-DD.json.gz` sur MinIO.
- Bases SQLite locales `rne_<start_date>.db` puis compressées (`rne_<date>.db.gz`) et archivées sur MinIO.
- Fichier de métadonnées `latest_rne_date.json` indiquant la date de reprise.
- Notifications Mattermost de statut.

## 9. Réutilisabilité du pipeline RNE
- Versionnement MinIO (bases compressées + métadonnées) permettant reprise incrémentale et historisation.
- Client API robuste (tokens multiples, retries, adaptation de `pageSize`, pagination `searchAfter`) réutilisable pour d’autres flux RNE similaires.
- Mécanismes de dédoublonnage générique (`SELECT DISTINCT`, `VACUUM`, upsert par SIREN) applicables à d’autres pipelines SQLite de masse.
- Prétraitements pandas par chunks et scripts Airflow avec nettoyage des répertoires temporaires, utilisables comme patrons pour d’autres intégrations.

## 10. Limites techniques / risques
Section à approfondir hors Codex.

## 11. Annexes éventuelles
Aucune annexe supplémentaire basée sur les sources.
