# Architecture du pipeline RNE

## 1. Contexte et objectifs du pipeline
- L'Annuaire des Entreprises orchestre plusieurs collectes quotidiennes, dont la base RNE fournie par l'INPI, afin d'alimenter son lac de données MinIO puis la chaîne ETL SQLite/Elasticsearch exposée à l'API de recherche.【F:README.md†L50-L99】
- Le pipeline RNE vise à industrialiser ce flux : récupérer le stock FTP et les flux différentiels quotidiens, consolider les données dans une base SQLite versionnée (`rne.db`) et alimenter les tables SIRENE (dirigeants, unités légales, sièges, immatriculations) qui servent ensuite à l'indexation publique.【F:config.py†L69-L92】【F:workflows/data_pipelines/etl/task_functions/create_dirig_tables.py†L36-L111】

## 2. Sources de données (inputs)
### Stock FTP INPI
- Le processeur `RneStockProcessor` orchestre un téléchargement via un script shell `get_stock.sh` qui encapsule `lftp` pour rapatrier `stock_rne.zip` vers un dossier temporaire avant envoi dans MinIO (`rne/stock`).【F:workflows/data_pipelines/rne/stock/processor.py†L14-L73】【F:workflows/data_pipelines/rne/stock/get_stock.sh†L3-L21】
- Chaque fichier extrait du zip est immédiatement téléversé dans le préfixe MinIO configuré par `RNE_STOCK_CONFIG`, garantissant une séparation stricte des artefacts sources et normalisés.【F:workflows/data_pipelines/rne/stock/processor.py†L44-L73】

### Flux API RNE (diff quotidien)
- Le DAG `get_flux_rne` tourne chaque nuit (1h) : il nettoie les dossiers temporaires, appelle `get_every_day_flux` puis purge et notifie Mattermost.【F:workflows/data_pipelines/rne/flux/DAG.py†L23-L57】
- `get_every_day_flux` détermine la fenêtre temporelle à traiter (start = dernier fichier disponible ou `RNE_DEFAULT_START_DATE`, end = J-1) puis rejoue `get_and_save_daily_flux_rne` jour par jour, en reprenant le `searchAfter` SIREN si besoin.【F:workflows/data_pipelines/rne/flux/flux_tasks.py†L104-L236】
- `ApiRNEClient` gère les tokens SSO, un `requests.Session` persistant et jusqu'à 100 tentatives avec adaptation dynamique de `pageSize` (100→5→1) lorsque l'API renvoie des erreurs mémoire/HTTP ; `searchAfter` est ajouté à l'URL pour paginer exhaustivement les siren triés.【F:workflows/data_pipelines/rne/flux/rne_api.py†L13-L120】
- Chaque page JSON est journalisée ligne à ligne dans `rne_flux_YYYY-MM-DD.json`, compressée en `.gz`, stockée sur MinIO (`rne/flux/`) et supprimée localement afin de limiter l'espace disque.【F:workflows/data_pipelines/rne/flux/flux_tasks.py†L117-L207】

### Métadonnées et garde-fous
- `latest_rne_date.json` enregistré sur MinIO fournit la date de reprise pour les DAGs flux et base ; il est téléchargé puis alimenté dans les XComs par `get_start_date_minio`. Si le fichier est absent (`NoSuchKey`), un traitement complet (stock + flux) est déclenché.【F:workflows/data_pipelines/rne/database/task_functions.py†L31-L63】
- Après traitement d'un lot, `upload_latest_date_rne_minio` incrémente la date et renvoie la nouvelle valeur dans MinIO afin de versionner l'état et d'éviter de retraiter des fichiers incomplets.【F:workflows/data_pipelines/rne/database/task_functions.py†L356-L385】

## 3. Cibles (outputs)
### Bases SQLite versionnées
- Le dépôt définit `RNE_DATABASE_LOCATION` dans l'arborescence Airflow (`.../extract_transform_load_db/data/rne.db`), tandis que les répertoires temporaires et MinIO (`rne/database/`) sont configurés globalement.【F:config.py†L69-L92】
- À chaque exécution du DAG `fill_rne_database`, un fichier `rne_<start_date>.db` est créé (ou récupéré) dans `/tmp/rne/database/`, rempli puis compressé en `rne_<start_date>.db.gz` avant d'être renommé sur MinIO selon la dernière date de flux traitée pour conserver un historique versionné.【F:workflows/data_pipelines/rne/database/task_functions.py†L108-L347】

### Tables métiers
- Le script `create_tables` matérialise les tables `unite_legale`, `siege`, `dirigeant_pp`, `dirigeant_pm`, `immatriculation`, `etablissement` et `activite` avec leurs colonnes fonctionnelles et index composites pour accélérer les requêtes ultérieures.【F:workflows/data_pipelines/rne/database/process_rne.py†L18-L194】
- Les enregistrements sont insérés table par table via `insert_unites_legales_into_db`, qui gère les sièges, activités et établissements rattachés à chaque SIREN, et inclut le nom du fichier source (`file_name`) pour tracer l'origine.【F:workflows/data_pipelines/rne/database/process_rne.py†L289-L430】

### MinIO & diffusion interne
- Les fichiers `.gz` (stock, flux, bases) sont centralisés sur MinIO aux préfixes `rne/stock`, `rne/flux` et `rne/database`, ce qui permet à d'autres DAGs (ETL, publication) de se baser sur les mêmes artefacts.【F:config.py†L81-L91】【F:workflows/data_pipelines/rne/database/task_functions.py†L338-L377】

### Intégration ETL SIRENE
- Le DAG ETL attache la base RNE à la base SIRENE pour copier ou fusionner les tables dirigeants, immatriculations, unités légales et sièges avec pandas pour le préprocessing (tri, déduplication, enrichissement de rôles).【F:workflows/data_pipelines/etl/task_functions/create_dirig_tables.py†L36-L111】【F:workflows/data_pipelines/etl/task_functions/create_immatriculation_table.py†L10-L46】【F:workflows/data_pipelines/etl/task_functions/create_unite_legale_tables.py†L73-L155】【F:workflows/data_pipelines/etl/task_functions/create_etablissements_tables.py†L165-L195】【F:workflows/data_pipelines/etl/data_fetch_clean/dirigeants.py†L1-L105】

## 4. Description détaillée du pipeline
### Étape 1 – Acquisition du stock initial
1. Airflow déclenche le téléchargement FTP via `get_stock.sh`, qui rapatrie un zip unique dans `/tmp/rne/stock` en utilisant des identifiants sécurisés dans les variables Airflow (`RNE_FTP_URL`).【F:workflows/data_pipelines/rne/stock/processor.py†L18-L42】【F:workflows/data_pipelines/rne/stock/get_stock.sh†L3-L21】
2. Les fichiers JSON décompressés sont envoyés un par un sur MinIO (`rne/stock/...`) avant suppression locale pour préserver l'espace disque.【F:workflows/data_pipelines/rne/stock/processor.py†L44-L73】

### Étape 2 – Ingestion du flux quotidien
1. `compute_start_date` lit la dernière date disponible dans MinIO ; si aucune, il repart de `RNE_DEFAULT_START_DATE` pour couvrir tout l'historique.【F:workflows/data_pipelines/rne/flux/flux_tasks.py†L104-L205】
2. `get_and_save_daily_flux_rne` crée un fichier par jour, rejoue l'API en suivant le `searchAfter` SIREN et sauvegarde en `.gz` sur MinIO après chaque journée ; en cas d'erreur, le fichier partiellement rempli est quand même archivé pour reprise et la tentative échoue explicitement.【F:workflows/data_pipelines/rne/flux/flux_tasks.py†L117-L207】
3. Les notifications Mattermost sont envoyées en fin de traitement ou en cas d'échec pour supervision 24/7.【F:workflows/data_pipelines/rne/flux/DAG.py†L23-L57】【F:workflows/data_pipelines/rne/flux/flux_tasks.py†L238-L254】

### Étape 3 – Normalisation & validation (Pydantic, mapping)
1. `inject_records_into_db` lit les fichiers stock/flux ; pour les flux, il tolère les erreurs JSON en les journalisant et en continuant le streaming afin d'éviter les blocages, puis insère par lots de 100 000 records pour maîtriser la mémoire.【F:workflows/data_pipelines/rne/database/process_rne.py†L196-L238】
2. Chaque ligne est validée par le modèle Pydantic `RNECompany` (héritant de `BaseModel`) qui impose la présence d'une formalité structurée et expose des helpers (`is_personne_morale`, etc.) pour la logique métier aval.【F:workflows/data_pipelines/rne/database/rne_model.py†L205-L219】
3. Le mapping `map_rne_company_to_ul` (non détaillé ici) assemble les données dans un objet `UniteLegale`, ensuite exploité par les fonctions d'insertion pour peupler les tables relationnelles.【F:workflows/data_pipelines/rne/database/process_rne.py†L289-L430】

### Étape 4 – Nettoyage (dédoublonnage global + upsert par SIREN)
1. `find_and_delete_same_siren` supprime les enregistrements plus anciens pour un SIREN donné avant toute insertion, garantissant un comportement « upsert » à grain SIREN/fichier.【F:workflows/data_pipelines/rne/database/process_rne.py†L256-L336】
2. Après l'injection, `remove_duplicates_from_tables` est rejoué sur les principales tables puis un `VACUUM` SQLite compresse la base.【F:workflows/data_pipelines/rne/database/task_functions.py†L250-L274】
3. Les pipelines pandas côté ETL effectuent des tris/dédoublonnages supplémentaires et agrègent les rôles dirigeants pour la diffusion finale.【F:workflows/data_pipelines/etl/data_fetch_clean/dirigeants.py†L11-L105】

### Étape 5 – Consolidation dans SQLite & indexation
1. `create_tables` prépare la structure puis `insert_unites_legales_into_db` peuple les tables, en ajoutant des index `idx_siren_*` pour accélérer les jointures entre unités, sièges et activités.【F:workflows/data_pipelines/rne/database/process_rne.py†L18-L194】【F:workflows/data_pipelines/rne/database/process_rne.py†L289-L430】
2. Le DAG `fill_rne_database` orchestre le séquencement : récupère la dernière base, traite le stock si nécessaire, applique les flux, nettoie, contrôle, publie puis notifie.【F:workflows/data_pipelines/rne/database/DAG.py†L23-L93】
3. Le DAG ETL rattache ensuite la base `rne.db` au SIRENE master, copie les tables et reconstruit les champs enrichis (dirigeants, unités légales, sièges), étape préalable à la reconstruction complète des index Elasticsearch de l'annuaire.【F:workflows/data_pipelines/etl/task_functions/create_dirig_tables.py†L36-L111】【F:workflows/data_pipelines/etl/task_functions/create_unite_legale_tables.py†L73-L155】【F:workflows/data_pipelines/etl/task_functions/create_etablissements_tables.py†L165-L231】

### Étape 6 – Contrôles volumétriques et garde-fous
1. `check_db_count` impose des seuils (20 M d'UL, 11 M de dirigeants PP, 1 M de dirigeants PM, etc.). Toute anomalie déclenche une exception Airflow, stoppant la publication d'une version suspecte.【F:workflows/data_pipelines/rne/database/task_functions.py†L276-L314】
2. Le dernier fichier de flux est volontairement ignoré lors du traitement pour éviter d'intégrer une journée potentiellement encore en cours de génération.【F:workflows/data_pipelines/rne/database/task_functions.py†L202-L247】
3. `latest_rne_date.json` et les notifications Mattermost permettent un suivi opérationnel (date couverte, succès/échec).【F:workflows/data_pipelines/rne/database/task_functions.py†L31-L63】【F:workflows/data_pipelines/rne/database/task_functions.py†L356-L393】

### Étape 7 – Versionnement de la base & diffusion
1. `upload_db_to_minio` compresse la base en `.gz`, l'envoie vers `rne/database/` et supprime l'artefact local pour éviter les dérives de stockage ; le nommage `rne_<last_date>.db.gz` sert d'ID de version temporelle.【F:workflows/data_pipelines/rne/database/task_functions.py†L322-L353】
2. `get_latest_db` est capable de restaurer la version précédente (J-1) pour appliquer des flux incrémentaux, ce qui permet des reprises rapides après incident.【F:workflows/data_pipelines/rne/database/task_functions.py†L108-L146】

### Étape 8 – Intégration dans SIRENE
1. Les tâches `create_dirig_pp_table` et `create_dirig_pm_table` lisent `rne.db`, extraient les données par chunks de 100k SIREN, nettoient via pandas puis insèrent dans `sirene.db` avec index sur `siren`.【F:workflows/data_pipelines/etl/task_functions/create_dirig_tables.py†L53-L111】
2. `add_rne_siren_data_to_unite_legale_table` et `add_rne_data_to_siege_table` exécutent des `UPDATE` suivis d'`INSERT OR IGNORE` pour fusionner les champs RNE avec les stocks INSEE, garantissant que l'annuaire possède les données les plus fraîches côté identité et siège.【F:workflows/data_pipelines/etl/task_functions/create_unite_legale_tables.py†L127-L155】【F:workflows/data_pipelines/etl/task_functions/create_etablissements_tables.py†L165-L195】
3. `copy_immatriculation_table` copie intégralement la table RNE vers SIRENE pour assurer la cohérence des informations capitalistiques, puis crée un index `idx_siren_immat`.【F:workflows/data_pipelines/etl/task_functions/create_immatriculation_table.py†L10-L40】
4. `get_rne_database` fournit aussi la date « last modified » utilisée pour l'API de métadonnées `data_source_updates.json` exposée publiquement.【F:workflows/data_pipelines/etl/task_functions/create_dirig_tables.py†L36-L51】【F:workflows/data_pipelines/etl/task_functions/create_json_last_modified.py†L112-L134】

## 5. Technologies & composants
- **Airflow** : Deux DAGs (`get_flux_rne`, `fill_rne_database`) orchestrent flux et consolidation avec stratégies de retries, notifications et nettoyage des répertoires temporaires.【F:workflows/data_pipelines/rne/flux/DAG.py†L23-L57】【F:workflows/data_pipelines/rne/database/DAG.py†L30-L93】
- **MinIO** : Client maison `MinIOClient` utilisé dans tous les modules pour lister, télécharger ou téléverser stock, flux, bases et métadonnées ; les préfixes sont centralisés dans `config.py`.【F:config.py†L81-L91】【F:workflows/data_pipelines/rne/database/task_functions.py†L31-L377】
- **SQLite** : Base locale `rne.db` attachée ensuite à `sirene.db`; création de tables/index et opérations `VACUUM` pour maintenir des performances acceptables sur plusieurs dizaines de millions de lignes.【F:workflows/data_pipelines/rne/database/process_rne.py†L18-L194】【F:workflows/data_pipelines/rne/database/task_functions.py†L250-L274】
- **Pydantic** : Les modèles `RNECompany` sécurisent la validation du JSON brut avant mapping, facilitant l'évolution du schéma sans effets de bord.【F:workflows/data_pipelines/rne/database/rne_model.py†L205-L219】
- **pandas** : Utilisé dans les fonctions `preprocess_*` pour nettoyer, ordonner et dédupliquer les tables dirigeants avant insertion dans SIRENE (groupby, uppercase, mapping de rôles).【F:workflows/data_pipelines/etl/data_fetch_clean/dirigeants.py†L1-L105】
- **Scripts shell** : `get_stock.sh` encapsule l'accès FTP et standardise le nommage du zip stock, appelé depuis Airflow pour simplifier la rotation des secrets.【F:workflows/data_pipelines/rne/stock/get_stock.sh†L3-L21】
- **Gestion des tokens API** : `ApiRNEClient` obtient des tokens SSO, réessaie automatiquement les appels et adapte dynamiquement la pagination `pageSize` pour contourner les limites du service RNE.【F:workflows/data_pipelines/rne/flux/rne_api.py†L13-L120】

## 6. Dataflow complet (ASCII)
```
[FTP INPI Stock]
        |
        v
+-----------------+       +------------------+       +-----------------+
| MinIO rne/stock |<--+-->|  MinIO rne/flux  |-->+-->| SQLite rne.db   |
+-----------------+   |   +------------------+   |   +-----------------+
                      |                          |        |
                      |                          |        v
                      |                          |  Tables nettoyées
                      |                          | (UL, siège, dir.)
                      |                          |        |
                      |                          v        v
                      +------------------> ETL SIRENE (sirene.db)
                                             |
                                             v
                                       Indexation & API
```

## 7. Règles de gestion importantes
- **Stratégie par SIREN** : avant toute insertion, les lignes portant le même SIREN (et un `file_name` différent) sont supprimées, assurant l'écrasement complet par unité légale/siege/dirigeant au sein de la base RNE.【F:workflows/data_pipelines/rne/database/process_rne.py†L256-L336】
- **Dédoublonnage global** : une passe générique sur les tables majeures supprime les doublons restants et déclenche un `VACUUM` pour préserver la compacité de la base.【F:workflows/data_pipelines/rne/database/task_functions.py†L250-L274】
- **Ordre d’ingestion** : stock intégral (si pas de start_date) → flux incrémentaux (sauf le dernier fichier en cours) → suppression des doublons → contrôles volumétriques → publication MinIO → mise à jour métadonnées.【F:workflows/data_pipelines/rne/database/task_functions.py†L149-L248】
- **UPDATE/INSERT dans SIRENE** : les requêtes `UPDATE ... SET` synchronisent les champs déjà présents, puis `INSERT OR IGNORE` ajoute les SIREN/sieges manquants pour éviter les doublons côté SIRENE.【F:workflows/data_pipelines/etl/task_functions/create_unite_legale_tables.py†L127-L155】【F:workflows/data_pipelines/etl/task_functions/create_etablissements_tables.py†L165-L195】
- **Vérifications de cohérence** : seuils minimaux, notifications et versionnement empêchent l'exposition d'une base partielle ; l'API flux ignore automatiquement les journées incomplètes et se relance avec les tokens corrects en cas d'erreur HTTP spécifique.【F:workflows/data_pipelines/rne/database/task_functions.py†L202-L314】【F:workflows/data_pipelines/rne/flux/rne_api.py†L85-L120】

## 8. Limites et points sensibles
- **Taille des fichiers** : le stock et les flux journaliers peuvent atteindre plusieurs Go ; l'écriture ligne à ligne puis la compression `.gz` limitent la mémoire mais imposent des E/S importantes, d'où la suppression immédiate des fichiers locaux.【F:workflows/data_pipelines/rne/flux/flux_tasks.py†L117-L207】
- **SQLite** : malgré l'indexation, les opérations `VACUUM` sont nécessaires pour maintenir les performances sur des tables de dizaines de millions de lignes ; la consolidation reste monothreadée et peut durer plusieurs heures.【F:workflows/data_pipelines/rne/database/process_rne.py†L18-L194】【F:workflows/data_pipelines/rne/database/task_functions.py†L250-L274】
- **Flux incomplets** : le dernier fichier quotidien n'est pas injecté par sécurité ; en cas d'interruption prolongée, la reprise peut devoir rejouer plusieurs jours, augmentant le temps de traitement.【F:workflows/data_pipelines/rne/database/task_functions.py†L202-L247】
- **API RNE** : limites de rate limiting, erreurs 500 « Allowed memory size » et expirations de token sont gérées par `ApiRNEClient` (retries, baisse de pageSize), mais peuvent rallonger considérablement l'exécution et nécessitent une supervision active.【F:workflows/data_pipelines/rne/flux/rne_api.py†L85-L118】

## Résumé
Le pipeline RNE combine une collecte FTP + API, une normalisation Pydantic, un nettoyage/contrôle rigoureux et une intégration forte avec la base SIRENE et l'écosystème Annuaire. Son architecture repose sur Airflow, MinIO et SQLite pour assurer la traçabilité et le versionnement des données INPI avant leur exposition publique.【F:README.md†L50-L99】【F:config.py†L69-L116】【F:workflows/data_pipelines/rne/database/DAG.py†L23-L93】
