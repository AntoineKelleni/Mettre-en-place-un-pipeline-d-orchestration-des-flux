<p align="center">
  <img src="logo_OCR.jpg" alt="Logo Academy" width="200">
</p>

# Pipeline d'orchestration de flux : Traitement & Analyse de Données – Kestra + DuckDB + Pandas
## Introduction

Ce projet met en place un pipeline data complet, automatisé et industrialisé avec Kestra, permettant :

La conversion de fichiers Excel en CSV

Le chargement et le nettoyage des données

La jointure entre systèmes ERP / Web / Table de correspondance

Le calcul automatisé du chiffre d’affaires

La détection des vins premium via un z-score

La validation systématique des résultats (tests QC)

L’export des livrables finaux (Excel & CSV)

Ce workflow reproduit les pratiques d’un Data Engineer.

## Topologie du pipeline Kestra UI
<p align="center"> <img src="KELLENI_Antoine_2_workflow_kestra_topologie_112025.png" alt="Topologie Kestra UI" width="500"> </p>

Le pipeline est orchestré par Kestra et s’exécute automatiquement chaque 15 du mois à 9h via un déclencheur CRON.

### Il est constitué de 3 grands blocs :
```
Ingestion & préparation : conversion des fichiers Excel → CSV

Chargement, nettoyage & analyse : DuckDB

Exports métiers : Excel CA & fichiers Premium/Ordinaire
```

## Technologies utilisées

```Kestra``` – Orchestration du workflow

```DuckDB``` – Moteur analytique embarqué

```Pandas``` – Nettoyage & export

```Python``` – Scripts de conversion & calculs

```Docker``` – Isolation des environnements d’exécution

```OpenPyXL``` – Manipulation Excel



### 1️⃣ Conversion Excel → CSV
3 tasks :
```convert_erp```
```convert_web``` 
```convert_liaison```


Objectifs :
```
- Charger les fichiers Excel (ERP, Web, Liaison)

- Écrire les équivalents CSV

- Loguer le nombre de lignes
```

### 2️⃣ Chargement des données dans DuckDB
1 task : 
```load_duckdb```
```
- Création des tables erp, web, liaison

- Lecture via read_csv_auto

- Log automatique des lignes & colonnes
```

### 3️⃣ Nettoyage & dédoublonnage
1 task: ```clean_tables```

Méthodologie :
```
ERP → unique par product_id

Web → unique par sku

Liaison → unique par product_id

Suppression des valeurs nulles

Conservation de la 1ère occurrence (ROW_NUMBER)
```


Contrôles :

```erp_clean``` = 825 lignes

```web_clean``` = 714 lignes

```liaison_clean``` = 825 lignes


 ### 3️⃣ 3bis️ Test de conformité
task :```test_clean_counts```
```
SELECT CASE
  WHEN (SELECT COUNT(*) FROM erp_clean) = 825
  AND  (SELECT COUNT(*) FROM web_clean) = 714
  AND  (SELECT COUNT(*) FROM liaison_clean) = 825
  THEN 1 ELSE 1/0 END;
  ```

### 4️⃣ Jointure des systèmes
task : ```join_tables```

```
Jointure ERP → LIAISON → WEB avec cast sécurisé :

ON TRY_CAST(w.sku AS BIGINT) = TRY_CAST(l.id_web AS BIGINT)


Résultat attendu : 714 lignes fusionnées.
```

### 4️⃣ 4bis Test
task : ```test_join```

```
Vérification nb lignes

Aucun champ null dans la fusion
```

### 5️⃣ Calcul du chiffre d’affaires
task : ```compute_ca```

Formule :
```
price * total_sales AS chiffre_affaires
```

La table finale ca contient :
```
product_id

product_name

price

total_sales

chiffre_affaires
```
``` CA total attendu : 65 652,60 €```

### 5️⃣ 5bis️ Test
task : ```test_ca```
```
ROUND((SELECT SUM(chiffre_affaires) FROM ca), 2) = 65652.60
```
### 6️⃣ Exports métiers (Execution parallèle)
exports → Parallel block

Contient :

➤ export_ca

Recalcul Pandas

Validation du CA

Export Excel : rapport_chiffre_affaires.xlsx

➤ premium_detection

Calcul du z-score

Premium = z-score > 2

Export :

vins_premium.csv

vins_ordinaires.csv

Test intégré :
 30 vins premium

## Structure du repository
```
project/
│── data/
│   ├── Fichier_erp.xlsx
│   ├── Fichier_web.xlsx
│   ├── fichier_liaison.xlsx
│
│── outputs/
│   ├── rapport_chiffre_affaires.xlsx
│   ├── vins_premium.csv
│   ├── vins_ordinaires.csv
│
│── .kestra/
│── workflow.yaml
│── README.md
│── docker-compose.yml
```
## Installation & Exécution
### 1️⃣ Lancer Kestra avec Docker
```
docker-compose up -d
```

Kestra accessible sur :

```URL :``` "http://localhost:8080"

### 2️⃣ Déclarer le workflow

Depuis l’interface Kestra → "Flows" → Import → workflow.yaml

### 3️⃣ Exécuter manuellement

 Bouton ```Run``` dans l’UI
ou :

```
kestra flow start bottleneck.bottleneck_workflow
```

### 4️⃣ Résultats générés

Dans l’onglet Outputs du flow :

```rapport_chiffre_affaires.xlsx```

```vins_premium.csv```

```vins_ordinaires.csv```

### Résultats finaux
``` 
Indicateur	               Valeur

Lignes ERP nettoyées	    825
Lignes Web nettoyées	    714
Lignes Liaison nettoyées	825
Lignes fusionnées	        714
CA total	                65 652,60 €
Vins premium détectés	    30
```
Toutes les valeurs ont été validées automatiquement via les steps test_*.



## Conclusion

Ce pipeline Kestra offre une solution complète, robuste et automatisée pour consolider, contrôler et analyser les données produits.

Il adopte une architecture professionnelle et garantit la fiabilité des résultats via des tests intégrés.
