# Modèle de données final (SQLite RNE)

Ce document décrit le schéma final de la base SQLite produite par le pipeline RNE après traitement des stocks et flux INPI. Les sources proviennent des JSON ND récupérés auprès de l'API RNE (`companies/diff`) ou du stock FTP, ingérés puis mappés dans les tables cibles. Les types indiqués sont ceux créés dans SQLite.

## Table `unite_legale`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| Stock/flux INPI (JSON RNE) | JSON | `siren` | Copie directe du SIREN de la formalité. | `siren` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `identite.entreprise.denomination` | Copie directe quand l'identité entreprise est disponible. | `denomination` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `dirigeants[0].descriptionPersonne.{nom, nomUsage, prenoms}` | Pour une personne physique, reprend l'identité du premier dirigeant détecté. | `nom`, `nom_usage`, `prenom` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `identite.entreprise.nomCommercial` | Copie directe. | `nom_commercial` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `createdAt` ou `formality.content.natureCreation.dateCreation` | Choix prioritaire : timestamp de création, sinon date déclarée dans la formalité. | `date_creation` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `updatedAt` | Copie directe de la date de mise à jour. | `date_mise_a_jour` | DATE |
| Stock/flux INPI (JSON RNE) | JSON | `formality.content.formeExerciceActivitePrincipale` ou activité principale détectée | Ajout du code de forme d'exercice principale ; alimente aussi `nature_entreprise` dans l'`immatriculation`. | `forme_exercice_activite_principale` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `formality.content.natureCessationEntreprise.etatAdministratifInsee` | Copie directe de l'état administratif. | `etat_administratif` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `formality.formeJuridique` puis `natureCreation.formeJuridique` puis `identite.entreprise.{formeJuridique, formeJuridiqueInsee}` | Sélection hiérarchique de la meilleure forme juridique disponible. | `nature_juridique` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `identite.entreprise.effectifSalarie` | Copie directe de la tranche d'effectif. | `tranche_effectif_salarie` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | Adresse siège `adresseEntreprise.adresse.*` | Mapping champ à champ vers une adresse structurée puis concaténée. | `adresse` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `formality.diffusionINSEE` | Copie directe. | `statut_diffusion` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |

## Table `siege`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| Stock/flux INPI (JSON RNE) | JSON | `siren` | Copie directe. | `siren` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `etablissementPrincipal.descriptionEtablissement.siret` | Copie directe. | `siret` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `etablissementPrincipal.descriptionEtablissement.{enseigne, nomCommercial}` | Copie directe. | `enseigne`, `nom_commercial` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `etablissementPrincipal.adresse.{pays, codePays, commune, codePostal, codeInseeCommune, voie, numVoie, typeVoie, indiceRepetition, complementLocalisation, distributionSpeciale}` | Mapping champ à champ vers l'adresse du siège. | Adresse détaillée | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| Stock/flux INPI (JSON RNE) | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |

## Table `dirigeant_pp`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| Stock/flux INPI (JSON RNE) | JSON | `composition.pouvoirs[].individu.descriptionPersonne.siren` (hérité) | Copie du SIREN UL associé. | `siren` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| Stock/flux INPI (JSON RNE) | JSON | `descriptionPersonne.{nom, nomUsage, prenoms, genre, dateDeNaissance, nationalite, situationMatrimoniale}` | Copie directe, concaténation des prénoms si liste. | Identité (`nom`, `nom_usage`, `prenoms`, `genre`, `date_de_naissance`, `nationalite`, `situation_matrimoniale`) | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `composition.pouvoirs[].roleEntreprise` | Copie directe du code rôle. | `role` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |

## Table `dirigeant_pm`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| Stock/flux INPI (JSON RNE) | JSON | `siren` | Copie du SIREN UL associé. | `siren` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `composition.pouvoirs[].entreprise.siren` | Nettoyage des espaces avant stockage. | `siren_dirigeant` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| Stock/flux INPI (JSON RNE) | JSON | `composition.pouvoirs[].entreprise.denomination` | Copie directe. | `denomination` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `composition.pouvoirs[].entreprise.roleEntreprise` | Copie directe du rôle. | `role` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `composition.pouvoirs[].entreprise.pays` | Copie directe. | `pays` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `composition.pouvoirs[].entreprise.formeJuridique` | Copie directe. | `forme_juridique` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |

## Table `immatriculation`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| Stock/flux INPI (JSON RNE) | JSON | `siren` | Copie directe. | `siren` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| Stock/flux INPI (JSON RNE) | JSON | `identite.entreprise.dateImmat` | Copie directe. | `date_immatriculation` | DATE |
| Stock/flux INPI (JSON RNE) | JSON | `detailCessationEntreprise.{dateRadiation,dateEffet,dateCessationTotaleActivite}` | Sélection de la première date disponible dans l'ordre radiation → effet → cessation totale. | `date_radiation` | DATE |
| Stock/flux INPI (JSON RNE) | JSON | `identite.entreprise.indicateurAssocieUnique` | Copie directe. | `indicateur_associe_unique` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `description.montantCapital` | Copie directe. | `capital_social` | REAL |
| Stock/flux INPI (JSON RNE) | JSON | `description.dateClotureExerciceSocial` | Copie directe. | `date_cloture_exercice` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `description.duree` | Copie directe. | `duree_personne_morale` | INT |
| Stock/flux INPI (JSON RNE) | JSON | `nature_entreprise` dérivée | Ensemble des formes d'exercice détectées (siège + établissements) sérialisées en liste. | `nature_entreprise` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `identite.entreprise.dateDebutActiv` | Copie directe. | `date_debut_activite` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `description.capitalVariable` | Copie directe. | `capital_variable` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `description.deviseCapital` | Copie directe. | `devise_capital` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |

## Table `etablissement`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| Stock/flux INPI (JSON RNE) | JSON | `siren` | Copie directe. | `siren` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `autresEtablissements[].descriptionEtablissement.siret` | Copie directe. | `siret` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| Stock/flux INPI (JSON RNE) | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |

## Table `activite`

| Data Source | Type source | Champ source | Transformation / Logique | Champ cible | Type cible |
| --- | --- | --- | --- | --- | --- |
| Stock/flux INPI (JSON RNE) | JSON | `siren` | Repris du SIREN UL courant. | `siren` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `activites[].siret` ou SIRET d'établissement | Copie du SIRET porteur de l'activité. | `siret` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `activites[].categoryCode` | Copie directe. | `code_category` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `activites[].indicateurPrincipal` | Copie directe. | `indicateur_principal` | BOOLEAN |
| Stock/flux INPI (JSON RNE) | JSON | `activites[].indicateurProlongement` | Copie directe. | `indicateur_prolongement` | BOOLEAN |
| Stock/flux INPI (JSON RNE) | JSON | `activites[].dateDebut` | Copie directe. | `date_debut` | DATE |
| Stock/flux INPI (JSON RNE) | JSON | `activites[].formeExercice` | Copie directe. | `form_exercice` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `activites[].categorisationActivite{1,2,3}` | Copie directe. | `categorisation_activite1/2/3` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `activites[].indicateurActiviteeApe` | Copie directe. | `indicateur_activitee_ape` | BOOLEAN |
| Stock/flux INPI (JSON RNE) | JSON | `activites[].codeApe` | Copie directe. | `code_ape` | TEXT |
| Stock/flux INPI (JSON RNE) | JSON | `activites[].activiteRattacheeEirl` | Copie directe. | `activite_rattachee_eirl` | BOOLEAN |
| Stock/flux INPI (JSON RNE) | JSON | `updatedAt` | Copie directe. | `date_mise_a_jour` | DATE |
| Stock/flux INPI (JSON RNE) | JSON | Nom du fichier JSON traité | Traçabilité du fichier source. | `file_name` | TEXT |
