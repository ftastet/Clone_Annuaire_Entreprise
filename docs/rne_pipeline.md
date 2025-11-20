# Vue d'ensemble du pipeline RNE

## Objectif global du pipeline
Le pipeline RNE (Registre National des Entreprises) orchestre l'acquisition du stock historique, des flux quotidiens et leur consolidation dans une base SQLite versionnée publiée sur MinIO. Cette base sert ensuite à enrichir l'annuaire SIRENE (unité légale, établissements, dirigeants et immatriculations) dans l'ETL principal, garantissant des données RNE fraîches et normalisées pour l'annuaire numérique.

## Lien entre RNE et l'annuaire SIRENE
La base RNE construite quotidiennement est transférée vers l'ETL SIRENE : les tables RNE locales sont attachées au SQLite SIRENE, puis copiées, nettoyées et fusionnées avec les tables SIRENE (dirigeants, immatriculations, unités légales, sièges, historiques). Les tâches `create_dirig_tables`, `copy_immatriculation_table`, `create_unite_legale_tables` et `create_etablissements_tables` encapsulent cette intégration avec des prétraitements pandas, des requêtes SQL d'`INSERT OR IGNORE` et des mises à jour directes sur les champs alimentés par RNE.

# Sources de données (inputs)
| Source | Composants | Format | Emplacement |
| --- | --- | --- | --- |
| Stock historique INPI | `RneStockProcessor`, script `get_stock.sh` via `lftp`, enchaîné avec l'envoi MinIO | Archive ZIP contenant des JSON | Téléchargée depuis `RNE_FTP_URL`, stockée localement dans `/tmp/rne/stock`, puis vers MinIO `rne/stock/` |
| Flux quotidien INPI | DAG `get_flux_rne`, tâche `get_every_day_flux`, fonction `get_and_save_daily_flux_rne`, client `ApiRNEClient` | JSON ND compressé en `.json.gz` par jour | MinIO `rne/flux/` (`ae/<ENV>/rne/flux/`), staging local `RNE_FLUX_DATADIR` |
| Fichiers de métadonnées | `latest_rne_date.json`, bases `rne_<date>.db.gz` | JSON + SQLite compressé | MinIO `rne/database/` |

# Cibles (outputs)
- **Bases SQLite RNE versionnées** : chaque run publie `rne_<date>.db.gz` sur MinIO après recompression et renomme la version selon la dernière date de flux traitée, assurant un archivage journalier.
- **Tables normalisées RNE** : `unite_legale`, `siege`, `dirigeant_pp`, `dirigeant_pm`, `immatriculation`, `etablissement`, `activite` créées dans SQLite avec index pour les jointures rapides.
- **Tables enrichies dans SIRENE** : import en blocs de `dirigeant_pp/pm`, copie distincte d'`immatriculation`, mise à jour de `unite_legale` et `siege`, ainsi que des tables dérivées (historique, dates de fermeture).
- **Notifications Mattermost** : un message vert récapitule les plages traitées (stock/flux, base) et un message rouge signale les échecs du DAG de flux.

# Grandes étapes du pipeline RNE
## 4.1 Acquisition des données
- **Stock initial via FTP** : `RneStockProcessor.download_stock` exécute `get_stock.sh` (lftp) pour rapatrier `stock_rne.zip`, extrait chaque JSON puis l'expédie sur MinIO `rne/stock/`. Le dossier temporaire est créé si besoin et les fichiers extraits sont supprimés après upload.
- **Flux quotidien via API diff** : le DAG `get_flux_rne` s'exécute à 1h, purge le répertoire temporaire, puis `get_every_day_flux` boucle de la dernière date stockée jusqu'à J-1. `get_and_save_daily_flux_rne` gère la pagination via `searchAfter` (`last_siren`), stocke les réponses en JSON ND, compresse en `.gz` et pousse sur MinIO. Les erreurs déclenchent une sauvegarde partielle (pour reprise) puis une exception qui supprime les fichiers incomplets pour forcer un nouvel essai. Une notification de succès ou d'échec est envoyée à Mattermost.
- **Client API et pagination** : `ApiRNEClient` gère des jetons multiples, un `requests.Session` résilient et ajuste dynamiquement la `pageSize` (100 → 5 → 1) selon les erreurs HTTP 500, tout en ajoutant `searchAfter` au paramètre d'URL. Les codes 401/403/429 provoquent un rafraîchissement du token, les erreurs critiques déclenchent jusqu'à 100 tentatives avec pauses.

## 4.2 Constitution et versioning de la base SQLite RNE
- **Lecture des métadonnées** : `get_start_date_minio` télécharge `latest_rne_date.json` de MinIO, en extrait la `latest_date` et la réémet comme start_date. En absence de fichier (`NoSuchKey`), la base est reconstruite depuis le stock complet.
- **Création/Reprise de la base** : `create_db` instancie `rne_<start_date>.db` et applique `create_tables`. Si un start_date existe, `get_latest_db` rapatrie la base de la veille depuis MinIO (`rne_<previous>.db.gz`) pour la poursuivre.
- **Chargement ordonné** : `process_stock_json_files` n'injecte le stock que lors d'une reconstruction complète, tandis que `process_flux_json_files` récupère tous les flux disponibles, ignore le dernier fichier (potentiellement incomplet), décompresse chaque `.json.gz`, charge les JSON en respectant l'ordre chronologique et supprime les fichiers locaux. La dernière date traitée est stockée pour le versioning.
- **Versioning** : `upload_db_to_minio` recompresse `rne_<start>.db` en `.gz`, l'upload sous `rne_<last_date_processed>.db.gz` et efface les fichiers locaux. `upload_latest_date_rne_minio` écrit une nouvelle `latest_rne_date.json` (date traitée +1) pour la prochaine exécution.

## 4.3 Transformation, mapping et qualité
- **Validation Pydantic** : chaque enregistrement JSON est validé via `RNECompany` (Pydantic) pour homogénéiser les sous-blocs (personne physique, morale, exploitation). Des helpers (ex. `is_personne_physique`) exposent les branchements métier.
- **Mapping interne** : `map_rne_company_to_ul` convertit un `RNECompany` vers un modèle `UniteLegale` interne, en récupérant l'identité, la cessation, les établissements, dirigeants, adresses et activités. Des utilitaires choisissent la meilleure source (forme juridique, natures d'entreprise, etc.).
- **Insertion structurée** : `insert_unites_legales_into_db` écrit les enregistrements dans les tables normalisées (`unite_legale`, `siege`, `dirigeant_pp`, `dirigeant_pm`, `immatriculation`, `etablissement`, `activite`). Avant chaque insertion, `find_and_delete_same_siren` supprime toutes les lignes plus anciennes pour le même SIREN afin d'implémenter un upsert à grain RNE.
- **Déduplication globale** : après l'injection, `remove_duplicates` rejoue `SELECT DISTINCT` table par table dans des tables temporaires, reconstruit les index puis exécute `VACUUM` pour conserver une base compacte. `check_db_count` impose des planchers volumétriques (≥20 M UL/sièges/immatriculations, 11 M dirigeants PP, 1 M dirigeants PM) et lève une exception si un seuil est franchi, forçant l'échec du DAG pour protéger la qualité.

## 4.4 Intégration dans l'ETL SIRENE
- **Téléchargement du dernier dump** : `get_rne_database` (DAG ETL) récupère le `rne.db.gz` le plus récent sur MinIO, le décompresse puis l'attache à la base SIRENE locale via `sqlite3 ATTACH`. Chaque table est ensuite traitée par blocs pour limiter la RAM (chunks de 100 000 SIREN).
- **Dirigeants** : `create_dirig_pp_table` et `create_dirig_pm_table` lisent la base RNE par blocs, appliquent `preprocess_personne_physique` ou `preprocess_dirigeant_pm` (tri, dédoublonnage, homogénéisation de la casse, regroupement des rôles via `map_roles`) puis réécrivent les tables SIRENE. Les index `siren` sont recréés après chargement.
- **Immatriculations** : `copy_immatriculation_table` crée la table cible en clonant la structure RNE et insère un `SELECT DISTINCT` pour éviter les doublons, suivi d'un index sur `siren`.
- **Unités légales & sièges** : `add_rne_siren_data_to_unite_legale_table` met à jour les lignes existantes (champ `from_rne`, `date_mise_a_jour_rne`) et insère les SIREN absents via `INSERT OR IGNORE`. En parallèle, `add_rne_data_to_siege_table` met à jour `date_mise_a_jour_rne` puis insère les sièges manquants avec les attributs RNE (adresse, enseigne).
- **Historisation et dérivés** : les tables `ancien_siege`, `historique_unite_legale`, `historique_etablissement`, `date_fermeture_*` sont regénérées à partir des flux INSEE puis complétées par les données RNE pour refléter les changements de sièges et fermetures.

## 4.5 Supervision et notifications
- Les DAGs `get_flux_rne` (1h) et `fill_rne_database` (2h) incluent un nettoyage systématique des répertoires temporaires, une limite `max_active_runs=1` et un `dagrun_timeout` généreux. En cas d'échec (flux incomplet, seuil volumétrique KO, erreur API), les tâches lèvent une exception pour faire échouer le DAG et déclencher la notification rouge. Les réussites envoient un message vert avec les bornes temporelles traitées.

# Technologies et composants principaux
- **Airflow** : deux DAGs dédiés (flux, base) avec dépendances explicites, scheduling quotidien, gestion des répertoires temporaires via `BashOperator` et callbacks de notification.
- **MinIO** : stockage objet pour les stocks (`rne/stock/`), flux (`rne/flux/`), bases versionnées (`rne/database/`) et métadonnées (`latest_rne_date.json`). Les fonctions `get_files`, `send_files`, `get_latest_file` orchestrent téléchargements et uploads sécurisés.
- **SQLite** : base locale pour RNE (tables + index) et pour SIRENE. Les scripts utilisent `ATTACH DATABASE`, des requêtes `INSERT OR IGNORE`, `VACUUM` et des index dédiés pour garantir les performances.
- **Pydantic** : validation forte du schéma RNE (`RNECompany` et modèles imbriqués) avant mapping vers les modèles internes `UniteLegale`, `Siege`, etc.
- **pandas** : utilisé dans les prétraitements dirigeants/établissements pour trier, dédoublonner, homogénéiser la casse et agréger les rôles, réduisant les doublons avant insertion.
- **Scripts shell** : `get_stock.sh` encapsule lftp pour sécuriser le téléchargement du stock en une ligne robuste.
- **Mattermost** : `send_message` diffuse des notifications success/failure pour les flux et la base RNE, fournissant la traçabilité aux opérateurs.

# Schéma fonctionnel du flux de données
```mermaid
flowchart TD
    A[FTP INPI (stock ZIP)] -->|get_stock.sh + RneStockProcessor| B[MinIO rne/stock/];
    A2[RNE API companies/diff (flux J)] -->|ApiRNEClient + get_every_day_flux| C[MinIO rne/flux/ (.json.gz)];
    B --> D[Processus fill_rne_database];
    C --> D;
    D -->|Validation Pydantic + mapping + dédup| E[Base SQLite RNE versionnée];
    E -->|Compression + versioning| F[MinIO rne/database/ + latest_rne_date.json];
    F -->|get_rne_database| G[ETL SIRENE SQLite];
    G -->|Prétraitements pandas + SQL| H[Tables SIRENE enrichies (UL, sièges, dirigeants, immat)];
    H --> I[Annuaire enrichi + notifications Mattermost];
```

# Détail des tables de sortie RNE
- **unite_legale** (16 colonnes) : identifiants SIREN, civilité (nom/prénom/nom usage), dénominations, dates de création/mise à jour, activité principale, tranche d'effectif, nature juridique, état administratif, forme d'exercice, statut de diffusion, adresse formatée, provenance (`file_name`). Sert de socle pour l'unité légale dans SIRENE.
- **siege** (17 colonnes) : SIREN/SIRET siège, enseigne, nom commercial, adresse détaillée (pays, codes, voie, complément), date de mise à jour et fichier source. Alimente la table `siege` SIRENE pour les informations de localisation et d'enseigne du siège social.
- **dirigeant_pp** (11 colonnes) : SIREN, date de mise à jour, identité (nom, nom usage, prénoms, genre, date de naissance), rôle, nationalité, situation matrimoniale, fichier source. Permet de publier les dirigeants personnes physiques liés à chaque SIREN.
- **dirigeant_pm** (8 colonnes) : SIREN entreprise, SIREN du dirigeant personne morale, date de mise à jour, dénomination, rôle, pays, forme juridique, fichier source. Fournit les dirigeants personnes morales et leurs rôles consolidés.
- **immatriculation** (13 colonnes) : SIREN, dates (mise à jour, immatriculation, radiation, début d'activité, clôture d'exercice), indicateur associé unique, capital social/variable, devise, durée de la personne morale, nature d'entreprise sérialisée, fichier source. Alimente la section juridique (capital, nature) de l'annuaire.
- **etablissement** (4 colonnes) : SIREN, SIRET, date de mise à jour, fichier source, utilisé pour reconstituer l'inventaire des établissements RNE et y associer les activités.
- **activite** (15 colonnes) : SIREN/SIRET, code de catégorie, indicateurs principal/prolongement, dates de début, formes d'exercice, catégorisations, indicateur/code APE, rattachement EIRL, date de mise à jour, fichier source. Ces activités sont liées au siège et aux autres établissements pour enrichir les codes métiers dans SIRENE.

# Règles de gestion importantes
1. **Upsert par SIREN** : avant chaque insertion, toutes les lignes RNE plus anciennes pour un même SIREN sont supprimées pour éviter les doublons et garantir que la version la plus récente remplace l'ancienne.
2. **Déduplication globale** : après chargement, chaque table est recalculée avec `SELECT DISTINCT`, les index sont recréés puis `VACUUM` est lancé, assurant l'unicité globale et la compaction des fichiers.
3. **Contrôles volumétriques** : la fonction `check_db_count` applique des seuils (20 M UL/immatriculations, 11 M dirigeants PP, 1 M dirigeants PM) et échoue volontairement le DAG si un seuil est sous-performant (ex. flux incomplet).
4. **Gestion des flux partiels** : `process_flux_json_files` ignore systématiquement le dernier fichier (susceptible d'être en cours d'écriture) et `get_and_save_daily_flux_rne` supprime les fichiers locaux corrompus, garantissant que seuls les jours complets sont injectés.
5. **Choix de la meilleure valeur** : les helpers de mapping (forme juridique, nature d'entreprise, adresse, dirigeants) parcourent les différents blocs d'un `RNECompany` pour sélectionner la valeur la plus fiable (ex. `get_forme_juridique`, `get_nature_entreprise_list`).
6. **Traitements par blocs** : les copies vers SIRENE (dirigeants) utilisent des chunks de 100 000 lignes pour limiter la consommation mémoire et assurer des `commit` réguliers.
7. **Gestion des tokens API** : en cas de codes 401/403/429, un nouveau token est généré et la requête est rejouée ; les erreurs 500 réduisent la taille de page pour contourner les limitations de la plateforme INPI.

# Résumé exécutif
Ce pipeline RNE assure l'ingestion quotidienne du stock et des flux INPI, leur validation Pydantic et leur normalisation dans une base SQLite versionnée, stockée dans MinIO. Il garantit une fraîcheur quotidienne (J-1) grâce à la capture du flux `companies/diff`, au versioning `latest_rne_date.json` et aux contrôles volumétriques qui bloquent toute publication partielle.

La base RNE sert ensuite à enrichir l'ETL Annuaire : les tables SIRENE sont rafraîchies en blocs, nettoyées via pandas et synchronisées avec les données RNE (dirigeants, immatriculations, unités légales, sièges). Les requêtes SQL dédiées appliquent des `INSERT OR IGNORE` et des `UPDATE` ciblés pour combiner les meilleures informations INSEE/RNE tout en respectant les contraintes d'unicité.

Enfin, la supervision (DAGs séquencés, notifications Mattermost, suppression des fichiers partiels, `VACUUM`) offre une visibilité claire aux opérateurs et assure que seules des bases cohérentes et complètes alimentent l'annuaire SIRENE, consolidant la qualité des données exposées aux utilisateurs finaux.
