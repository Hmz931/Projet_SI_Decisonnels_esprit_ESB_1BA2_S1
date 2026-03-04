Les 3 fichiers sont régénérés avec des anomalies réalistes. Voici le récapitulatif exact de ce que Talend devra traiter :

🗄️ SQL — employes_salaires.sql

- 12 doublons sur la table employes (même matricule, nom en casse différente ex: BEN ALI vs ben ali)
- NULLs : ville (~10%), email (~8%), téléphone (~10%), poste (~7%)
- 66 salaires NULL sur 1 200 lignes (~6%)

📄 CSV — absences_presences.csv

- 66 lignes dupliquées sur 624 (~12%)
- NULLs : motif (46 cas), durée (41 cas), justifié (28 cas)

📊 Excel — formations.xlsx

- 30 lignes dupliquées sur 280 (~12%)
- NULLs : coût (25 cas), statut (19 cas), durée (13 cas)