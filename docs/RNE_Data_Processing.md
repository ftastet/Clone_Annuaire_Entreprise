# Traitement des données RNE pour l'Annuaire des Entreprises

## 1. Objectif et finalité
- Garantir une alimentation quotidienne (J-1) du Registre National des Entreprises (RNE) dans l'annuaire SIRENE afin d'exposer des informations fraîches sur les unités légales, établissements, dirigeants et immatriculations.
- Industrialiser l'ensemble du flux : collecte du stock FTP et des flux différentiels, validation, normalisation, nettoyage, versioning et diffusion vers l'ETL principal puis l'API publique.

## 2. Architecture et composants clés
- **Airflow** : deux DAGs dédiés orchestrent l'acquisition des flux (`get_flux_rne`) et la consolidation/base (`fill_rne_database`), avec nettoyage des répertoires temporaires, dépendances explicites, retries et notifications Mattermost.
- **MinIO** : stockage objet unique pour tous les artefacts RNE (stock `rne/stock/`, flux `rne/flux/`, bases `rne/database/`, métadonnées `latest_rne_date.json`). Client applicatif (`MinIOClient`) centralisé pour lister, télécharger et téléverser.
- **SQLite** : base locale `rne.db` versionnée puis attachée à `sirene.db` pour l'intégration ; usage d'`ATTACH`, `INSERT OR IGNORE`, index dédiés et `VACUUM` pour maintenir les performances sur des dizaines de millions de lignes.
- **Pydantic** : validation stricte du JSON brut via `RNECompany` et modèles imbriqués avant mapping vers les modèles internes (`UniteLegale`, `Siege`, etc.).
- **pandas** : prétraitements (tri, uppercase, déduplication, agrégation des rôles dirigeants) lors de la régénération des tables SIRENE.
- **Scripts shell** : `get_stock.sh` encapsule `lftp` pour rapatrier le stock FTP, standardiser le nommage et gérer les identifiants sécurisés.
- **Gestion API** : `ApiRNEClient` gère les tokens SSO, un `requests.Session` résilient, la pagination `searchAfter` et l'adaptation dynamique de `pageSize` (100→5→1) en cas d'erreurs 500/429/401/403.
- **Supervision** : notifications Mattermost vert/rouge, seuils volumétriques bloquants et versioning MinIO pour éviter toute diffusion partielle.

## 3. Sources de données et récupération
### 3.1 Stock FTP initial
- Téléchargé via `RneStockProcessor` qui exécute `get_stock.sh` (`lftp`) depuis `RNE_FTP_URL` vers `/tmp/rne/stock`.
- Archive `stock_rne.zip` extraite localement ; chaque JSON est immédiatement envoyé sur MinIO `rne/stock/` avant suppression locale pour limiter l'espace disque.

### 3.2 Flux différentiel quotidien (API INPI)
- DAG `get_flux_rne` planifié à 1h : purge les dossiers temporaires puis appelle `get_every_day_flux` sur la fenêtre [dernière date connue ; J-1].
- `get_and_save_daily_flux_rne` rejoue l'API jour par jour : pagination exhaustive via `searchAfter` (SIREN triés), stockage ligne à ligne en JSON ND, compression en `.json.gz`, upload sur MinIO `rne/flux/`, suppression locale.
- Gestion d'erreurs : baisse progressive de `pageSize`, rafraîchissement de token, 100 tentatives maximum ; en cas d'échec, le fichier partiellement rempli est archivé mais l'exception force la reprise ultérieure.

### 3.3 Métadonnées et garde-fous de reprise
- `latest_rne_date.json` sur MinIO fixe la date de reprise pour les DAGs flux et base ; s'il est absent (`NoSuchKey`), le pipeline reconstruit intégralement (stock + flux).
- Après chaque lot traité, `upload_latest_date_rne_minio` écrit la nouvelle date (dernier jour complet +1) afin d'éviter toute ré-ingestion inutile.

## 4. Chaînage fonctionnel de bout en bout
### 4.1 Séquencement général
1. **Initialisation** : lecture de la date de départ (`get_start_date_minio`). Si aucune, déclenchement du traitement complet avec stock.
2. **Acquisition** : téléchargement du stock si nécessaire, puis collecte de tous les flux disponibles en ignorant systématiquement le dernier fichier (potentiellement incomplet).
3. **Construction de la base RNE** : création ou reprise d'une base `rne_<start_date>.db`, validation/mapping/insertion, déduplication et contrôles volumétriques.
4. **Versioning et publication** : compression `rne_<start_date>.db.gz`, renommage en `rne_<last_date_processed>.db.gz` sur MinIO, mise à jour de `latest_rne_date.json`.
5. **Intégration SIRENE** : téléchargement de la dernière base RNE, rattachement à `sirene.db`, prétraitements pandas, mises à jour et insertions SQL, reconstruction des tables dérivées.
6. **Supervision** : notifications Mattermost et nettoyage des répertoires temporaires.

### 4.2 Traitement détaillé des données RNE
- **Création de la base** : `create_db` instancie la base et appelle `create_tables` (tables normalisées + index). Si une base précédente existe pour la date de reprise, `get_latest_db` la rapatrie depuis MinIO pour appliquer les nouveaux flux.
- **Injection contrôlée** :
  - Stock injecté uniquement lors d'une reconstruction complète (`process_stock_json_files`).
  - Flux injectés par ordre chronologique (`process_flux_json_files`), après décompression. Les erreurs JSON sont journalisées mais n'interrompent pas l'injection (tolérance de parsing).
  - `inject_records_into_db` lit les fichiers par flux/stock, valide chaque ligne via Pydantic et alimente les tables par lots de 100 000 lignes pour maîtriser la mémoire.
- **Mapping et valorisation** : `map_rne_company_to_ul` transforme chaque `RNECompany` en objets internes (UL, siège, dirigeants, établissements, activités, immatriculation) en choisissant les meilleures valeurs disponibles (formes juridiques, dates, nature d'entreprise, adresses). Les helpers dédiés (`get_forme_juridique`, `get_date_creation`, `get_detail_cessation`, `get_nature_entreprise_list`, etc.) combinent plusieurs sources au sein du JSON.
- **Upsert par SIREN** : avant toute insertion, `find_and_delete_same_siren` supprime les lignes plus anciennes pour le SIREN dans toutes les tables concernées, garantissant que la version la plus récente remplace l'ancienne.
- **Dédoublonnage global** : `remove_duplicates_from_tables` rejoue des `SELECT DISTINCT` table par table, reconstruit les index puis exécute `VACUUM` pour compacter la base.
- **Contrôles volumétriques** : `check_db_count` impose des minima (≥20 M UL/immatriculations/sièges, 11 M dirigeants PP, 1 M dirigeants PM). Toute anomalie fait échouer le DAG pour éviter la publication d'une base suspecte.

### 4.3 Versionnement et diffusion de la base
- `upload_db_to_minio` recompresse la base et la publie sur MinIO sous le nom basé sur la dernière date de flux traitée, assurant un archivage quotidien.
- Les bases sont éphémères localement (suppression après upload) et peuvent être restaurées rapidement (`get_latest_db`) pour appliquer des flux incrémentaux ou rejouer après incident.

### 4.4 Intégration dans l'ETL SIRENE
- **Récupération** : `get_rne_database` télécharge et décompresse la dernière base RNE depuis MinIO, puis l'attache à `sirene.db` via `sqlite3 ATTACH`.
- **Dirigeants** : `create_dirig_pp_table` et `create_dirig_pm_table` lisent `db_rne.dirigeant_*` par blocs de 100k SIREN, appliquent `preprocess_personne_physique`/`preprocess_dirigeant_pm` (uppercase, déduplication, agrégation des rôles via `map_roles`), recréent les tables et leurs index.
- **Immatriculation** : `copy_immatriculation_table` clone la structure RNE et insère un `SELECT DISTINCT` complet, puis crée un index `siren`.
- **Unités légales et sièges** : `add_rne_siren_data_to_unite_legale_table` met à jour les lignes existantes (`from_rne`, `date_mise_a_jour_rne`) et insère les SIREN absents via `INSERT OR IGNORE`. `add_rne_data_to_siege_table` met à jour `date_mise_a_jour_rne` puis insère les sièges manquants avec les attributs RNE (enseignes, adresses).
- **Historisation et dérivés** : les tables `ancien_siege`, `historique_unite_legale`, `historique_etablissement`, `date_fermeture_*` sont regénérées après ces mises à jour pour refléter les changements de sièges et fermetures.
- **Traçabilité** : la date « last modified » de la base RNE est propagée vers l'API `data_source_updates.json`.

## 5. Modèle de données et mapping
### 5.1 Principaux blocs JSON en entrée
- **Métadonnées de formalité** : `siren`, `updatedAt`, `createdAt`, `origin`, `formality.*` (type de personne, forme juridique, diffusion INSEE, code APE, date de création, indicateurs de création/cessation).
- **Identité d'entreprise ou entrepreneur** : dénomination, forme juridique (texte + codifiée), nom commercial, tranche d'effectif, dates (immatriculation, début d'activité), capital social/variable, devise, durée, date de clôture, indicateur associé unique.
- **Description personne physique** : noms/prénoms/genre/date de naissance/nationalité/situation matrimoniale, adresses personnelles (non répliquées).
- **Composition et dirigeants** : `composition.pouvoirs[]` pour dirigeants personnes physiques (`INDIVIDU`) ou morales (`ENTREPRISE`) avec rôles, coordonnées et formes juridiques ; possibilité de récupérer l'entrepreneur individuel via `identite.entrepreneur` si `composition` est absent.
- **Adresses et établissements** : `adresseEntreprise.adresse.*` (siège), `etablissementPrincipal` (SIRET, enseigne, nom commercial, adresse, activités) et `autresEtablissements[]` (SIRET, adresses, activités).
- **Activités** : catégorie, indicateurs principal/prolongement/APE, date de début, forme d'exercice, catégorisations fines, rattachement EIRL.
- **Cessation et état administratif** : dates de cessation/radiation (`detailCessationEntreprise`), `natureCessationEntreprise.etatAdministratifInsee`.

### 5.2 Tables normalisées RNE (`rne.db`)
- **unite_legale** : SIREN, identités (dénomination/nom/prénom/nom_usage), nom commercial, dates de création/mise à jour, tranche d'effectif, nature juridique (choix priorisé entre formalité, nature de création, identité), état administratif, forme d'exercice, statut de diffusion, adresse formatée à partir d'`adresseEntreprise`, provenance `file_name`.
- **siege** : SIREN/SIRET, enseigne, nom commercial, adresse détaillée (pays, codes INSEE/postal, voie, numéro, type, indice, complément, distribution spéciale), date de mise à jour, `file_name`.
- **dirigeant_pp** : SIREN, date de mise à jour, identité (nom, nom_usage, prénoms concaténés, genre, date de naissance), rôle (`roleEntreprise`), nationalité, situation matrimoniale, `file_name`.
- **dirigeant_pm** : SIREN entreprise, SIREN dirigeant nettoyé, dénomination, rôle, pays, forme juridique, date de mise à jour, `file_name`.
- **immatriculation** : SIREN, dates (immatriculation, radiation via priorisation radiations>effet>cessation, début d'activité, clôture), indicateur associé unique, capital social/variable, devise, durée, nature d'entreprise sérialisée (forme principale + activités principales), date de mise à jour, `file_name`.
- **etablissement** : SIREN/SIRET, date de mise à jour, `file_name` (inclut siège et établissements secondaires).
- **activite** : SIREN/SIRET, codes de catégorie/APE, indicateurs principal/prolongement/APE/EIRL, date de début, forme d'exercice, catégorisations fines, date de mise à jour, `file_name`.

### 5.3 Règles de mapping essentielles
- `updatedAt` alimente `date_mise_a_jour` dans toutes les tables pour aligner la temporalité.
- Les rôles dirigeants proviennent de `composition.pouvoirs[].roleEntreprise`, les libellés ne sont pas utilisés ; les adresses personnelles des dirigeants sont ignorées.
- `Adresse` est structurée puis concaténée via `format_address()` pour `unite_legale.adresse` ; les adresses d'établissements sont copiées champ à champ dans `siege`.
- `get_nature_entreprise_list` combine `formeExerciceActivitePrincipale` et les activités principales (siège + établissements) pour alimenter `immatriculation.nature_entreprise`.
- Les champs JSON non exploités (ex. `origin`, `formality.companyName`, `formality.codeAPE`, adresses privées) sont conservés dans les modèles pour évolutivité mais ignorés dans le mapping actuel.

### 5.4 Enrichissement des tables SIRENE
- **unite_legale** : mise à jour des lignes existantes (`from_rne`, `date_mise_a_jour_rne`) puis insertion des SIREN manquants via `INSERT OR IGNORE` avec l'ensemble des colonnes RNE.
- **siege** : mise à jour de `date_mise_a_jour_rne`, insertion des sièges manquants avec les attributs d'adresse et d'enseigne RNE.
- **dirigeant_pp / dirigeant_pm** : tables reconstruites depuis `db_rne` avec prétraitements pandas (uppercase, déduplication, agrégation des rôles) et création d'index `siren`.
- **immatriculation** : copie complète (`SELECT DISTINCT`) depuis `db_rne`, index `siren` recréé.
- **Tables dérivées** : historiques (`historique_*`, `ancien_siege`) et dates de fermeture recalculés après l'injection RNE.

## 6. Règles de gestion et garde-fous
1. **Ordre d’ingestion** : stock (si aucune date) → flux journaliers complets (dernier fichier ignoré) → déduplication locale → contrôles volumétriques → publication MinIO → mise à jour des métadonnées.
2. **Upsert strict** : suppression préalable des données du SIREN dans toutes les tables RNE avant réinsertion pour refléter la dernière version.
3. **Dédoublonnage global + VACUUM** : reconstructions `SELECT DISTINCT` post-injection puis compression SQLite.
4. **Flux partiels** : suppression des fichiers locaux en erreur et reprise automatique ; dernier fichier quotidien volontairement exclu pour éviter une journée en cours.
5. **Pagination résiliente** : adaptation dynamique de `pageSize` et rafraîchissement token ; gestion des erreurs HTTP 401/403/429/500.
6. **Traitements par blocs** : lecture/insertion par chunks de 100 000 lignes pour les dirigeants et l'injection initiale afin de maîtriser la mémoire.
7. **Seuils bloquants** : volumes minimum imposés pour UL, dirigeants et immatriculations ; échec du DAG en cas d'anomalie.
8. **Traçabilité** : `file_name` stocke l'origine JSON et les bases sont nommées par dernière date de flux traité pour identifier chaque version.

## 7. Points sensibles et limites
- **Taille des fichiers** : stock et flux peuvent atteindre plusieurs Go ; l'écriture ligne à ligne puis la compression `.gz` évitent la saturation mémoire au prix de fortes E/S.
- **Performance SQLite** : consolidation monothreadée pouvant durer plusieurs heures ; `VACUUM` régulier indispensable pour maintenir les performances.
- **Flux incomplets** : exclusion du dernier flux et possibilité de devoir rejouer plusieurs jours après interruption, rallongeant le temps de traitement.
- **API RNE** : limites de rate limiting et erreurs « Allowed memory size » nécessitant retries prolongés et supervision active.

## 8. Schéma fonctionnel
```mermaid
flowchart TD
    A[FTP INPI (stock ZIP)] -->|get_stock.sh + RneStockProcessor| B[MinIO rne/stock/]
    A2[RNE API companies/diff (flux J)] -->|ApiRNEClient + get_every_day_flux| C[MinIO rne/flux/ (.json.gz)]
    B --> D[Processus fill_rne_database]
    C --> D
    D -->|Validation Pydantic + mapping + déduplication| E[Base SQLite RNE versionnée]
    E -->|Compression + versioning| F[MinIO rne/database/ + latest_rne_date.json]
    F -->|get_rne_database| G[ETL SIRENE SQLite]
    G -->|Prétraitements pandas + SQL| H[Tables SIRENE enrichies (UL, sièges, dirigeants, immat)]
    H --> I[Annuaire enrichi + notifications Mattermost]
```
