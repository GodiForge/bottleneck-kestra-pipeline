# 🍷 Automatisation du Pipeline Data — BottleNeck

> Mise en place d'un flux automatisé mensuel avec **Kestra**, **DuckDB**, **Python** et **Docker** pour calculer le chiffre d'affaires et trier les vins millésimes.

## 🎯 C'est quoi ce projet ?

Chez **BottleNeck** (marchand de vin), les données sont réparties dans trois fichiers Excel différents :

1. **`erp.xlsx`** : le catalogue interne avec les prix et les stocks ;
2. **`web.xlsx`** : les ventes réalisées sur le site e-commerce ;
3. **`liaison.xlsx`** : la table de correspondance qui permet de relier les références de l'ERP avec celles du site web.

Jusqu'ici, Stéphane (Data Analyst) croisait et nettoyait ces trois fichiers **à la main** chaque mois. C'était long et il y avait un risque d'erreur de copier-coller.

**L'objectif de ce projet** est d'automatiser tout ce travail pour que chaque **15 du mois à 9h00**, le pipeline tourne tout seul et génère **3 fichiers prêts à l'emploi** :

* 📊 **`rapport_ca_bottleneck.xlsx`** : le rapport financier (chiffre d'affaires par bouteille et chiffre d'affaires global) pour le pilotage ;
* 🥂 **`vins_premium.csv`** : la liste des **vins millésimes / prestige** pour **Laure** (clientèle B2B) ;
* 🍷 **`vins_ordinaires.csv`** : la liste des **vins ordinaires** pour **Maria** (clientèle B2C).

## 🛠️ Les outils utilisés (expliqués simplement)

Pour que le système soit rapide, fiable et facile à relancer sur n'importe quel ordinateur, chaque outil a un rôle bien précis :

* **Kestra (le chef d'orchestre) :** Ce n'est pas lui qui fait les calculs. Son rôle est d'organiser la recette : lancer les étapes dans le bon ordre (ou en même temps pour gagner du temps), déclencher le traitement tous les 15 du mois, et alerter si quelque chose bloque.
* **DuckDB (le moteur SQL ultra-rapide) :** Contrairement à un gros serveur de base de données comme PostgreSQL qu'il faut laisser allumé en permanence, DuckDB tourne directement en mémoire. Il lit les fichiers Excel, nettoie les erreurs et croise les tableaux en moins de 2 secondes, puis s'éteint tout seul.
* **Python (`pandas` & `openpyxl`) :** Utilisé à la fin du flux pour faire le calcul statistique (le *Z-Score*) et fabriquer proprement le fichier Excel avec ses deux onglets.
* **Docker :** Permet de faire tourner Kestra dans une boîte isolée prête à l'emploi, sans avoir à installer des dizaines de logiciels sur le poste.

## ⚙️ Comment fonctionne le pipeline ?

Le workflow complet (`flows/03_bottleneck_avec_tests.yaml`) enchaîne **4 grandes étapes** en **12 secondes** environ :

1. **Vérification des fichiers sources :** Contrôle que `erp.xlsx`, `liaison.xlsx` et `web.xlsx` sont bien présents et lisibles.
2. **Nettoyage en parallèle (DuckDB) :** Les 3 fichiers sont nettoyés en même temps :
   * Suppression des lignes vides et des doublons ;
   * Filtrage des vraies fiches produits dans `web.xlsx` (pour retirer les lignes qui correspondaient juste aux photos des bouteilles) ;
   * Conservation des identifiants au format texte afin de ne pas perdre les références qui contiennent un tiret (comme `13127-1`).
3. **Fusion des tableaux et calcul du CA (DuckDB) :** Croisement des 3 sources propres pour obtenir les **714 bouteilles vendues en ligne** et calcul du chiffre d'affaires par produit.
4. **Création des 3 fichiers métiers en parallèle (Python) :**
   * **Branche 1 (Excel) :** Création de `rapport_ca_bottleneck.xlsx` (détail par produit + synthèse globale).
   * **Branche 2 (CSV) :** Calcul du **Z-Score** pour repérer les vins dont le prix sort du lot par rapport à la moyenne du catalogue ($z = \frac{\text{prix} - \text{moyenne}}{\text{écart-type}}$). Si $z > 2$ (soit plus de 88,11 €), le vin va dans `vins_premium.csv`, sinon dans `vins_ordinaires.csv`.

### 🛡️ Les 5 tests qualité (« garde-fous ») et gestion des pannes

Comme le pipeline tourne tout seul le 15 du mois, **5 tests automatiques** vérifient les données à chaque étape :

1. **Zéro doublon** sur les identifiants produits (`825` dans ERP, `825` dans Liaison, `714` dans Web).
2. **Zéro valeur manquante** sur les clés, les prix et les quantités vendues.
3. **Cohérence de la fusion :** exactement **714 produits** réconciliés sans perte.
4. **Cohérence du chiffre d'affaires :** aucun montant négatif et total exact validé (**70 568,60 €**).
5. **Cohérence du tri Z-Score :** exactement **30 vins millésimes** et **684 vins ordinaires** ($30 + 684 = 714$).

> **Que se passe-t-il en cas de problème ?**
> * Si DuckDB a un micro-ralentissement, Kestra réessaie automatiquement **3 fois** (toutes les 10 secondes).
> * Si un fichier manque ou qu'un test détecte une anomalie, **le pipeline s'arrête immédiatement en rouge** avant de fabriquer de faux fichiers, enregistre une alerte claire dans les logs, et permet de relancer en un clic avec le bouton **Replay** une fois le problème corrigé.

## 📊 Chiffres clés validés

| Indicateur | Valeur obtenue | Fichier de sortie |
| :--- | :--- | :--- |
| **Nombre de références actives vendues en ligne** | **714 bouteilles** | `rapport_ca_bottleneck.xlsx` |
| **Chiffre d'affaires global** | **70 568,60 €** | `rapport_ca_bottleneck.xlsx` |
| **Prix moyen d'une bouteille (écart-type)** | **32,49 €** ($\pm 27,81\text{ €}$) | `rapport_ca_bottleneck.xlsx` |
| **Vins millésimes / prestige ($z > 2$, soit $> 88,11\text{ €}$)** | **30 vins** (pour Laure — B2B) | `vins_premium.csv` |
| **Vins ordinaires ($z \le 2$)** | **684 vins** (pour Maria — B2C) | `vins_ordinaires.csv` |

## 📁 Organisation du dépôt

```text
bottleneck-kestra-pipeline/
├── data/
│   ├── input/                          # Dossier où déposer erp.xlsx, liaison.xlsx et web.xlsx
│   └── output/                         # Dossier où récupérer les 3 fichiers générés (.xlsx et .csv)
├── flows/                              # Workflows Kestra au format YAML
│   ├── 01_bottleneck_pipeline.yaml     # Version 1 : squelette initial
│   ├── 02_bottleneck_nominal.yaml      # Version 2 : pipeline complet sans les tests
│   └── 03_bottleneck_avec_tests.yaml   # Version 3 (finale) : pipeline complet avec les 5 tests
├── .env.example                        # Exemple de fichier de configuration (sans mot de passe)
├── .gitignore                          # Empêche d'envoyer les mots de passe et fichiers temporaires
├── docker-compose.yml                  # Configuration du conteneur Docker Kestra
└── README.md                           # Documentation du projet
```

## 🚀 Comment lancer le projet en local

### 1. Préparer les identifiants et les données

1. **Docker Desktop** doit être installé et démarré sur votre ordinateur.
2. Copiez le fichier `.env.example` et renommez la copie en **`.env`**, puis renseignez l'adresse e-mail et le mot de passe de votre choix pour vous connecter à Kestra *(le fichier `.env` est ignoré par Git et n'est jamais publié)*.
3. Placez les trois fichiers sources (`erp.xlsx`, `liaison.xlsx` et `web.xlsx`) dans le dossier **`data/input/`**.

### 2. Démarrer Kestra avec Docker

Ouvrez un terminal dans le dossier du projet et lancez :

```bash
docker compose up -d
```

### 3. Importer et lancer le workflow dans Kestra

1. Ouvrez votre navigateur sur **`http://localhost:8080`** et connectez-vous avec les identifiants définis dans votre `.env`.
2. Allez dans **Flows** $\rightarrow$ **`+ Create`**, copiez-collez le contenu de **`flows/03_bottleneck_avec_tests.yaml`** et cliquez sur **Save**.
3. Cliquez sur le bouton **Execute** (en conservant les paramètres par défaut `/app/data/input` et `2.0`).
4. Une fois toutes les barres passées au vert (~12 secondes), téléchargez les 3 fichiers générés (`rapport_ca_bottleneck.xlsx`, `vins_premium.csv` et `vins_ordinaires.csv`) depuis l'onglet **Input/Output** vers `data/output/`.