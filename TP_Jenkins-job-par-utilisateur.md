# TP Jenkins --- Gestion des Jobs par Utilisateur (Matrix Security + Dossiers)

Compatible : **Jenkins 2.528.2**

Ce TP vous apprend à configurer Jenkins pour que **chaque utilisateur ne
voie que ses propres jobs**, grâce à :

-   Matrix-Based Security\
-   Dossiers Jenkins (CloudBees Folders)\
-   Permissions par utilisateur\
-   Vues filtrées automatiquement

------------------------------------------------------------------------

# 🎯 Objectifs du TP

-   Activer la sécurité avancée dans Jenkins\
-   Créer un dossier par utilisateur\
-   Protéger les jobs par permissions\
-   Vérifier que chaque utilisateur ne voit que son espace\
-   Mettre en place une vue personnalisée par utilisateur

------------------------------------------------------------------------

# 🧩 1. Activer Matrix-Based Security

1.  Aller dans :\
    **Manage Jenkins → Security → Configure Global Security**

2.  Dans **Authorization**, sélectionner :\
    **Matrix-based security**

3.  Ajouter les utilisateurs :

    -   `dev1`\
    -   `qa1`\
    -   `ops1`

4.  Donner uniquement :

```{=html}
<!-- -->
```
    Overall: Read
    Job: Read
    View: Read

⚠ **Ne jamais donner Job:Discover ou Job:Configure à tout le monde !**

------------------------------------------------------------------------

# 🧩 2. Installer (si nécessaire) CloudBees Folders

Menu :\
**Manage Jenkins → Manage Plugins → Available**

Rechercher : **Folders**

Installer + Redémarrer Jenkins

------------------------------------------------------------------------

# 🧩 3. Créer un dossier par utilisateur

Dashboard → **New Item**

Créer :

    DEV/
    QA/
    OPS/

Chaque dossier correspondra à l'espace d'un utilisateur.

------------------------------------------------------------------------

# 🧩 4. Appliquer les permissions par dossier

### Exemple : dossier `DEV/`

1.  Aller dans : `DEV/` → **Configure**
2.  Descendre à : **Enable project-based matrix authorization**
3.  Activer le bouton
4.  Ajouter l'utilisateur `dev1`
5.  Donner les permissions :

```{=html}
<!-- -->
```
    Job: Read
    Job: Build
    Job: Configure
    Job: Discover

❌ Ne pas ajouter `qa1`\
❌ Ne pas ajouter `ops1`

Résultat :\
➡ Seul **dev1** voit et gère les jobs dans `DEV/`.

Répéter pour :

-   `QA/` → utilisateur autorisé : `qa1`
-   `OPS/` → utilisateur autorisé : `ops1`

------------------------------------------------------------------------

# 🧩 5. Créer les jobs dans les dossiers

### Exemple :

Dans **DEV/** placer :

    dev-build-backend
    dev-build-frontend
    dev-test-ui

Dans **QA/** :

    qa-test-backend
    qa-test-api

Dans **OPS/** :

    ops-deploy-prod
    ops-maintenance

Chaque job hérite automatiquement des permissions de son dossier.

------------------------------------------------------------------------

# 🧩 6. Vérifier que l'isolation fonctionne

Connexion :

### 🔹 Si vous vous connectez en `dev1`

Vous devez voir uniquement :

    DEV/

### 🔹 Si vous vous connectez en `qa1`

Vous devez voir :

    QA/

### 🔹 Si vous vous connectez en `ops1`

Vous devez voir :

    OPS/

------------------------------------------------------------------------

# 🧩 7. Bonus --- Ajouter une vue par utilisateur

1.  Dans le dossier (`DEV/`) → **+ New View**
2.  Nom : `Vue Dev`
3.  Type : **List View**
4.  Activer : **Use a regular expression**

Regex :

    ^dev-.*$

------------------------------------------------------------------------

# 🧩 8. Bonus --- Empêcher l'accès au Dashboard global

Dans Matrix Security globale :

Pour tous les utilisateurs (dev1, qa1, ops1) :

❌ décocher

    View: Create
    Job: Create

Cela force les utilisateurs à travailler uniquement **dans leur dossier
sécurisé**.

------------------------------------------------------------------------

# 🎉 Fin du TP

Vous avez maintenant un Jenkins sécurisé où chaque utilisateur travaille
dans son propre espace, sans voir les jobs des autres.


