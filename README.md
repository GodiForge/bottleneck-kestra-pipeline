# 🍷 Automatisation du Pipeline Data — BottleNeck

> Mise en place d'un flux automatisé mensuel avec **Kestra**, **DuckDB**, **Python** et **Docker** pour calculer le chiffre d'affaires et segmenter les vins millésimes.

## 🎯 Contexte et objectif

Chez **BottleNeck** (marchand de vin), les données sont réparties dans **trois fichiers Excel** issus de systèmes distincts :

1. **`erp.xlsx`** : le catalogue interne avec les prix et les stocks ;
2. **`web.xlsx`** : les ventes réalisées sur la boutique en ligne ;
3. **`liaison.xlsx`** : la table de correspondance reliant les identifiants de l'ERP (`product_id`) à ceux du site web (`id_web`).

Jusqu'ici, **Stéphane** (Data Analyst) nettoyait et croisait ces trois fichiers manuellement chaque mois, ce qui prenait du temps et exposait les exports à des risques d'erreurs de saisie.

**L'objectif de ce projet** est d'automatiser l'ensemble du traitement afin que chaque **15 du mois à 9h00**, le pipeline s'exécute automatiquement et génère **3 fichiers prêts à l'emploi** :

* 📊 **`rapport_ca_bottleneck.xlsx`** : le rapport financier (chiffre d'affaires par bouteille et chiffre d'affaires global) pour le pilotage ;
* 🥂 **`vins_premium.csv`** : la liste des **vins millésimes / prestige** ($z > 2$) pour **Laure** (clientèle B2B) ;
* 🍷 **`vins_ordinaires.csv`** : la liste des **vins ordinaires** ($z \le 2$) pour **Maria** (clientèle B2C).

## 🛠️ Architecture et outils

Chaque composant du pipeline répond à un rôle précis :

* **Kestra (Orchestrateur) :** Pilote l'enchaînement des tâches définies en YAML, gère le parallélisme, la planification mensuelle et les reprises sur erreur. Il ne porte pas les calculs lourds mais coordonne les moteurs d'exécution.
* **DuckDB (Moteur SQL analytique) :** Exécute les requêtes SQL directement en mémoire sur les fichiers Excel et Parquet intermédiaires en moins de 2 secondes, sans nécessiter de serveur de base de données dédié.
* **Python (`pandas` & `openpyxl`) :** Réalise le calcul statistique du *Z-Score* et génère le classeur Excel multi-onglets ainsi que les fichiers CSV finaux.
* **Docker :** Permet d'exécuter Kestra dans un conteneur isolé et reproductible sur n'importe quel poste.

## ⚙️ Fonctionnement du pipeline

Le workflow final (`flows/03_bottleneck_avec_tests.yaml`) enchaîne **4 étapes principales** en **12 secondes** environ :

1. **Contrôle d'ingestion :** Vérifie que `erp.xlsx`, `liaison.xlsx` et `web.xlsx` sont présents et lisibles.
2. **Nettoyage en parallèle (DuckDB) :** Les 3 sources sont traitées simultanément :
   * Suppression des valeurs manquantes et dédoublonnage sur les clés primaires ;
   * Filtrage des fiches produits dans `web.xlsx` (`post_type = 'product'`, écartant les 85 lignes sans référence et les lignes correspondant aux images) ;
   * Lecture des identifiants au format texte (`VARCHAR`) afin de conserver les références alphanumériques avec tiret (comme `13127-1`).
3. **Réconciliation et calcul du CA (DuckDB) :** Jointure triple des sources propres pour obtenir les **714 bouteilles vendues en ligne** et calcul du chiffre d'affaires par produit ($\text{prix} \times \text{quantité}$).
4. **Génération des livrables en parallèle (Python) :**
   * **Branche 1 (Excel) :** Création de `rapport_ca_bottleneck.xlsx` (onglet `CA_par_produit` + onglet `Synthese_CA_Global`).
   * **Branche 2 (CSV) :** Calcul du **Z-Score** sur le prix unitaire ($z = \frac{\text{prix} - \mu}{\sigma}$). Les vins avec $z > 2$ (au-dessus de 88,11 €) sont exportés dans `vins_premium.csv`, les autres dans `vins_ordinaires.csv`.

### 🛡️ Contrôles qualité et gestion des pannes

**5 tests bloquants** sécurisent l'exécution à chaque étape :

1. **Unicité des clés :** `0` doublon sur `product_id` et `sku` (`825` dans ERP, `825` dans Liaison, `714` dans Web).
2. **Complétude :** `0` valeur manquante sur les clés, les prix et les quantités.
3. **Cohérence de la jointure :** Exactement **714 produits** réconciliés sans perte.
4. **Cohérence du chiffre d'affaires :** Aucun montant négatif et total exact validé (**70 568,60 €**).
5. **Cohérence de la segmentation Z-Score :** Exactement **30 vins millésimes** et **684 vins ordinaires** ($30 + 684 = 714$).

> **Résilience en cas d'incident :**
> * Chaque tâche DuckDB intègre une politique de reprise automatique (**3 tentatives** espacées de 10 secondes via `retry`).
> * En cas de fichier manquant ou d'échec d'un test qualité, **le pipeline s'arrête immédiatement (`FAILED`)** avant de générer des fichiers corrompus, journalise une alerte dans les logs et permet une reprise en un clic via le bouton **Replay**.

## 📊 Résultats attendus

| Indicateur | Valeur validée | Fichier de sortie |
| :--- | :--- | :--- |
| **Références actives vendues en ligne** | **714 bouteilles** | `rapport_ca_bottleneck.xlsx` |
| **Chiffre d'affaires global** | **70 568,60 €** | `rapport_ca_bottleneck.xlsx` |
| **Prix unitaire moyen (écart-type)** | **32,49 €** ($\pm 27,81$ €) | `rapport_ca_bottleneck.xlsx` |
| **Vins millésimes / prestige ($z > 2$, soit $> 88,11$ €)** | **30 vins** (Laure — B2B) | `vins_premium.csv` |
| **Vins ordinaires ($z \le 2$)** | **684 vins** (Maria — B2C) | `vins_ordinaires.csv` |

## 📁 Structure du dépôt

```text
bottleneck-kestra-pipeline/
├── data/
│   ├── input/                          # Sources Excel (erp.xlsx, liaison.xlsx, web.xlsx)
│   └── output/                         # Livrables générés (.xlsx et .csv)
├── flows/                              # Workflows Kestra au format YAML
│   ├── 01_bottleneck_pipeline.yaml     # Version 1 : squelette initial
│   ├── 02_bottleneck_nominal.yaml      # Version 2 : pipeline complet sans tests
│   └── 03_bottleneck_avec_tests.yaml   # Version 3 (finale) : pipeline complet avec les 5 tests
├── .env.example                        # Modèle de variables d'environnement (sans secret)
├── .gitignore                          # Exclusions Git (.env, données temporaires, logs)
├── docker-compose.yml                  # Configuration du service Docker Kestra
└── README.md                           # Documentation du projet
```

## 🚀 Installation et lancement en local

### Étape 1 — Cloner le dépôt et préparer l'environnement

1. Vérifiez que **Docker Desktop** est installé et démarré sur votre machine.
2. Ouvrez un terminal et clonez ce dépôt :
   ```bash
   git clone https://github.com/VOTRE_COMPTE/VOTRE_DEPOT.git
   cd bottleneck-kestra-pipeline
   ```
3. Dupliquez `.env.example` en **`.env`**, puis renseignez l'adresse e-mail et le mot de passe souhaités pour l'authentification Kestra *(le fichier `.env` est ignoré par Git)*.
4. Créez les répertoires de travail locaux :
   * **Sous Windows (CMD) :**
     ```cmd
     mkdir data\input data\staging data\output
     ```
   * **Sous Linux / macOS :**
     ```bash
     mkdir -p data/input data/staging data/output
     ```
5. Copiez les trois fichiers sources (**`erp.xlsx`**, **`liaison.xlsx`** et **`web.xlsx`**) dans **`data/input/`**.

### Étape 2 — Démarrer Kestra via Docker Compose

Depuis la racine du projet, lancez le conteneur en arrière-plan :

```bash
docker compose up -d
```

> **Note :** Lors du premier démarrage, Docker télécharge l'image Kestra. Patientez **30 à 60 secondes** avant d'accéder à l'interface web.

### Étape 3 — Importer et exécuter le workflow

1. Ouvrez **`http://localhost:8080`** dans votre navigateur et connectez-vous avec les identifiants définis dans votre fichier `.env`.
2. Dans le menu de gauche, cliquez sur **Flows** $\rightarrow$ **`+ Create`**.
3. Copiez le contenu de **`flows/03_bottleneck_avec_tests.yaml`**, collez-le dans l'éditeur Kestra et cliquez sur **Save**.
4. Cliquez sur **Execute** en haut à droite, conservez les paramètres par défaut (`/app/data/input` et `2.0`) et validez.
5. Attendez que toutes les tâches passent au vert (**~40 secondes** à la première exécution le temps de préparer l'environnement Python, puis **~12 secondes** ensuite).
6. Ouvrez l'onglet **Input/Output** de l'exécution pour télécharger `rapport_ca_bottleneck.xlsx`, `vins_premium.csv` et `vins_ordinaires.csv` dans **`data/output/`**.

## 🧪 Validation et tests

### Test 1 — Vérification de l'exécution nominale

1. **Vue Gantt :** Les **18 tâches** du workflow doivent afficher le statut **`SUCCESS`**.
2. **Onglet Logs :** Vérifiez la présence des deux lignes de validation :
   * `[SUCCÈS TEST 4 - COHÉRENCE CA] CA global validé = 70568.60 EUR sur 714 références.`
   * `[SUCCÈS TEST 5 - Z-SCORE] 30 vins millésimes (z > 2.0) et 684 vins ordinaires validés.`
3. **Livrables :** Contrôlez dans `data/output/` que `rapport_ca_bottleneck.xlsx` affiche **70 568,60 €**, que `vins_premium.csv` contient **30 lignes** et `vins_ordinaires.csv` **684 lignes**.

### Test 2 — Validation du blocage sur anomalie métier

1. Lancez une exécution en remplaçant **`seuil_zscore`** par **`99`**.
2. Les tests 1 à 4 passent au vert, puis le **Test 5 bloque l'exécution (`FAILED`)** car aucun vin millésime n'est détecté au lieu de 30. Aucun fichier CSV erroné n'est produit.

### Test 3 — Validation de la gestion des pannes techniques

1. Lancez une exécution en remplaçant **`dossier_entree`** par **`/app/data/dossier_inexistant`**.
2. Dans l'onglet **Logs**, Kestra effectue **3 tentatives** espacées de 10 secondes (`retry`), puis déclenche la tâche d'alerte **`alerte_panne_pipeline`** (`ERROR`).

## 🔧 Commandes Docker utiles

Toutes ces commandes s'exécutent depuis le répertoire racine du projet (là où se trouve `docker-compose.yml`) :

| Action | Commande |
| :--- | :--- |
| **Démarrer Kestra en arrière-plan** | `docker compose up -d` |
| **Afficher les 50 dernières lignes de logs** | `docker compose logs --tail=50` |
| **Vérifier l'état du conteneur** | `docker compose ps` |
| **Arrêter Kestra** (sans supprimer les données) | `docker compose stop` |
| **Arrêter et supprimer le conteneur** (volumes conservés) | `docker compose down` |