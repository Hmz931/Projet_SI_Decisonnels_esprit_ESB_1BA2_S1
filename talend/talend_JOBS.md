# ⚙️ Documentation Technique Complète - Jobs Talend ETL
## Projet SI Décisionnels · Entrepôt de Données RH

> **Auteurs** : Hamza Bouguerra · Fares Messedi - ESPRIT 1BA2  
> **Deadline** : 14 Mars 2026  
> **ETL** : Talend Open Studio for Data Integration  
> **SGBD** : PostgreSQL 18 (pgAdmin 4)  
> **Base source** : `rh_entreprise` | **Base cible** : `rh_dw`

---

## 📐 Architecture du Pipeline ETL

```
┌─────────────────────────────────────────────────────────────┐
│                     SOURCES OPÉRATIONNELLES                  │
│                                                             │
│  PostgreSQL rh_entreprise    CSV              Excel         │
│  ┌─────────────────────┐  ┌──────────────┐  ┌──────────┐  │
│  │ employes (112 rows) │  │ absences     │  │formations│  │
│  │ salaires (1200 rows)│  │ _presences   │  │.xlsx     │  │
│  └─────────────────────┘  │ .csv (582)   │  │(266 rows)│  │
│                            └──────────────┘  └──────────┘  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                ┌───────────▼───────────┐
                │   TALEND OPEN STUDIO  │
                │   6 Jobs ETL          │
                │   Extract → Transform │
                │          → Load       │
                └───────────┬───────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                  DATA WAREHOUSE rh_dw                        │
│                                                             │
│  dim_employe  dim_service  dim_temps  dim_absence           │
│  dim_motif_absence  dim_formation  →  fait_rh               │
└─────────────────────────────────────────────────────────────┘
```

---

## 📁 Analyse Détaillée des Sources Réelles

### Source 1 - PostgreSQL `rh_entreprise` : table `employes`

**Structure de la table :**
```sql
CREATE TABLE IF NOT EXISTS employes (
    matricule       VARCHAR(10),
    nom             VARCHAR(50),
    prenom          VARCHAR(50),
    service         VARCHAR(50),
    salaire_mensuel DECIMAL(10,2),
    ville           VARCHAR(50),
    email           VARCHAR(100)
);
```

**Échantillon de données réelles :**
```
'EMP001','Blaxis','Orvan','Ressources Humaines',4402,'Ariana','orvan.blaxis@entreprise.tn'
'EMP002','Mendris','Relia','Informatique',3963,'Kairouan','relia.mendris@entreprise.tn'
'EMP004','Dravon','Tavin','Juridique',2249,'Monastir',NULL
'EMP009','Brendis','Veria','Marketing',NULL,'Sousse','veria.brendis@entreprise.tn'
'EMP010','Grelvon','Nemia','Logistique',1111,'Sfax','nemia.grelvon@entreprise.tn'
'EMP010','GRELVON','Nemia','Logistique',1111,'Sfax','nemia.grelvon@entreprise.tn'  ← DOUBLON
```

**Statistiques mesurées :**

| Indicateur | Valeur |
|---|---|
| Total INSERT | **112** (dont 12 doublons) |
| Employés uniques | **100** |
| NULL salaire_mensuel | **14** cas (EMP009, EMP011, EMP013…) |
| NULL ville | **7** cas (EMP014, EMP017, EMP022, EMP024, EMP038, EMP055, EMP066) |
| NULL email | **9** cas (EMP004, EMP011, EMP061, EMP066, EMP069, EMP070…) |
| Salaire min / max / moy | 815 DT / 4445 DT / 2592 DT |
| Villes | Ariana · Bizerte · Gabès · Gafsa · Kairouan · Monastir · Nabeul · Sfax · Sousse · Tunis |

**10 services distincts :**
Administration · Commercial · Finance · Informatique · Juridique · Logistique · Marketing · Production · Qualité · Ressources Humaines

**12 doublons exacts (même matricule, nom en MAJUSCULES) :**
```
EMP010 → 'Grelvon'  vs 'GRELVON'     EMP053 → 'Alvorix'  vs 'ALVORIX'
EMP013 → 'Dravon'   vs 'DRAVON'      EMP057 → 'Sylmor'   vs 'SYLMOR'
EMP027 → 'Krevan'   vs 'KREVAN'      EMP063 → 'Brendis'  vs 'BRENDIS'
EMP028 → 'Tholux'   vs 'THOLUX'      EMP087 → 'Moxalis'  vs 'MOXALIS'
EMP040 → 'Perthas'  vs 'PERTHAS'     EMP095 → 'Darven'   vs 'DARVEN'
EMP046 → 'Luxorin'  vs 'LUXORIN'     EMP100 → 'Calvera'  vs 'CALVERA'
```

---

### Source 2 - PostgreSQL `rh_entreprise` : table `salaires`

**Structure de la table :**
```sql
CREATE TABLE IF NOT EXISTS salaires (
    id_salaire      INT,
    matricule       VARCHAR(10),
    salaire_mensuel DECIMAL(10,2),
    mois            INT,
    annee           INT,
    prime           DECIMAL(10,2)
);
```

**Échantillon de données réelles :**
```
1, 'EMP001', 4402, 1, 2024, 150
2, 'EMP001', NULL, 2, 2024, 100    ← NULL salaire
3, 'EMP001', 4402, 3, 2024, 300
4, 'EMP001', 4402, 4, 2024, 150
5, 'EMP001', 4402, 5, 2024,   0
```

**Statistiques mesurées :**

| Indicateur | Valeur |
|---|---|
| Total lignes | **1200** (100 emp × 12 mois) |
| NULL salaire_mensuel | **114** lignes |
| Salaire min / max / moy | 815 / 4445 / 2592 DT |
| Primes possibles | 0 · 100 · 150 · 200 · 300 DT |
| Période | 12 mois de 2024 (mois 1→12) |

---

### Source 3 - `absences_presences.csv`

**Structure (séparateur `;`) :**
```
id_absence ; matricule ; date_absence ; motif ; duree_jours ; justifie ; remarque
```

**Échantillon de données réelles :**
```
ABS0487 ; EMP093 ; 2023-09-07 ; Congé annuel        ; 5 ; Oui ;
ABS0304 ; EMP056 ; 2023-02-12 ; Accident de travail ; 2 ; Oui ;
ABS0371 ; EMP070 ; 2022-11-19 ; Maternité/Paternité ; 6 ; Oui ;
ABS0336 ; EMP063 ; 2024-08-02 ; Congé annuel        ; 3 ; Oui ;
ABS0365 ; EMP069 ; 2022-09-23 ;                     ; 6 ; Oui ;  ← motif vide
```

**Statistiques mesurées :**

| Indicateur | Valeur |
|---|---|
| Total lignes | **582** |
| Doublons exacts | **62** (10.7%) |
| NULL / vide motif | **54** (9.3%) |
| NULL / vide duree_jours | **50** (8.6%) |
| NULL / vide justifie | **53** (9.1%) |
| Colonne remarque | **582 vides** - à ignorer |
| Plage de dates | 2022-01-02 → 2024-12-27 |
| Durées possibles (jours) | 1 · 2 · 3 · 4 · 5 · 6 · 7 · 8 · 9 · 10 |
| Matricules distincts | 86 employés concernés |

**Distribution des motifs :**

| Motif | Count |
|---|---|
| Formation | 96 |
| Accident de travail | 94 |
| Maladie | 92 |
| Maternité/Paternité | 84 |
| Congé annuel | 82 |
| Congé sans solde | 80 |
| Vide (NULL) | 54 |

**Valeurs `justifie` :** `Oui` (529) · vide/NULL (53)

---

### Source 4 - `formations.xlsx` (feuille `Formations`)

**Structure Excel :**
```
ID Formation | Matricule | Nom | Prénom | Service | Intitulé | Durée (h) | Année | Coût (DT) | Statut
```

**Échantillon de données réelles :**
```
FORM0147 | EMP060 | Tholux  | Daxon  | Production        | Sécurité au travail      |  8 | 2022 |  500 | Planifiée
FORM0042 | EMP017 | Darven  | Kelia  | Ressources Humaines| ERP SAP                 | 40 | 2024 | NULL | NULL
FORM0024 | EMP010 | Grelvon | Nemia  | Logistique        | Excel avancé             | 16 | 2022 | 1000 | Complétée
FORM0014 | EMP005 | Moxalis | Gaxon  | Logistique        | Droit du travail         |NULL| 2022 | 1500 | Complétée
FORM0027 | EMP011 | Brendis | Tavin  | Commercial        | Analyse de données       | 40 | 2023 |  500 | NULL
```

**Statistiques mesurées :**

| Indicateur | Valeur |
|---|---|
| Total lignes | **266** |
| Doublons exacts | **28** (10.5%) |
| NULL Coût (DT) | **31** (11.7%) |
| NULL Statut | **30** (11.3%) |
| NULL Durée (h) | **24** (9.0%) |
| Années couvertes | 2022 · 2023 · 2024 |
| Coût min / max / moy | 500 / 2000 / 1176 DT |
| Durées possibles (h) | 8 · 16 · 20 · 24 · 32 · 40 |

**Distribution des statuts :**
En cours (89) · Complétée (77) · Planifiée (70) · NULL (30)

**Distribution par service :**
Ressources Humaines (38) · Commercial (36) · Logistique (33) · Qualité (30) · Administration (27) · Production (25) · Marketing (25) · Informatique (20) · Juridique (19) · Finance (13)

**16 intitulés de formation :**
Analyse de données · Communication professionnelle · Cybersécurité · Droit du travail · ERP SAP · Excel avancé · Leadership · Management de projet · Marketing digital · Négociation commerciale · Power BI · Python pour la data · Qualité ISO 9001 · SQL et bases de données · Sécurité au travail · Talend ETL

---

## ⚡ Ordre d'exécution des Jobs

```
J0_InitialLoad          ← créer le schéma rh_dw
        ↓
J1_Load_DIM_EMPLOYE ──┐
J2_Load_DIM_TEMPS   ──┘  ← peuvent tourner en parallèle
        ↓
J3_Load_DIM_ABSENCE
J4_Load_DIM_FORMATION
        ↓
J5_Load_FAIT_RH          ← obligatoirement EN DERNIER
```

---

## J0 - Job_InitialLoad

**Rôle :** Créer toutes les tables du DW dans PostgreSQL `rh_dw`.

> ⚠️ **Prérequis :** Créer les deux bases dans pgAdmin **avant** de lancer ce job :
> ```sql
> CREATE DATABASE rh_entreprise;
> CREATE DATABASE rh_dw;
> ```

### Composants Talend

```
[tPostgresqlConnection]
  host     = localhost
  port     = 5432
  database = rh_dw
  user     = postgres
        ↓
[tFileCheck]
  Vérifie : absences_presences.csv
  Vérifie : formations.xlsx
  → si manquant → [tDie] "Fichier source introuvable : {nom}"
        ↓
[tPostgresqlRow]
  Exécute le DDL PostgreSQL ci-dessous
        ↓
[tLogRow]
  "[OK] Schéma rh_dw créé - {timestamp}"
```

### Script DDL PostgreSQL complet

```sql
-- ═══════════════════════════════════════════════════════
-- SCHÉMA DU DATA WAREHOUSE RH - PostgreSQL
-- Projet SI Décisionnels · ESPRIT 1BA2 · 2025-2026
-- ═══════════════════════════════════════════════════════

-- Dimension Employé
-- Source : table employes (rh_entreprise)
CREATE TABLE IF NOT EXISTS dim_employe (
    sk_employe    SERIAL        PRIMARY KEY,
    matricule     VARCHAR(10)   NOT NULL UNIQUE,
    nom           VARCHAR(50),
    prenom        VARCHAR(50),
    service       VARCHAR(50),
    salaire_mensuel DECIMAL(10,2),
    ville         VARCHAR(50),
    email         VARCHAR(100)
);

-- Dimension Service
-- Extraite des services distincts de la table employes
CREATE TABLE IF NOT EXISTS dim_service (
    sk_service  SERIAL      PRIMARY KEY,
    nom_service VARCHAR(50) NOT NULL UNIQUE
);

-- Dimension Temps
-- Générée programmatiquement : 2022-01-01 → 2024-12-31
CREATE TABLE IF NOT EXISTS dim_temps (
    sk_temps      SERIAL      PRIMARY KEY,
    date_complete DATE        NOT NULL UNIQUE,
    jour          INTEGER,
    mois          INTEGER,
    annee         INTEGER,
    trimestre     INTEGER,
    semaine       INTEGER,
    nom_mois      VARCHAR(20),
    nom_jour      VARCHAR(20),
    est_weekend   SMALLINT    DEFAULT 0
);

-- Dimension Absence
-- Source : absences_presences.csv
CREATE TABLE IF NOT EXISTS dim_absence (
    sk_absence   SERIAL      PRIMARY KEY,
    id_absence   VARCHAR(10),
    matricule    VARCHAR(10),
    date_absence DATE,
    motif        VARCHAR(50),
    duree_jours  INTEGER     DEFAULT 0,
    justifie     VARCHAR(15)
);

-- Dimension Motif Absence
-- Extraite des motifs distincts du CSV
CREATE TABLE IF NOT EXISTS dim_motif_absence (
    sk_motif  SERIAL      PRIMARY KEY,
    motif     VARCHAR(50) NOT NULL UNIQUE,
    categorie VARCHAR(30)
);

-- Dimension Formation
-- Source : formations.xlsx (feuille Formations)
CREATE TABLE IF NOT EXISTS dim_formation (
    sk_formation SERIAL         PRIMARY KEY,
    id_formation VARCHAR(10),
    matricule    VARCHAR(10),
    intitule     VARCHAR(100),
    duree_heures INTEGER        DEFAULT 0,
    annee        INTEGER,
    cout         NUMERIC(10,2)  DEFAULT 0.00,
    statut       VARCHAR(20)
);

-- Table de Faits RH
-- Grain : 1 ligne par employé
CREATE TABLE IF NOT EXISTS fait_rh (
    id_fait          SERIAL        PRIMARY KEY,
    sk_employe       INTEGER       NOT NULL REFERENCES dim_employe(sk_employe),
    sk_service       INTEGER       NOT NULL REFERENCES dim_service(sk_service),
    sk_temps         INTEGER       REFERENCES dim_temps(sk_temps),
    sk_absence       INTEGER       REFERENCES dim_absence(sk_absence),
    sk_formation     INTEGER       REFERENCES dim_formation(sk_formation),
    sk_motif         INTEGER       REFERENCES dim_motif_absence(sk_motif),
    salaire_mensuel  NUMERIC(10,2),
    salaire_annuel   NUMERIC(10,2),
    prime            NUMERIC(10,2) DEFAULT 0,
    nb_jours_absence INTEGER       DEFAULT 0,
    nb_formations    INTEGER       DEFAULT 0
);
```

> 💡 **PostgreSQL vs MySQL :**
> `SERIAL` = `AUTO_INCREMENT` · `NUMERIC` = `DECIMAL` · `SMALLINT` = `TINYINT(1)`

---

## J1 - Job_Load_DIM_EMPLOYE

**Source :** PostgreSQL `rh_entreprise` → tables `employes` + `salaires`  
**Cibles :** `dim_employe` + `dim_service` dans `rh_dw`

### Anomalies réelles à traiter

```
112 INSERT → 100 employés uniques (12 doublons à éliminer)

Doublons : même matricule, nom en MAJUSCULES
  EMP010 → 'Grelvon'  vs 'GRELVON'   · EMP053 → 'Alvorix'  vs 'ALVORIX'
  EMP013 → 'Dravon'   vs 'DRAVON'    · EMP057 → 'Sylmor'   vs 'SYLMOR'
  EMP027 → 'Krevan'   vs 'KREVAN'    · EMP063 → 'Brendis'  vs 'BRENDIS'
  EMP028 → 'Tholux'   vs 'THOLUX'    · EMP087 → 'Moxalis'  vs 'MOXALIS'
  EMP040 → 'Perthas'  vs 'PERTHAS'   · EMP095 → 'Darven'   vs 'DARVEN'
  EMP046 → 'Luxorin'  vs 'LUXORIN'   · EMP100 → 'Calvera'  vs 'CALVERA'

NULLs colonne employes :
  salaire_mensuel → 14 cas  (EMP009, EMP011, EMP013, EMP017, EMP025…)
  email           →  9 cas  (EMP004, EMP011, EMP061, EMP066, EMP069…)
  ville           →  7 cas  (EMP014, EMP017, EMP022, EMP024, EMP038, EMP055, EMP066)
```

### Flux Talend

```
[tPostgresqlConnection - rh_entreprise]
        ↓
[tPostgresqlInput]
  Query : SELECT * FROM employes
        ↓
[tMap - 1. Normalisation de la casse]
  nom    → StringHandling.UPCASE(StringHandling.TRIM(row.nom))
  prenom → StringHandling.UPCASE_FIRST_LOWER(StringHandling.TRIM(row.prenom))
        ↓
[tUniqRow]
  Clé de déduplication : matricule
  Flux rejeté → [tLogRow] "DOUBLON éliminé : {matricule} · {nom}"
        ↓
[tMap - 2. Corrections NULL + séparation flux]
  Flux 1 ──→ [tPostgresqlOutput] dim_employe   (INSERT)
  Flux 2 ──→ [tPostgresqlOutput] dim_service   (INSERT + ON CONFLICT DO NOTHING)
```

### Expressions tMap - Flux 1 (dim_employe)

```java
out1.matricule        = row.matricule
out1.nom              = row.nom      // déjà normalisé (UPCASE)
out1.prenom           = row.prenom   // déjà normalisé
out1.service          = row.service
out1.salaire_mensuel  = (row.salaire_mensuel == null) ? 0.0 : row.salaire_mensuel
out1.ville            = (row.ville == null || row.ville.isEmpty())
                        ? "Non renseignée" : row.ville
out1.email            = (row.email == null || row.email.isEmpty())
                        ? "non-renseigne@entreprise.tn" : row.email
```

### Expressions tMap - Flux 2 (dim_service)

```java
out2.nom_service = row.service
// ON CONFLICT (nom_service) DO NOTHING → 10 services uniques chargés
```

### Configuration tPostgresqlOutput - dim_service

```
Action          : Insert
Gestion conflit : ON CONFLICT (nom_service) DO NOTHING
```

> ⚠️ PostgreSQL n'a pas `INSERT IGNORE`. Équivalent exact :  
> `INSERT INTO dim_service (nom_service) VALUES (?) ON CONFLICT (nom_service) DO NOTHING`

### Résultat attendu

| Table | Entrée | Sortie |
|---|---|---|
| dim_employe | 112 inserts | **100 lignes** |
| dim_service | 100 lignes | **10 lignes** (1/service) |

---

## J2 - Job_Load_DIM_TEMPS

**Source :** Générée programmatiquement - aucun fichier requis  
**Cible :** `dim_temps`  
**Période :** 2022-01-01 → 2024-12-31 *(couvre toutes les dates des 3 sources)*

### Flux Talend

```
[tFixedFlowInput]
  Colonnes : date_debut (Date) = "2022-01-01"
             date_fin   (Date) = "2024-12-31"
        ↓
[tFlowToIterate]
  Génère une ligne par jour de la plage
        ↓
[tMap - Calcul des attributs temporels]
        ↓
[tPostgresqlOutput - dim_temps]
  Action  : INSERT
  Conflit : ON CONFLICT (date_complete) DO NOTHING  ← idempotent
```

### Expressions tMap - Calcul des attributs

```java
import java.util.Calendar;

Calendar cal = Calendar.getInstance();
cal.setTime(row.date_courante);

out.date_complete = row.date_courante;
out.jour          = cal.get(Calendar.DAY_OF_MONTH);
out.mois          = cal.get(Calendar.MONTH) + 1;
out.annee         = cal.get(Calendar.YEAR);
out.trimestre     = (out.mois <= 3) ? 1
                  : (out.mois <= 6) ? 2
                  : (out.mois <= 9) ? 3 : 4;
out.semaine       = cal.get(Calendar.WEEK_OF_YEAR);
out.nom_mois      = new String[]{
                    "Janvier","Février","Mars","Avril","Mai","Juin",
                    "Juillet","Août","Septembre","Octobre","Novembre","Décembre"
                    }[out.mois - 1];
out.nom_jour      = new String[]{
                    "Dimanche","Lundi","Mardi","Mercredi",
                    "Jeudi","Vendredi","Samedi"
                    }[cal.get(Calendar.DAY_OF_WEEK) - 1];
out.est_weekend   = (cal.get(Calendar.DAY_OF_WEEK) == Calendar.SATURDAY
                  || cal.get(Calendar.DAY_OF_WEEK) == Calendar.SUNDAY) ? 1 : 0;
```

### Exemples de lignes générées

| date_complete | jour | mois | annee | trimestre | nom_mois | nom_jour | est_weekend |
|---|---|---|---|---|---|---|---|
| 2022-01-01 | 1 | 1 | 2022 | 1 | Janvier | Samedi | **1** |
| 2022-01-03 | 3 | 1 | 2022 | 1 | Janvier | Lundi | 0 |
| 2024-07-15 | 15 | 7 | 2024 | 3 | Juillet | Lundi | 0 |
| 2024-12-31 | 31 | 12 | 2024 | 4 | Décembre | Mardi | 0 |

### Résultat attendu
- **1 096 lignes** dans dim_temps (3 ans : 365 + 365 + 366 jours)

> 💡 Ce job est **indépendant** - il peut tourner en parallèle de J1.

---

## J3 - Job_Load_DIM_ABSENCE

**Source :** `data/raw/absences_presences.csv` (séparateur `;`, encodage UTF-8)  
**Cibles :** `dim_absence` + `dim_motif_absence`

### Anomalies réelles à traiter

```
582 lignes brutes → 520 lignes uniques

Doublons exacts   : 62 lignes  (10.7%)  → tUniqRow (clé : id_absence+matricule+date)
NULL motif        : 54 cas     (9.3%)   → remplacer par "Non précisé"
NULL duree_jours  : 50 cas     (8.6%)   → remplacer par 0
NULL justifie     : 53 cas     (9.1%)   → remplacer par "Non renseigné"
Colonne remarque  : 582 vides           → ne pas charger dans dim_absence

Motifs valides dans le fichier :
  Formation (96) · Accident de travail (94) · Maladie (92)
  Maternité/Paternité (84) · Congé annuel (82) · Congé sans solde (80)

Valeurs justifie : 'Oui' (529) · vide (53)
```

### Flux Talend

```
[tFileInputDelimited]
  Fichier        : absences_presences.csv
  Séparateur     : ;
  Encodage       : UTF-8
  Header         : true  (ligne 1 = noms de colonnes)
  Schéma         : voir tableau ci-dessous
        ↓
[tUniqRow]
  Clé            : id_absence + matricule + date_absence
  Flux rejeté    → [tLogRow] "DOUBLON CSV éliminé : {id_absence}"
        ↓
[tReplaceList]
  Colonne motif    : "" → "Non précisé"
  Colonne justifie : "" → "Non renseigné"
        ↓
[tMap]
  Corrections NULL + séparation flux
  Flux 1 ──→ [tPostgresqlOutput] dim_absence
  Flux 2 ──→ [tUniqRow motif] ──→ [tPostgresqlOutput] dim_motif_absence
```

### Schéma tFileInputDelimited

| Colonne CSV | Nom Talend | Type Talend | Longueur | Action |
|---|---|---|---|---|
| id_absence | id_absence | String | 10 | charger |
| matricule | matricule | String | 10 | charger |
| date_absence | date_absence | Date | - | format : `yyyy-MM-dd` |
| motif | motif | String | 50 | corriger (tReplaceList) |
| duree_jours | duree_jours | Integer | - | corriger (NULL→0) |
| justifie | justifie | String | 15 | corriger (tReplaceList) |
| remarque | remarque | String | - | **ne pas mapper en sortie** |

### Expressions tMap - Flux 1 (dim_absence)

```java
out1.id_absence   = row.id_absence
out1.matricule    = row.matricule
out1.date_absence = row.date_absence
out1.motif        = row.motif      // déjà corrigé par tReplaceList
out1.duree_jours  = (row.duree_jours == null) ? 0 : row.duree_jours
out1.justifie     = row.justifie   // déjà corrigé par tReplaceList
```

### Expressions tMap - Flux 2 (dim_motif_absence)

```java
// Catégorisation métier des motifs
String m = row.motif;
out2.motif     = m;
out2.categorie = m.equals("Maladie") || m.equals("Accident de travail")
               ? "Médical"
               : m.equals("Congé annuel") || m.equals("Congé sans solde")
               ? "Congé"
               : m.equals("Maternité/Paternité") ? "Familial"
               : m.equals("Formation")            ? "Professionnel"
               : "Autre";
```

### dim_motif_absence - Contenu attendu

| sk_motif | motif | categorie |
|---|---|---|
| 1 | Accident de travail | Médical |
| 2 | Congé annuel | Congé |
| 3 | Congé sans solde | Congé |
| 4 | Formation | Professionnel |
| 5 | Maladie | Médical |
| 6 | Maternité/Paternité | Familial |
| 7 | Non précisé | Autre |

### Résultat attendu

| Table | Entrée | Sortie |
|---|---|---|
| dim_absence | 582 lignes | **520 lignes** |
| dim_motif_absence | - | **7 lignes** |

---

## J4 - Job_Load_DIM_FORMATION

**Source :** `data/raw/formations.xlsx` - feuille `Formations`  
**Cible :** `dim_formation`

### Anomalies réelles à traiter

```
266 lignes brutes → 238 lignes uniques

Doublons exacts   : 28 lignes  (10.5%)  → tUniqRow (clé : ID Formation)
NULL Coût (DT)    : 31 cas     (11.7%)  → remplacer par 0.0
NULL Statut       : 30 cas     (11.3%)  → remplacer par "Inconnu"
NULL Durée (h)    : 24 cas     (9.0%)   → remplacer par 0

16 intitulés distincts, 3 années (2022·2023·2024)
Coûts réels : 500 · 800 · 1000 · 1200 · 1500 · 2000 DT
Durées réelles : 8 · 16 · 20 · 24 · 32 · 40 heures
Statuts : Complétée (77) · En cours (89) · Planifiée (70) · NULL (30→"Inconnu")
```

### Flux Talend

```
[tFileInputExcel]
  Fichier        : formations.xlsx
  Feuille        : Formations
  Première ligne : 2  (ligne 1 = header, à ignorer)
  Schéma         : voir tableau ci-dessous
        ↓
[tUniqRow]
  Clé            : id_formation
  Flux rejeté    → [tLogRow] "DOUBLON Excel éliminé : {id_formation}"
        ↓
[tMap - Corrections NULL]
        ↓
[tPostgresqlOutput - dim_formation]
  Action : INSERT
```

### Schéma tFileInputExcel

| Colonne Excel | Nom Talend | Type Talend | Longueur |
|---|---|---|---|
| ID Formation | id_formation | String | 10 |
| Matricule | matricule | String | 10 |
| Nom | nom | String | 50 |
| Prénom | prenom | String | 50 |
| Service | service | String | 50 |
| Intitulé | intitule | String | 100 |
| Durée (h) | duree_heures | Integer | - |
| Année | annee | Integer | - |
| Coût (DT) | cout | Double | - |
| Statut | statut | String | 20 |

### Expressions tMap

```java
out.id_formation = row.id_formation
out.matricule    = row.matricule
out.intitule     = row.intitule
out.duree_heures = (row.duree_heures == null) ? 0    : row.duree_heures
out.annee        = row.annee
out.cout         = (row.cout == null)         ? 0.0  : row.cout
out.statut       = (row.statut == null || row.statut.isEmpty())
                   ? "Inconnu" : row.statut
```

### Résultat attendu

| Table | Entrée | Sortie |
|---|---|---|
| dim_formation | 266 lignes | **238 lignes** |

---

## J5 - Job_Load_FAIT_RH

**⚠️ À exécuter OBLIGATOIREMENT EN DERNIER - J1 à J4 doivent être terminés**  
**Sources :** Toutes les tables DIM de `rh_dw` + table `salaires` de `rh_entreprise`  
**Cible :** `fait_rh`

### Logique métier

La table de faits contient **1 ligne par employé**. Pour chaque employé dans `dim_employe`, on :
1. Résout le `sk_service` via lookup sur `dim_service`
2. Agrège les absences (`SUM duree_jours`, `COUNT absences`) par matricule
3. Agrège les formations (`COUNT formations`) par matricule
4. Récupère le salaire moyen mensuel depuis la table `salaires`
5. Calcule le salaire annuel = mensuel × 12

### Flux Talend

```
[tPostgresqlInput - dim_employe]    ← flux principal
        ↓
[tMap central]
  ├── lookup dim_service            → sk_service  (par nom_service)
  ├── lookup agg_absences           → nb_jours_absence, nb_absences
  └── lookup agg_formations         → nb_formations
        ↓
[tPostgresqlOutput - fait_rh] INSERT

─── Sous-flux préalables (tHashOutput) ─────────────────────────────────

[tPostgresqlInput - rh_entreprise.salaires]
  SELECT matricule,
         AVG(salaire_mensuel) AS salaire_moy,
         SUM(prime)           AS prime_totale
  FROM salaires
  GROUP BY matricule
  → [tHashOutput "hash_salaires"]

[tPostgresqlInput - dim_absence]
  SELECT matricule,
         SUM(duree_jours) AS nb_jours_absence,
         COUNT(*)         AS nb_absences
  FROM dim_absence
  GROUP BY matricule
  → [tHashOutput "hash_absences"]

[tPostgresqlInput - dim_formation]
  SELECT matricule,
         COUNT(*) AS nb_formations
  FROM dim_formation
  GROUP BY matricule
  → [tHashOutput "hash_formations"]

[tPostgresqlInput - dim_service]
  SELECT sk_service, nom_service
  FROM dim_service
  → [tHashOutput "hash_service"]
```

### Requêtes SQL des sous-flux

```sql
-- Agrégation salaires (depuis rh_entreprise)
SELECT
    matricule,
    AVG(salaire_mensuel) AS salaire_mensuel_moy,
    SUM(prime)           AS prime_annuelle
FROM salaires
WHERE salaire_mensuel IS NOT NULL
GROUP BY matricule;

-- Agrégation absences (depuis rh_dw)
SELECT
    matricule,
    SUM(duree_jours) AS nb_jours_absence,
    COUNT(*)         AS nb_absences_total
FROM dim_absence
GROUP BY matricule;

-- Agrégation formations (depuis rh_dw)
SELECT
    matricule,
    COUNT(*) AS nb_formations
FROM dim_formation
GROUP BY matricule;
```

### Expressions tMap central

```java
// ── Clés surrogate ──────────────────────────────────────────
out.sk_employe = emp.sk_employe
out.sk_service = svc.sk_service      // lookup : emp.service = svc.nom_service
out.sk_temps   = null                // optionnel : sk de la date du jour

// ── Mesures salariales ──────────────────────────────────────
out.salaire_mensuel = (sal.salaire_mensuel_moy == null)
                      ? emp.salaire_mensuel : sal.salaire_mensuel_moy
out.salaire_annuel  = (out.salaire_mensuel == null)
                      ? 0.0 : out.salaire_mensuel * 12
out.prime           = (sal.prime_annuelle == null) ? 0.0 : sal.prime_annuelle

// ── Mesures absences ────────────────────────────────────────
out.nb_jours_absence = (abs.nb_jours_absence == null)
                       ? 0 : abs.nb_jours_absence

// ── Mesures formations ──────────────────────────────────────
out.nb_formations    = (frm.nb_formations == null)
                       ? 0 : frm.nb_formations
```

### Résultat attendu

| Table | Sortie | Détail |
|---|---|---|
| fait_rh | **100 lignes** | 1 ligne par employé unique |

---

## 📊 Tableau Récapitulatif Global

| Job | Source | Lignes entrée | Doublons éliminés | NULLs corrigés | Lignes chargées |
|---|---|---|---|---|---|
| J1 → dim_employe | SQL (employes) | 112 | **12** (casse) | 30 valeurs | **100** |
| J1 → dim_service | SQL (employes) | 100 | dédoublonné | - | **10** |
| J2 → dim_temps | générée | - | - | - | **1 096** |
| J3 → dim_absence | CSV (582) | 582 | **62** exacts | 157 valeurs | **520** |
| J3 → dim_motif_absence | CSV motifs | - | dédoublonné | - | **7** |
| J4 → dim_formation | Excel (266) | 266 | **28** exacts | 85 valeurs | **238** |
| J5 → fait_rh | DIM/* + salaires | 100 emp | - | agrégations | **100** |

---

## 🔑 Correspondances MySQL → PostgreSQL

| Concept | MySQL | PostgreSQL |
|---|---|---|
| Auto-incrément | `AUTO_INCREMENT` | `SERIAL` |
| Booléen/flag | `TINYINT(1)` | `SMALLINT` |
| Décimal | `DECIMAL(10,2)` | `NUMERIC(10,2)` |
| Ignorer doublons | `INSERT IGNORE` | `INSERT … ON CONFLICT DO NOTHING` |
| Connexion Talend | `tMysqlConnection` | `tPostgresqlConnection` |
| Lecture Talend | `tMysqlInput` | `tPostgresqlInput` |
| Écriture Talend | `tMysqlOutput` | `tPostgresqlOutput` |
| Requête DDL | `tMysqlRow` | `tPostgresqlRow` |
| Port | `3306` | **`5432`** |
| Sélection base | `USE rh_dw;` | connexion directe à la base |

---

## 🗂️ Structure du Projet Talend Open Studio

```
Projet_SI_Decisionnels/       ← nom du projet dans Talend
│
├── Jobs/
│   ├── J0_InitialLoad        0.1
│   ├── J1_Load_DIM_EMPLOYE   0.1
│   ├── J2_Load_DIM_TEMPS     0.1
│   ├── J3_Load_DIM_ABSENCE   0.1
│   ├── J4_Load_DIM_FORMATION 0.1
│   └── J5_Load_FAIT_RH       0.1
│
├── Métadonnées/
│   ├── Connexions DB/
│   │   ├── SRC_rh_entreprise  (PostgreSQL · localhost:5432)
│   │   └── TGT_rh_dw          (PostgreSQL · localhost:5432)
│   └── Fichiers délimités/
│       └── schema_absences_csv
│
└── Contextes/
    └── Default
        ├── pg_host     = localhost
        ├── pg_port     = 5432
        ├── src_db      = rh_entreprise
        ├── tgt_db      = rh_dw
        ├── pg_user     = postgres
        ├── pg_password = ****
        ├── csv_path    = C:/data/raw/absences_presences.csv
        └── xlsx_path   = C:/data/raw/formations.xlsx
```

> 💡 **Conseil :** Utiliser un **Context Group** Talend pour centraliser tous les paramètres.
> Cela permet de changer host/port/password en un seul endroit - indispensable
> si tu travailles entre ta machine à la maison et les postes de l'école.