# ⚙️ Documentation Détaillée des Jobs Talend
## Projet SI Décisionnels — Entrepôt de Données RH

> **Projet** : Création d'un entrepôt de données pour la gestion du personnel  
> **Auteurs** : Hamza Bouguerra · Fares Messedi — ESPRIT 1BA2  
> **Enseignant** : Sofien Boutaib | **Deadline** : 14 Mars 2026  
> **Outil** : Talend Open Studio for Data Integration  
> **Base source** : MySQL `rh_entreprise` | **Base cible** : MySQL `rh_dw`

---

## 📁 Analyse des sources réelles

### absences_presences.csv
| Indicateur | Valeur |
|---|---|
| Total lignes | **582** |
| Doublons exacts | **62** (10.7%) |
| NULL motif | **54** (9.3%) |
| NULL duree_jours | **50** (8.6%) |
| NULL justifie | **53** (9.1%) |
| Colonne remarque | **582 vides** → à ignorer |
| Matricules distincts | 86 employés concernés |
| Plage de dates | 2022-01-02 → 2024-12-27 |
| Motifs existants | Accident de travail · Congé annuel · Congé sans solde · Formation · Maladie · Maternité/Paternité |

### employes_salaires.sql
| Indicateur | Valeur |
|---|---|
| INSERT employes | **112** (dont 12 doublons) |
| INSERT salaires | **1200** (100 emp × 12 mois) |
| NULL poste | **14** |
| NULL date_embauche | **7** |
| NULL ville | **9** |
| Salaire min/max/moy | 815 DT / 4445 DT / 2592 DT |
| Primes possibles | 0 · 100 · 150 · 200 · 300 DT |
| Services distincts | 10 |

### formations.xlsx
| Indicateur | Valeur |
|---|---|
| Total lignes | **266** |
| Doublons exacts | **28** (10.5%) |
| NULL Coût (DT) | **31** (11.7%) |
| NULL Statut | **30** (11.3%) |
| NULL Durée (h) | **24** (9.0%) |
| Formations distinctes | 16 intitulés |
| Années couvertes | 2022 · 2023 · 2024 |
| Coût min/max/moy | 500 / 2000 / 1176 DT |
| Durées possibles (h) | 8 · 16 · 20 · 24 · 32 · 40 |
| Statuts | Complétée (77) · En cours (89) · Planifiée (70) · NULL (30) |

---

## ⚡ Ordre d'exécution

```
J0_InitialLoad
      ↓
J1_Load_DIM_EMPLOYE ──┐
J2_Load_DIM_TEMPS   ──┘  (parallèle possible)
      ↓
J3_Load_DIM_ABSENCE
J4_Load_DIM_FORMATION
      ↓
J5_Load_FAIT_RH   ← EN DERNIER obligatoirement
```

---

## J0 — Job_InitialLoad

**Rôle** : Initialiser la base cible `rh_dw`, créer toutes les tables, vérifier l'accès aux 3 sources.

### Composants et configuration

| Composant | Paramètres |
|---|---|
| `tMysqlConnection` | host=`localhost` · port=`3306` · db=`rh_dw` · user=`root` |
| `tFileCheck` | Chemins vers les 3 fichiers sources |
| `tMysqlRow` | Exécute le DDL ci-dessous |
| `tLogRow` | Affiche `[OK] Base rh_dw initialisée — {timestamp}` |
| `tDie` | Déclenché si un fichier source est introuvable |

### Flux

```
[tMysqlConnection] ──→ [tFileCheck] ──→ [tMysqlRow : DDL] ──→ [tLogRow]
                             ↓ (fichier manquant)
                        [tDie "Fichier source introuvable"]
```

### Script DDL — à coller dans `tMysqlRow`

```sql
CREATE DATABASE IF NOT EXISTS rh_dw;
USE rh_dw;

CREATE TABLE IF NOT EXISTS DIM_EMPLOYE (
    sk_employe    INT AUTO_INCREMENT PRIMARY KEY,
    matricule     VARCHAR(10)   NOT NULL UNIQUE,
    nom           VARCHAR(50),
    prenom        VARCHAR(50),
    service       VARCHAR(50),
    poste         VARCHAR(100),
    date_embauche DATE,
    ville         VARCHAR(50),
    email         VARCHAR(100),
    telephone     VARCHAR(20)
);

CREATE TABLE IF NOT EXISTS DIM_SERVICE (
    sk_service  INT AUTO_INCREMENT PRIMARY KEY,
    nom_service VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE IF NOT EXISTS DIM_TEMPS (
    sk_temps      INT AUTO_INCREMENT PRIMARY KEY,
    date_complete DATE        NOT NULL UNIQUE,
    jour          INT,
    mois          INT,
    annee         INT,
    trimestre     INT,
    semaine       INT,
    nom_mois      VARCHAR(20),
    nom_jour      VARCHAR(20),
    est_weekend   TINYINT(1)  DEFAULT 0
);

CREATE TABLE IF NOT EXISTS DIM_ABSENCE (
    sk_absence   INT AUTO_INCREMENT PRIMARY KEY,
    id_absence   VARCHAR(10),
    matricule    VARCHAR(10),
    date_absence DATE,
    motif        VARCHAR(50),
    duree_jours  INT         DEFAULT 0,
    justifie     VARCHAR(15)
);

CREATE TABLE IF NOT EXISTS DIM_MOTIF_ABSENCE (
    sk_motif  INT AUTO_INCREMENT PRIMARY KEY,
    motif     VARCHAR(50) NOT NULL UNIQUE,
    categorie VARCHAR(30)
);

CREATE TABLE IF NOT EXISTS DIM_FORMATION (
    sk_formation INT AUTO_INCREMENT PRIMARY KEY,
    id_formation VARCHAR(10),
    matricule    VARCHAR(10),
    intitule     VARCHAR(100),
    duree_heures INT          DEFAULT 0,
    annee        INT,
    cout         DECIMAL(10,2) DEFAULT 0.00,
    statut       VARCHAR(20)
);

CREATE TABLE IF NOT EXISTS FAIT_RH (
    id_fait          INT AUTO_INCREMENT PRIMARY KEY,
    sk_employe       INT           NOT NULL,
    sk_service       INT           NOT NULL,
    sk_temps         INT,
    sk_absence       INT,
    sk_formation     INT,
    sk_motif         INT,
    salaire_mensuel  DECIMAL(10,2),
    salaire_annuel   DECIMAL(10,2),
    prime            DECIMAL(10,2) DEFAULT 0,
    nb_jours_absence INT           DEFAULT 0,
    nb_formations    INT           DEFAULT 0,
    FOREIGN KEY (sk_employe)   REFERENCES DIM_EMPLOYE(sk_employe),
    FOREIGN KEY (sk_service)   REFERENCES DIM_SERVICE(sk_service),
    FOREIGN KEY (sk_temps)     REFERENCES DIM_TEMPS(sk_temps),
    FOREIGN KEY (sk_absence)   REFERENCES DIM_ABSENCE(sk_absence),
    FOREIGN KEY (sk_formation) REFERENCES DIM_FORMATION(sk_formation),
    FOREIGN KEY (sk_motif)     REFERENCES DIM_MOTIF_ABSENCE(sk_motif)
);
```

---

## J1 — Job_Load_DIM_EMPLOYE

**Source** : MySQL `rh_entreprise` → table `employes`  
**Cibles** : `DIM_EMPLOYE` + `DIM_SERVICE`

### Anomalies réelles identifiées

```
112 INSERT bruts → 100 employés uniques  (12 doublons)

Doublons (même matricule, casse différente) :
  EMP002 → 'Trabelsi'  vs 'trabelsi'
  EMP010 → 'Belhaj'    vs 'belhaj'
  EMP013 → 'Rekik'     vs 'rekik'
  EMP019 → 'Jallouli'  vs 'JALLOULI'
  EMP020 → 'Haddad'    vs 'HADDAD'
  ... (12 au total)

NULLs par colonne :
  poste         → 14 cas
  ville         →  9 cas
  date_embauche →  7 cas
```

### Flux Talend

```
[tMysqlConnection — rh_entreprise]
         ↓
[tMysqlInput]
  Query : SELECT * FROM employes
         ↓
[tMap — Normalisation casse]
  nom    → StringHandling.UPCASE(TRIM(nom))
  prenom → StringHandling.UPCASE_FIRST_LOWER(TRIM(prenom))
         ↓
[tUniqRow]
  Clé   : matricule
  → rejeté → [tLogRow] "DOUBLON éliminé : {matricule}"
         ↓
[tMap — Corrections NULL + split service]
  Flux 1 ──→ [tMysqlOutput] DIM_EMPLOYE  (INSERT)
  Flux 2 ──→ [tMysqlOutput] DIM_SERVICE  (INSERT IGNORE)
```

### Expressions tMap — Flux 1 (DIM_EMPLOYE)

```java
out1.matricule     = row.matricule
out1.nom           = row.nom
out1.prenom        = row.prenom
out1.service       = row.service
out1.poste         = (row.poste == null || row.poste.isEmpty())
                     ? "Non renseigné" : row.poste
out1.date_embauche = row.date_embauche
out1.ville         = (row.ville == null || row.ville.isEmpty())
                     ? "Non renseignée" : row.ville
out1.email         = (row.email == null || row.email.isEmpty())
                     ? "non-renseigne@entreprise.tn" : row.email
out1.telephone     = (row.telephone == null || row.telephone.isEmpty())
                     ? "Non renseigné" : row.telephone
```

### Expressions tMap — Flux 2 (DIM_SERVICE)

```java
out2.nom_service = row.service
// INSERT IGNORE sur UNIQUE KEY → 10 services uniques automatiquement
```

### Résultat attendu

| Table | Entrée | Sortie |
|---|---|---|
| DIM_EMPLOYE | 112 inserts | **100 lignes** |
| DIM_SERVICE | 100 lignes | **10 lignes** |

---

## J2 — Job_Load_DIM_TEMPS

**Source** : Générée programmatiquement — aucun fichier requis  
**Cible** : `DIM_TEMPS` | **Période** : 2022-01-01 → 2024-12-31

### Flux Talend

```
[tFixedFlowInput]
  date_debut = 2022-01-01
  date_fin   = 2024-12-31
         ↓
[tFlowToIterate] → une ligne par jour
         ↓
[tMap — Calcul des attributs temporels]
         ↓
[tMysqlOutput — DIM_TEMPS]
  Action : INSERT IGNORE
```

### Expressions tMap

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

### Résultat attendu
- **1 096 lignes** dans DIM_TEMPS (3 ans de calendrier)

> 💡 Ce job est indépendant — il peut tourner en parallèle de J1.

---

## J3 — Job_Load_DIM_ABSENCE

**Source** : `data/raw/absences_presences.csv` (séparateur `;`)  
**Cibles** : `DIM_ABSENCE` + `DIM_MOTIF_ABSENCE`

### Anomalies réelles identifiées

```
582 lignes brutes → 520 lignes uniques

Doublons exacts  : 62 lignes  (10.7%)
NULL motif       : 54 cas     (9.3%)  → "Non précisé"
NULL duree_jours : 50 cas     (8.6%)  → 0
NULL justifie    : 53 cas     (9.1%)  → "Non renseigné"
remarque         : 582 vides          → ne pas charger

Motifs valides présents dans le fichier :
  Accident de travail · Congé annuel · Congé sans solde
  Formation · Maladie · Maternité/Paternité
```

### Flux Talend

```
[tFileInputDelimited]
  Fichier    : absences_presences.csv
  Séparateur : ;
  Encodage   : UTF-8
  Header     : true
         ↓
[tUniqRow]
  Clé : id_absence + matricule + date_absence
  → rejeté → [tLogRow] "DOUBLON CSV : {id_absence}"
         ↓
[tReplaceList]
  motif    vide → "Non précisé"
  justifie vide → "Non renseigné"
         ↓
[tMap]
  Flux 1 ──→ [tMysqlOutput] DIM_ABSENCE
  Flux 2 ──→ [tUniqRow motif] ──→ [tMysqlOutput] DIM_MOTIF_ABSENCE
```

### Schéma tFileInputDelimited

| Colonne CSV | Nom Talend | Type Talend | Action |
|---|---|---|---|
| id_absence | id_absence | String | charger |
| matricule | matricule | String | charger |
| date_absence | date_absence | Date `yyyy-MM-dd` | charger |
| motif | motif | String | corriger |
| duree_jours | duree_jours | Integer | corriger |
| justifie | justifie | String | corriger |
| remarque | remarque | String | **ne pas mapper** |

### Expressions tMap — Flux 1 (DIM_ABSENCE)

```java
out1.id_absence   = row.id_absence
out1.matricule    = row.matricule
out1.date_absence = row.date_absence
out1.motif        = row.motif      // déjà corrigé par tReplaceList
out1.duree_jours  = (row.duree_jours == null) ? 0 : row.duree_jours
out1.justifie     = row.justifie   // déjà corrigé par tReplaceList
```

### Expressions tMap — Flux 2 (DIM_MOTIF_ABSENCE)

```java
String m = row.motif;
out2.motif     = m;
out2.categorie = m.equals("Maladie") || m.equals("Accident de travail") ? "Médical"
               : m.equals("Congé annuel") || m.equals("Congé sans solde") ? "Congé"
               : m.equals("Maternité/Paternité") ? "Familial"
               : m.equals("Formation") ? "Professionnel" : "Autre";
```

### Résultat attendu

| Table | Entrée | Sortie |
|---|---|---|
| DIM_ABSENCE | 582 lignes | **520 lignes** |
| DIM_MOTIF_ABSENCE | motifs uniques | **7 lignes** |

---

## J4 — Job_Load_DIM_FORMATION

**Source** : `data/raw/formations.xlsx` — feuille `Formations`  
**Cible** : `DIM_FORMATION`

### Anomalies réelles identifiées

```
266 lignes brutes → 238 lignes uniques

Doublons exacts  : 28 lignes  (10.5%)
NULL Coût (DT)   : 31 cas     (11.7%) → 0.0
NULL Statut      : 30 cas     (11.3%) → "Inconnu"
NULL Durée (h)   : 24 cas     (9.0%)  → 0

16 formations distinctes dans le fichier :
  Analyse de données · Communication professionnelle · Cybersécurité
  Droit du travail · ERP SAP · Excel avancé · Leadership
  Management de projet · Marketing digital · Négociation commerciale
  Power BI · Python pour la data · Qualité ISO 9001
  SQL et bases de données · Sécurité au travail · Talend ETL

Coûts (DT)   : 500 · 800 · 1000 · 1200 · 1500 · 2000
Durées (h)   : 8 · 16 · 20 · 24 · 32 · 40
Statuts      : Complétée (77) · En cours (89) · Planifiée (70) · NULL (30)
```

### Flux Talend

```
[tFileInputExcel]
  Fichier        : formations.xlsx
  Feuille        : Formations
  Première ligne : 2  (ligne 1 = header)
         ↓
[tUniqRow]
  Clé : ID Formation
  → rejeté → [tLogRow] "DOUBLON Excel : {id_formation}"
         ↓
[tMap — Corrections NULL]
         ↓
[tMysqlOutput — DIM_FORMATION]
  Action : INSERT
```

### Schéma tFileInputExcel

| Colonne Excel | Nom Talend | Type Talend |
|---|---|---|
| ID Formation | id_formation | String |
| Matricule | matricule | String |
| Nom | nom | String |
| Prénom | prenom | String |
| Service | service | String |
| Intitulé | intitule | String |
| Durée (h) | duree_heures | Integer |
| Année | annee | Integer |
| Coût (DT) | cout | Double |
| Statut | statut | String |

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
| DIM_FORMATION | 266 lignes | **238 lignes** |

---

## J5 — Job_Load_FAIT_RH

**⚠️ Exécuter EN DERNIER — J1 à J4 obligatoirement terminés**  
**Sources** : toutes les DIM de `rh_dw`  
**Cible** : `FAIT_RH`

### Flux Talend

```
[tMysqlInput] SELECT * FROM DIM_EMPLOYE   ← flux principal
         ↓
[tMap central]
  ├─ lookup DIM_SERVICE   (par nom_service) → sk_service
  ├─ lookup agg_absences  (par matricule)   → nb_jours_absence
  └─ lookup agg_formations(par matricule)   → nb_formations
         ↓
[tMysqlOutput — FAIT_RH] INSERT

─── Sous-flux préalables ──────────────────────────────────────
[tMysqlInput agg_absences]
  SELECT matricule,
         SUM(duree_jours) AS nb_jours_absence,
         COUNT(*)         AS nb_absences_total
  FROM DIM_ABSENCE GROUP BY matricule
  → [tHashOutput "agg_absences"]

[tMysqlInput agg_formations]
  SELECT matricule, COUNT(*) AS nb_formations
  FROM DIM_FORMATION GROUP BY matricule
  → [tHashOutput "agg_formations"]
```

### Expressions tMap central

```java
// Clés surrogate
out.sk_employe = emp.sk_employe
out.sk_service = svc.sk_service   // lookup emp.service = svc.nom_service

// Mesures
out.salaire_mensuel  = emp.salaire_mensuel
out.salaire_annuel   = (emp.salaire_mensuel == null)
                       ? 0.0 : emp.salaire_mensuel * 12
out.prime            = 0.0

// Agrégats
out.nb_jours_absence = (agg_abs.nb_jours_absence  == null)
                       ? 0 : agg_abs.nb_jours_absence
out.nb_formations    = (agg_form.nb_formations == null)
                       ? 0 : agg_form.nb_formations
```

### Résultat attendu

| Table | Sortie |
|---|---|
| FAIT_RH | **100 lignes** (1 par employé) |

---

## 📊 Tableau récapitulatif global

| Job | Source | Lignes entrée | Doublons supprimés | NULLs corrigés | Lignes chargées |
|---|---|---|---|---|---|
| J1 → DIM_EMPLOYE | SQL | 112 | **12** | 30 valeurs | **100** |
| J1 → DIM_SERVICE | SQL | 100 | dédoublonné | — | **10** |
| J2 → DIM_TEMPS | générée | — | — | — | **1 096** |
| J3 → DIM_ABSENCE | CSV | 582 | **62** | 157 valeurs | **520** |
| J3 → DIM_MOTIF_ABSENCE | CSV | 520 | dédoublonné | — | **7** |
| J4 → DIM_FORMATION | Excel | 266 | **28** | 85 valeurs | **238** |
| J5 → FAIT_RH | DIM/* | 100 emp | — | agrégations | **100** |

---

## 🗂️ Structure du projet Talend Open Studio

```
Projet_SI_Decisionnels/
│
├── Job Designs/
│   ├── J0_InitialLoad
│   ├── J1_Load_DIM_EMPLOYE
│   ├── J2_Load_DIM_TEMPS
│   ├── J3_Load_DIM_ABSENCE
│   ├── J4_Load_DIM_FORMATION
│   └── J5_Load_FAIT_RH
│
├── Metadata/
│   ├── Db Connections/
│   │   ├── SRC_rh_entreprise   (MySQL source · port 3306)
│   │   └── TGT_rh_dw           (MySQL cible  · port 3306)
│   └── File Delimited/
│       └── schema_absences_csv
│
└── Contexts/
    └── Default
        ├── src_host     = localhost
        ├── src_port     = 3306
        ├── src_db       = rh_entreprise
        ├── tgt_db       = rh_dw
        ├── db_user      = root
        ├── db_password  = ****
        ├── csv_path     = C:/data/raw/absences_presences.csv
        └── xlsx_path    = C:/data/raw/formations.xlsx
```

> 💡 Déclarer les chemins dans un **Context Group** permet de changer les sources
> sans modifier chaque job — utile si tu travailles entre la maison et l'école.
