# CS BAG v2 — Notes détaillées pour la section 4.2 du rapport

> Document de travail synthétisant : (a) l'historique des commits de la branche `develop`
> du dépôt `C:\CSBAG\appinventoring`, (b) les spécifications fonctionnelles Lot 1 / Lot 2 / Lot 3,
> (c) les scripts de migration SQL (`sql/V001` … `V026`).
> Objectif : disposer de toute la matière pour rédiger la section 4.2 « CS BAG v2 ».

---

## 1. Contexte et objectif du projet

**CS BAG** (Inventory / *application-web-Inventory*) est un **outil interne** de l'équipe
*Collaborative Solutions* de VINCI Energies. Il constitue un **inventaire applicatif** : il
recense les applications gérées par l'équipe, leurs modules, leurs référents, leurs ressources
techniques (bases de données, comptes de service, certificats, ressources Azure, serveurs
d'exploitation, applications Azure AD, frameworks…) et l'historique de leurs mises en production.

- **Accès restreint** : l'outil est réservé à l'équipe et s'utilise avec un **compte d'administration
  (ADM)**.
- **Enjeu** : la version historique (v1) souffrait d'une dette technique lourde (framework obsolète,
  modèle de données dénormalisé, ergonomie datée) et devait être fiabilisée et enrichie.

### Objectif de la v2
Moderniser et fiabiliser l'application, puis l'enrichir de nouvelles fonctionnalités,
réparties sur **trois lots** de spécifications (Lot 1, Lot 2, Lot 3).

---

## 2. Stack technique (vérifié dans le dépôt)

| Couche | Technologie | Détail |
|---|---|---|
| Front-end | **Angular 20** (`@angular/core ^20.0.0`) + Angular Material | SPA, migré depuis Angular 14 |
| Back-end | **ASP.NET Core .NET 10.0** | API REST, migré depuis .NET 6 (→ 8 → 10) |
| ORM / Data | **Entity Framework Core 10** (SqlServer) | Pattern *repository*, mapping AutoMapper |
| Base de données | **SQL Server** | Migrations versionnées `V001`…`V026` + scripts de rollback |
| Authentification | **Microsoft Entra ID** via OpenID Connect (`Microsoft.Identity.Web 4.5`) | Route guards côté Angular |
| Secrets | **Azure Key Vault** (`VE.CS.Libraries.Azure.KeyVault`) | Chaînes de connexion / secrets hors code |
| Observabilité | **Serilog** + **Application Insights** | Journalisation structurée |
| Batch mail | Application console / WebJob | Mail mensuel (comptes de service, expirations) |
| Qualité / Sécurité | **SonarQube** + **Checkmarx (CxOne)** | Intégrés à la CI GitLab (template VESI 2025) |

Modules de la solution : `Inventory.WebApp` (API + SPA), `Inventory.WebApp.Domain`,
`Inventory.WebApp.Models`, `Inventory.WebApp.Api.Models`, `Inventory.WebApp.Tests`,
`Inventory.Console.Secret` (traitements batch / secrets).

---

## 3. L'existant (v1) et ses limites

La v1 présentait :
- Un **front Angular 14** (voire des vestiges Angular 9/10/12 dans l'historique) devenu obsolète,
  incompatible avec les correctifs de sécurité récents.
- Un **back .NET 6** avec des **vulnérabilités NuGet** (dont AutoMapper) et des alertes SonarQube.
- Un **modèle de données dénormalisé** : de nombreuses valeurs (frameworks, types de module,
  comptes de service, serveurs d'exploitation, applications Azure, bases de données) étaient
  stockées en **texte libre**, source de **doublons** et d'incohérences.
- Une **ergonomie datée** : pas de barre de recherche, pas de loaders, filtres limités,
  formulaires peu lisibles.

Ces limites justifient la double démarche de la v2 : **remise à niveau technique** (migrations,
vulnérabilités, normalisation) **puis enrichissement fonctionnel** (les trois lots).

---

## 4. Socle technique commun (transverse aux 3 lots)

Réalisé principalement en amont / pendant le Lot 1, ce socle conditionne tout le reste.

### 4.1 Montée de version Angular (14 → 20)
Migration incrémentale, une version majeure à la fois pour maîtriser les *breaking changes* :
`15 → 16 → 17 → 18 → 19 → 20`.
- Correction de la **dépréciation `ngModel` avec `formControlName`**.
- Corrections de design consécutives à la migration.
- Résolution de failles Angular.

### 4.2 Montée de version .NET (6 → 8 → 10)
- Migration `.NET 6 → .NET 8` puis `.NET 8 → .NET 10`.
- **Correction des vulnérabilités NuGet** (notamment AutoMapper).
- Résolution d'alertes SonarQube (`LoggerFactory`, gestion des exceptions, `TypeLoadException`,
  warnings de compilation).
- Passage au format **SDK-style** des projets (suppression de `packages.config`).

### 4.3 Sécurité et qualité
- Mise en place de **CxOne / Checkmarx** dans la CI ; résolution de vulnérabilités
  **critiques (SCA)** et **haute gravité**.
- **Route guards** côté Angular (protection des routes).
- **Suppression des secrets Azure / SharePoint** des réponses d'API.
- Intégration **Azure Key Vault** (startup, profil UAT dans `launchSettings`, librairie interne
  depuis GitLab).
- Nettoyage continu des alertes SonarQube (complexité cognitive, duplications, captions de tables,
  descriptions).

### 4.4 Bonnes pratiques REST & nomenclature
- **Endpoints alignés sur les conventions REST** (passage des identifiants par *body* plutôt que
  par lien, séparation par contrôleur : `Equipe`, `Référent`…).
- **Uniformisation en anglais** au niveau de la couche Data (variables, entités).
- Changement de nomenclature métier : le rôle **« développeur » devient « référent »**.

### 4.5 Normalisation de la base de données (dette technique majeure)
Transformation des champs texte libre en **tables de référence** avec clés étrangères,
dédoublonnage et ajout de clés primaires. Scripts SQL associés :

| Migration | Objet normalisé |
|---|---|
| `V011_NormalisationFrameworks` | Frameworks |
| `V012_NormalisationTypes` | Types de module |
| `V013_NormalisationComptesServices` | Comptes de service |
| `V015_NormalisationServeursExploitation` | Serveurs d'exploitation |
| `V016_NormalisationApplicationsAzure` | Applications Azure |
| `V017_NormalisationBddUtilise` | Bases de données |
| `V019_NettoyageDoublonsEtAjoutPK` | Dédoublonnage + clés primaires |

Chaque normalisation a été menée **de bout en bout** : script SQL (+ rollback), entité EF Core,
mapping, API et écran Angular. Ajout de **validators** sur les entités de module.

> Gouvernance des migrations : toutes les migrations disposent d'un **script de rollback dédié**
> (chaque rollback n'annule que sa propre migration) et de scripts **PreFlight / PreBackup**
> (`V000_PreFlight_*`, `V019_PreBackup_*`) pour sécuriser les mises en production.

---

## 5. LOT 1 — Refonte fonctionnelle et ergonomique

> Périmètre : branche `maintenance-lot1`. C'est le lot le plus volumineux
> (fondations + refonte UI de toutes les pages).

### 5.1 Page d'accueil
- **Bouton « Ajouter une nouvelle application »** repositionné **au-dessus** de la liste
  (et libellé corrigé).
- **Loader** pendant le chargement de la page d'accueil.
- **Filtre d'équipe** *Tout / BU / Group* (front + back).
- **Barre de recherche** (style aligné sur la maquette).
- Refonte complète du design de la page (home), titres centrés, remplacement des anciens loaders.

### 5.2 Page « Tout voir »
- **Filtre par équipe** (avec option « Tout » et « sans équipe »).
- **Colonne Équipe** et **description** de la table.
- **Lien vers l'application** sur chaque ligne (colonne Application cliquable).
- **Référents** : filtre + colonne dédiés (avec auto-complétion).
- **Détail d'une ligne** dépliable dans le tableau.
- Filtre par framework nettoyé (suppression de la version dans le filtre).
- **Optimisation des performances** : *split queries* + `forkJoin` côté front pour réduire les
  allers-retours.

### 5.3 Certificats
- **Refonte de la page** (« remaster »), bouton retour, correction du bouton supprimer non visible,
  logo tronqué, rayons/bordures.
- **Popup de confirmation avant suppression** d'un certificat.
- Interdiction de créer **deux certificats/comptes avec la même date**.

### 5.4 Ajouter / Modifier une application
- Nouveaux champs : **Description**, renommage du champ **Logo**, liens (**GED VESI, HELP,
  dashboard Azure**), nettoyage des placeholders.
- **Sélection de plusieurs équipes** pour une même application (multi-sélection, `V001_AjoutEquipes`,
  `V002_AjoutColonnesApplication`).
- **Tableau des référents** avec **colonne Fonction** :
  - Nouvelle **table `Fonction`** (`V003_AjoutTableFonctions`), champ passé en **liste déroulante**,
    auto-complétion des référents, possibilité de modifier la fonction même si le référent existe,
    retrait du choix « Aucune ».
- Refonte du design des pages Ajouter / Modifier (icônes, largeur, responsive), **loaders**,
  **validators** de formulaire.

### 5.5 Détails de l'application
- Refonte du design (informations générales, résumé de module).
- Affichage de la **table des modules** et de la **table des référents** même lorsqu'elles sont
  vides.
- **Colonne Type** de module affichée (`V022`… non : `V004`/`V005`).
- **Modifier l'ordre des modules** (`V004_AjoutOrdreModules`) : le bouton n'apparaît que si
  `nbModule ≥ 1`, seule l'icône de la colonne concernée bouge au clic.
- **Historique des livraisons** par module (`V007_AjoutHistoriqueLivraison`,
  `V010_MigrationDateDerLivrVersHistorique`) : nouvelle page, redesign, correction du calcul de la
  période du nombre de livraisons, dates forcées au format FR.
- Loaders sur les pages Détails.

### 5.6 Ajouter / Modifier un module
- Refonte de la page (design, tailles de tableaux, boutons ajoutés aux tableaux).
- **Recherche de comptes de service / certificats** dans le formulaire.
- Colonnes **Type de ressource** (`V005_AjoutTypeRessource`), **VM** et **Dossier VM**
  (`V014_AjoutDossierVM`) dans l'association serveur d'exploitation.
- Changement du **processus de modification d'un module** (optimisation des allers-retours en BDD).
- Gestion des cas d'erreur (bordure rouge sur le champ « type »), blocage de la soumission après
  clic sur « Entrer ».

### 5.7 Comptes de service (nouvelle page)
- **Nouvelle page dédiée** aux comptes de service.
- Tri renommé « Trier par Nom » → **« Trier par compte »**.
- Date d'expiration des comptes de service (`V009_AjoutDateExpCptServices`).
- **Mail de début de mois** : ajout de la **table des comptes de service** dans le mail envoyé
  (rappel d'expiration).
- Correction d'un problème d'enregistrement des comptes de service.

### 5.8 Technique — définir les listes / propriétés
- Ajout de la propriété **Type** et de la propriété **Date de dernière mise à jour**
  (`V006_AjoutDateDerMaj`).
- Passage de champs texte en **listes déroulantes** (frameworks, types…), vérification des doublons.

---

## 6. LOT 2 — Ordre, archivage, types d'application, thumbprint

> Périmètre : commits entre `maintenance-lot1` et `maintenance-lot2`.

### 6.1 Modifier l'ordre de certains éléments
- **`V018_AjoutOrdreAssociationsModule`** : pouvoir modifier l'ordre d'affichage des associations
  d'un module (BDD, ressources Azure, comptes de service…).
- Désactivation de la réorganisation pour les applications **archivées**.

### 6.2 Application Azure AD
- **`V020_AjoutNomDescriptionApplicationsAzure`** : ajout des colonnes **Nom** et **Description**.
- Affichage sous forme de **liste déroulante** + **colonne Description**.

### 6.3 Historique de livraison (enrichi)
- **`V021_AjoutColonnesHistoriqueLivraison`** : ajout des champs **Document de MEP**,
  **Numéro de MEP**, **Fonctionnel associé**.
- Corrections d'affichage : champs MEP longs (retour à la ligne, défilement des pop-up),
  validation des **longueurs maximales** des champs.

### 6.4 Archiver un projet
- **`V022_AjoutArchivageApplication`** : statut **archivé (0/1)**.
- **Bouton d'archivage + popup de confirmation**, passage en **lecture seule** de l'application
  archivée (protection **back-end ET front-end** des routes de modification).
- **Filtre projets actifs / archivés** sur la page d'accueil et la page « Tout voir ».

### 6.5 Type d'application (Dali / Elastik)
- **`V023_AjoutApplicationsDaliElastik`** : tables dédiées aux **applications Dali** et
  **Elastik**, affichées dans la page « Tout voir ».

### 6.6 Thumbprint (empreinte de certificat)
- **`V024_AjoutThumbprintCertificats`** : ajout de la colonne **Thumbprint**.
- Implémentation **back + front**, **colonne + filtre + recherche** (page « Tout voir »),
  **validation du format** et normalisation des thumbprints, validation de la longueur maximale.

### 6.7 Sécurité / fiabilisation du lot 2
- Résolution de vulnérabilités **Checkmarx** (critiques + haute gravité), route guards.
- Scripts de **pré-vol** (`V000_PreFlight_MEP_Complet` couvrant V001→V024), scripts de
  **pré-sauvegarde et rollback** (V019, V022), **option de redirection des e-mails** pour les tests.

---

## 7. LOT 3 — Référentiels, historique des actions, taxonomies, alertes

> Périmètre : commits de `develop` postérieurs à `maintenance-lot2`.

### 7.1 Administrer les référentiels
Nouveau **menu d'administration** avec des pages dédiées pour gérer les données de référence,
avec **popup de modification** pour chaque référentiel :

| Écran d'administration | Table / entité |
|---|---|
| Frameworks | `Frameworks` |
| Bases de données | `BddUtilise` |
| Serveurs d'exploitation | `ServeursExploitation` |
| Types de ressources | `TypeRessource` |
| Référents | `DeveloppeursRef` |
| Fonctions | `Fonctions` |
| Applications Azure | `ApplicationsAzure` |
| Équipes | `Equipes` |

Détails d'implémentation : gestion administrative des **fonctions de référence**, des **frameworks**
(refonte du dialogue de renommage, **tous les frameworks rendus disponibles** pour l'association aux
modules), des **référents et bases de données** (ergonomie du formulaire d'ajout de BDD améliorée),
des **types de ressources**, des **applications Azure**, des **serveurs d'exploitation** et des
**équipes**.

### 7.2 Historique des actions (audit)
- **`V025_AjoutHistoriqueActions`** : tables d'historique des actions utilisateur.
- **Page dédiée** listant les actions : **archivage / suppression / ajout / modification**.
- Implémentation de l'**audit des actions utilisateur** + interface, puis plusieurs itérations :
  gestion des collections, **corrélation automatique**, **hiérarchisation et consolidation** de
  l'affichage détaillé, refonte de l'interface, optimisation.
- Script de purge : `PurgeHistoriqueActions.sql`.

### 7.3 Taxonomies
- **`V026_AjoutTaxonomiesModule`** : champ **`IntegreTaxonomies`** sur les modules
  (case à cocher côté module).
- **Affichage et filtrage** des modules par intégration de taxonomies (filtre **Oui / Non**),
  affichage dans les détails et la page « Tout voir ».

### 7.4 Alertes d'expiration Dali / Elastik
- Ajout des **alertes d'expiration** pour les applications **Dali** et **Elastik**
  (surveillance proactive, en lien avec le batch de notification).

---

## 8. Correspondance features ↔ spécifications ↔ SQL (récapitulatif)

| Fonctionnalité (spec) | Branche `feature/*` | Migration SQL | Lot |
|---|---|---|---|
| Barre de recherche accueil | `ajout-barre-de-recherche-accueil` | — | 1 |
| Filtre d'équipe (Tout/BU/Group) | `ajout-filtre-equipe` | `V001` | 1 |
| Multi-équipes / colonnes app | — | `V001`,`V002` | 1 |
| Table Fonction (référents) | — | `V003` | 1 |
| Ordre des modules | — | `V004` | 1 |
| Type de ressource | — | `V005` | 1 |
| Date dernière MàJ | — | `V006` | 1 |
| Historique de livraison | `historiqueLivraison` | `V007`,`V010` | 1 |
| Date exp. comptes de service | — | `V009` | 1 |
| Normalisation BDD | `normalisation-table-bdd`, `normalisation-entites` | `V011`–`V017` | 1 |
| Dossier VM | — | `V014` | 1 |
| Ordre associations module | `ajoutOrdreEntites` | `V018` | 2 |
| Nettoyage doublons + PK | — | `V019` | 2 |
| Nom/Description Apps Azure | `ameliorationTableauAzureAD` | `V020` | 2 |
| Colonnes historique livraison (MEP) | `historiqueLivraison` | `V021` | 2 |
| Archivage projet | `archivageProjet` | `V022` | 2 |
| Applications Dali / Elastik | `nvTablesDaliElastic` | `V023` | 2 |
| Thumbprint certificats | `gestionThumbprint` | `V024` | 2 |
| Historique des actions | `historiqueDesActions` | `V025` | 3 |
| Administrer les référentiels | `gestionRéférentiels` | (tables existantes) | 3 |
| Taxonomies module | `taxonomie` | `V026` | 3 |
| Alertes Dali / Elastik | `alertes-dali-elastik` | — | 3 |

---

## 9. Ma contribution personnelle (à préciser dans le rapport)

> À adapter selon ce que vous avez réellement porté. Éléments candidats repérés dans l'historique :
- Participation à la **normalisation du modèle de données** (tables de référence + migrations +
  rollbacks + PreFlight).
- Implémentation de fonctionnalités du Lot 2/3 (ex. **thumbprint**, **archivage**,
  **historique des actions**, **référentiels**, **taxonomies** — à confirmer).
- Résolution de vulnérabilités **Checkmarx / SonarQube** et fiabilisation de la CI.
- Contribution à la **montée de version** (.NET et/ou Angular).

*(Renseigner précisément vos commits pour cette sous-section.)*

---

## 10. Points structurants à mettre en avant dans la rédaction

1. **Double nature du projet** : dette technique (migrations, sécurité, normalisation) **+**
   valeur métier (nouvelles fonctionnalités des 3 lots).
2. **Approche par lots** : gouvernance claire (branches `maintenance-lot1/2`, `develop`),
   migrations SQL versionnées avec rollback et scripts de pré-vol → **maîtrise du risque en MEP**.
3. **Qualité / sécurité intégrées** : SonarQube + Checkmarx dans la CI, secrets dans Key Vault,
   authentification Entra ID, route guards.
4. **Normalisation** comme illustration concrète d'un refactoring de fond (texte libre → tables de
   référence) — bon point d'ancrage pour l'état de l'art (dette technique, normalisation des BDD).
5. **Ergonomie** : loaders, recherche, filtres, responsive — amélioration mesurable de
   l'expérience utilisateur.
