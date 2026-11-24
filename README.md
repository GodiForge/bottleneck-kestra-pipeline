# 🍷 Automatisation du Pipeline Data — BottleNeck

> Mise en place d'un flux automatisé mensuel avec **Kestra**, **DuckDB**, **Python** et **Docker** pour calculer le chiffre d'affaires et trier les vins millésimes.

## 🎯 Contexte et objectif du projet

Chez **BottleNeck** (marchand de vin), les données de l'entreprise sont réparties dans **trois fichiers Excel différents** qui ne communiquent pas directement entre eux :

1. **`erp.xlsx`** : le catalogue interne avec les prix et les stocks ;
2. **`web.xlsx`** : les ventes réalisées sur la boutique en ligne ;
3. **`liaison.xlsx`** : la table de correspondance qui permet de relier les identifiants de l'ERP (`product_id`) avec ceux du site web (`id_web`).

Jusqu'ici, **Stéphane** (Data Analyst) nettoyait et croisait ces trois fichiers **à la main** chaque mois. C'était long, répétitif, avec un risque d'erreur de copier-coller.

**L'objectif de ce projet** est d'automatiser tout ce traitement pour que chaque **15 du mois à 9h00**, le pipeline tourne tout seul et génère **3 fichiers prêts à l'emploi** :

* 📊 **`rapport_ca_bottleneck.xlsx`** : le rapport financier (chiffre d'affaires par bouteille et chiffre d'affaires global) pour le pilotage ;
* 🥂 **`vins_premium.csv`** : la liste des **vins millésimes / prestige** ($z > 2$) pour **Laure** (clientèle B2B) ;
* 🍷 **`vins_ordinaires.csv`** : la liste des **vins ordinaires** ($z \le 2$) pour **Maria** (clientèle B2C).

---

## 🛠️ Les outils utilisés (expliqués simplement)

Pour que le système soit rapide, fiable et facile à relancer sur n'importe quel ordinateur, chaque outil a un rôle bien précis :

* **Kestra (le chef d'orchestre) :** Ce n'est pas lui qui fait les calculs. Son rôle est d'organiser la recette en YAML : lancer les étapes dans le bon ordre (ou en parallèle pour gagner du temps), déclencher le flux tous les 15 du mois, et alerter si quelque chose bloque.
* **DuckDB (le moteur SQL ultra-rapide) :** Contrairement à un gros serveur de base de données qu'il faut laisser tourner en permanence, DuckDB s'exécute directement en mémoire. Il lit les fichiers Excel, nettoie les anomalies et croise les tableaux en moins de 2 secondes, puis s'éteint tout seul.
* **Python (`pandas` & `openpyxl`) :** Utilisé à la fin du flux pour faire le calcul statistique (le *Z-Score*) et fabriquer proprement le fichier Excel multi-onglets ainsi que les deux fichiers CSV.
* **Docker :** Permet de faire tourner Kestra dans un conteneur isolé prêt à l'emploi, sans avoir à installer des dizaines de logiciels sur le poste.

---

## ⚙️ Comment fonctionne le pipeline ?

Le workflow final (`flows/03_bottleneck_avec_tests.yaml`) enchaîne **4 étapes principales** en **12 secondes** environ :

1. **Vérification des fichiers sources :** Contrôle que `erp.xlsx`, `liaison.xlsx` et `web.xlsx` sont bien présents et lisibles.
2. **Nettoyage en parallèle (DuckDB) :** Les 3 fichiers sont nettoyés en même temps :
   * Suppression des lignes vides et des doublons ;
   * Filtrage des vraies fiches produits dans `web.xlsx` (`post_type = 'product'`, pour retirer les 85 lignes sans référence et les lignes qui correspondaient juste aux photos des bouteilles) ;
   * Conservation des identifiants au format texte afin de ne pas perdre les références qui contiennent un tiret (comme `13127-1`).
3. **Fusion des tableaux et calcul du CA (DuckDB) :** Croisement des 3 sources propres pour obtenir les **714 bouteilles vendues en ligne** et calcul du chiffre d'affaires par produit ($\text{prix} \times \text{quantité}$).
4. **Création des 3 fichiers métiers en parallèle (Python) :**
   * **Branche 1 (Excel) :** Création de `rapport_ca_bottleneck.xlsx` (onglet `CA_par_produit` + onglet `Synthese_CA_Global`).
   * **Branche 2 (CSV) :** Calcul du **Z-Score** pour repérer les vins dont le prix sort du lot par rapport à la moyenne du catalogue ($z = \frac{\text{prix} - \text{moyenne}}{\text{écart-type}}$). Si $z > 2$ (soit plus de 88,11 €), le vin va dans `vins_premium.csv`, sinon dans `vins_ordinaires.csv`.

### 🛡️ Les 5 tests qualité (« garde-fous ») et la gestion des pannes

Comme le pipeline tourne automatiquement le 15 du mois, **5 tests bloquants** vérifient les données à chaque étape :

1. **Zéro doublon** sur les identifiants produits (`825` dans ERP, `825` dans Liaison, `714` dans Web).
2. **Zéro valeur manquante** sur les clés, les prix et les quantités vendues.
3. **Cohérence de la fusion :** exactement **714 produits** réconciliés sans perte.
4. **Cohérence du chiffre d'affaires :** aucun montant négatif et total exact validé (**70 568,60 €**).
5. **Cohérence du tri Z-Score :** exactement **30 vins millésimes** et **684 vins ordinaires** ($30 + 684 = 714$).

> **Que se passe-t-il en cas de problème ?**
> * Si DuckDB ou un fichier a un micro-ralentissement, Kestra réessaie automatiquement **3 fois** (toutes les 10 secondes).
> * Si un fichier manque ou qu'un test détecte une anomalie, **le pipeline s'arrête immédiatement en rouge (`FAILED`)** avant de fabriquer de faux fichiers, enregistre une alerte claire dans les logs, et permet de relancer en un clic avec le bouton **Replay** une fois le problème corrigé.

---

## 📊 Chiffres clés validés

| Indicateur | Valeur obtenue | Fichier de sortie |
| :--- | :--- | :--- |
| **Nombre de références actives vendues en ligne** | **714 bouteilles** | `rapport_ca_bottleneck.xlsx` |
| **Chiffre d'affaires global** | **70 568,60 €** | `rapport_ca_bottleneck.xlsx` |
| **Prix moyen d'une bouteille (écart-type)** | **32,49 €** ($\pm 27,81\text{ €}$) | `rapport_ca_bottleneck.xlsx` |
| **Vins millésimes / prestige ($z > 2$, soit $> 88,11\text{ €}$)** | **30 vins** (pour Laure — B2B) | `vins_premium.csv` |
| **Vins ordinaires ($z \le 2$)** | **684 vins** (pour Maria — B2C) | `vins_ordinaires.csv` |

---

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

---

## 🚀 1. Comment débuter et lancer le projet en local (depuis zéro)

### Étape 1 — Récupérer le projet et préparer les dossiers

1. Vérifiez que **Docker Desktop** est installé et démarré sur votre ordinateur.
2. Ouvrez un terminal et clonez ce dépôt :
   ```bash
   git clone https://github.com/VOTRE_COMPTE/VOTRE_DEPOT.git
   cd bottleneck-kestra-pipeline
   ```
3. Copiez le fichier `.env.example` et renommez la copie en **`.env`**, puis renseignez l'adresse e-mail et le mot de passe de votre choix pour vous connecter à Kestra *(le fichier `.env` est ignoré par Git et n'est jamais publié)*.
4. Créez les dossiers de travail nécessaires (ignorés par Git) :
   * **Sous Windows (PowerShell) :**
     ```powershell
     New-Item -ItemType Directory -Force -Path "data\input", "data\staging", "data\output"
     ```
   * **Sous Linux / macOS :**
     ```bash
     mkdir -p data/input data/staging data/output
     ```
5. Placez les trois fichiers sources Excel (**`erp.xlsx`**, **`liaison.xlsx`** et **`web.xlsx`**) dans le dossier **`data/input/`**.

### Étape 2 — Démarrer Kestra avec Docker

Dans le terminal (toujours dans le dossier du projet), lancez :

```bash
docker compose up -d
```

> **Note :** Au tout premier démarrage, Docker télécharge l'image Kestra. Patientez **30 à 60 secondes** le temps que le serveur démarre complètement.

### Étape 3 — Importer et lancer le workflow dans Kestra

1. Ouvrez votre navigateur sur **`http://localhost:8080`** et connectez-vous avec les identifiants définis dans votre fichier `.env`.
2. Dans le menu de gauche, cliquez sur **Flows** $\rightarrow$ **`+ Create`**.
3. Ouvrez le fichier **`flows/03_bottleneck_avec_tests.yaml`**, copiez tout son contenu, collez-le dans l'éditeur Kestra et cliquez sur **Save**.
4. Cliquez sur le bouton violet **Execute** en haut à droite, conservez les paramètres par défaut (`/app/data/input` et `2.0`) et validez.
5. Patientez jusqu'à ce que toutes les barres passent au vert (**~40 secondes** la première fois le temps de charger l'environnement Python, puis **~12 secondes** aux exécutions suivantes).
6. Ouvrez l'onglet **Input/Output** de l'exécution et téléchargez les 3 livrables générés (`rapport_ca_bottleneck.xlsx`, `vins_premium.csv` et `vins_ordinaires.csv`) vers votre dossier **`data/output/`**.

---

## 🧪 2. Comment tout tester et vérifier après le lancement

Une fois le pipeline lancé, vous pouvez effectuer ces **3 vérifications** pour valider le bon fonctionnement complet :

### Test 1 — Contrôler les résultats normaux (chiffres clés et fichiers)

1. **Vérifier les tâches dans Kestra :** Ouvrez l'onglet **Gantt** de votre exécution. Les **18 tâches** (y compris `test_qualite_erp`, `test_qualite_liaison`, `test_qualite_web` et `test_3_coherence_jointure`) doivent toutes afficher le statut vert **`SUCCESS`**.
2. **Vérifier les calculs dans les logs :** Ouvrez l'onglet **Logs** et vérifiez la présence des deux confirmations :
   * `[SUCCÈS TEST 4 - COHÉRENCE CA] CA global validé = 70568.60 EUR sur 714 références.`
   * `[SUCCÈS TEST 5 - Z-SCORE] 30 vins millésimes (z > 2.0) et 684 vins ordinaires validés.`
3. **Vérifier les 3 fichiers dans `data/output/` :**
   * `rapport_ca_bottleneck.xlsx` contient 2 onglets (`CA_par_produit` avec 714 lignes et `Synthese_CA_Global` affichant `70 568,60 €`).
   * `vins_premium.csv` contient exactement **30 vins** (tous à plus de 88,11 €).
   * `vins_ordinaires.csv` contient exactement **684 vins**.

### Test 2 — Tester le blocage en cas d'anomalie métier (les garde-fous)

Pour vérifier que les tests empêchent bien la sortie de fichiers erronés :

1. Dans Kestra, cliquez sur **Execute** sur le workflow `bottleneck_pipeline`.
2. Remplacez la valeur de **`seuil_zscore`** par **`99`** (au lieu de `2.0`) et lancez.
3. **Résultat attendu :** Les contrôles 1 à 4 passent au vert, mais le **Test 5 bloque immédiatement en rouge (`FAILED`)** car aucun vin millésime n'est trouvé au lieu de 30. Aucun fichier CSV faux n'est produit.

### Test 3 — Tester la gestion des pannes techniques (`retry` et `Replay`)

Pour vérifier la réaction du pipeline si un dossier ou un fichier manque le 15 du mois :

1. Dans Kestra, cliquez sur **Execute**.
2. Remplacez la valeur de **`dossier_entree`** par **`/app/data/dossier_inexistant`** et lancez.
3. **Résultat attendu :** Dans l'onglet **Logs**, Kestra tente **3 reprises automatiques** toutes les 10 secondes (`retry`), puis déclenche la tâche d'alerte **`alerte_panne_pipeline`** en rouge (`ERROR`).
4. Vous pouvez ensuite tester la reprise en un clic via le bouton **Replay** sur une exécution valide.

---

## 🔧 Commandes Docker utiles au quotidien

| Action souhaitée | Commande à lancer dans le terminal |
| :--- | :--- |
| **Démarrer Kestra en arrière-plan** | `docker compose up -d` |
| **Arrêter Kestra** (sans perdre vos workflows) | `docker compose stop` |
| **Voir les logs du serveur Kestra** | `docker logs -f bottleneck-kestra` |
| **Vérifier si le conteneur tourne** | `docker ps` |
| **Supprimer le conteneur** (les données restent sauvegardées) | `docker compose down` |