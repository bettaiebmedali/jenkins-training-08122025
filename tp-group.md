# TP Jenkins --- GitFlow, Maven, Organisation des Vues et Travail d'Équipe

**Compatible : Jenkins 2.528.2**

Ce TP vous apprendra à organiser un workflow d'équipe complet autour :

-   d'un dépôt GitHub\
-   d'un workflow GitFlow\
-   d'un projet Maven commun\
-   de Jenkins (folders, views, jobs)

L'objectif est que chaque équipe travaille sur **un seul projet Maven
commun**, en créant ses propres branches features et leurs jobs Jenkins
associés.

------------------------------------------------------------------------

## 🎯 Objectifs du TP

-   Former **2 équipes de 5 personnes**
-   Créer **1 projet Maven par équipe**
-   Mettre en place **un workflow GitFlow complet**
-   Créer des branches feature individuelles
-   Organiser Jenkins avec **folders + views**
-   Créer **1 job Jenkins par personne**
-   Automatiser les builds Maven des branches feature

------------------------------------------------------------------------

## 🧩 1. Préparation de l'environnement

### ✔ Prérequis

-   Jenkins **2.528.2**
-   Maven installé
-   Git installé
-   Plugins Jenkins :
    -   Git Plugin
    -   Maven Integration Plugin
    -   Folders Plugin

------------------------------------------------------------------------

## 🧩 2. Organisation des équipes

Deux équipes travaillent en parallèle :

### 👥 Équipe A

-   5 stagiaires

### 👥 Équipe B

-   5 stagiaires

Chaque équipe désigne :

-   **1 responsable GitFlow**
-   **1 responsable Jenkins**
-   **3 développeurs feature**

------------------------------------------------------------------------

## 🧩 3. Création du dépôt GitHub (1 par équipe)

Chaque équipe crée un dépôt GitHub :

-   `team-a-jenkins-project`
-   `team-b-jenkins-project`

Contenu obligatoire :

-   `README.md`
-   `.gitignore` (Java)
-   Branche : `main`
-   Branche : `develop`

------------------------------------------------------------------------

## 🧩 4. GitFlow --- Création du projet Maven (un seul projet par équipe)

### 🔹 Étape 1 --- Création de la branche d'initialisation

``` bash
git checkout -b feature/init-projet
```

Dans cette branche :

-   création du projet Maven\
-   ajout de la structure minimale\
-   ajout d'une classe Java simple\
-   commit + push

### 🔹 Étape 2 --- Intégration dans develop

-   Création d'une Pull Request vers `develop`\
-   Merge après validation

Ce projet Maven devient la base de toutes les futures features.

------------------------------------------------------------------------

## 🧩 5. GitFlow --- Développement individuel

Chaque stagiaire :

``` bash
git checkout develop
git pull
git checkout -b feature/<prenom>
```

Travail demandé :

-   ajouter une classe\
-   modifier une méthode\
-   ajouter un test simple\
-   commit + push\
-   créer une Pull Request vers `develop`\
-   merge après validation

------------------------------------------------------------------------

## 🧩 6. Jenkins --- Organisation générale

Créer dans Jenkins deux dossiers :

-   `Team-A`
-   `Team-B`

Dans chaque dossier :

Chaque stagiaire doit :

-   créer **sa propre view personnalisée**
-   créer **un job Maven** lié à sa branche `feature/<prenom>`

Chaque view doit n'afficher **que les jobs du stagiaire**.

------------------------------------------------------------------------

## 🧩 7. Jenkins --- Création des jobs Maven

Pour chaque stagiaire :

Nom du job :

    build-<prenom>

Configuration :

-   Git repository : dépôt de l'équipe\
-   Branch : `feature/<prenom>`\
-   Build command : `clean install`\
-   Pas de credentials\
-   Agent : `any`

Objectif : **réussir un build individuel.**

------------------------------------------------------------------------

## 🧩 8. Cycle GitFlow final

L'équipe réalise :

1.  Merge de toutes les features dans `develop`\
2.  Création de la branche de release :

``` bash
git checkout -b release/1.0.0
```

3.  Mise à jour de la version dans le `pom.xml`\
4.  Merge dans `main` et `develop`\
5.  Création du tag final :

``` bash
git tag v1.0.0
git push origin --tags
```

------------------------------------------------------------------------

## 🧩 9. Livrables attendus

-   URL du repo GitHub
-   Liste des branches
-   Screenshots Jenkins :
    -   folders\
    -   views\
    -   jobs\
    -   logs de build\
-   Tag final : `v1.0.0`
