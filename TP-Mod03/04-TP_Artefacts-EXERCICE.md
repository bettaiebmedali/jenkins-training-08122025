# TP3 — Jenkins Avancé : Build & Deploy d'une Application Maven

## 🎯 Objectif
Vous allez reproduire le TP précédent (Build-App → Deploy-App), mais cette fois-ci sur un **vrai projet Maven Java**.

Vous devez créer **2 pipelines Jenkins** :

1. **Build-Maven-App** → compile et produit un artefact `.jar`
2. **Deploy-Maven-App** → télécharge et "déploie" cet artefact

---

## 🧱 1. Job : Build-Maven-App

### ✔ Paramètres obligatoires
- `BUILD_ENV` : DEV, TEST, PROD  
- `VERSION` : version entrée par l’utilisateur

### ✔ Variables d’environnement à définir
- `APP_NAME`
- `BUILD_DATE`

### ✔ Tâches obligatoires du pipeline
1. **Cloner un projet Maven**
   Exemple : `https://github.com/spring-projects/spring-petclinic.git`  
   ou un autre projet Java Maven.

2. **Exécuter un build Maven**
mvn clean package -DskipTests

3. **Archiver le JAR**
- Le JAR se trouve dans `target/*.jar`
- Il doit être archivé avec `archiveArtifacts`

4. **Déclencher automatiquement le job Deploy-Maven-App**
- en passant :
  - `BUILD_ID` = numéro du build
  - `DEPLOY_ENV` = environnement choisi (DEV / TEST / PROD)

---

## 🧱 2. Job : Deploy-Maven-App

### ✔ Paramètres obligatoires
- `DEPLOY_ENV` : DEV, TEST, PROD  
- `BUILD_ID` : Numéro de build à récupérer

### ✔ Tâches obligatoires du pipeline
1. **Télécharger l’artefact JAR** depuis Build-Maven-App  
- URL à construire :  
  `JENKINS_URL/job/Build-Maven-App/BUILD_ID/artifact/target/*.jar`

2. **Placer le JAR dans un dossier `build/`**

3. **Lister le fichier téléchargé**

4. **Simuler un déploiement**  
- Afficher le message :  
  > Déploiement de l’application sur DEPLOY_ENV

---

## 📝 Livrable attendu
Vous devez fournir :

- 2 pipelines Jenkins fonctionnels
- Un build Maven qui génère un artefact
- Le job de déploiement qui télécharge le JAR
- Le chaînage automatique Build → Deploy

Aucun code n’est donné.  
Vous devez écrire **vos pipelines vous-mêmes**.

Bonne chance 🚀
