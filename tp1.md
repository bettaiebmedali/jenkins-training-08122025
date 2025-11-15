# 🧱 Formation Jenkins Complète — TPs Pratiques (v2.528.1)

---
## 🧩 TP1 — Les Fondamentaux de Jenkins

### 🎯 Objectif
Apprendre à utiliser les éléments de base de Jenkins :
- Variables d’environnement  
- Credentials (identifiants sécurisés)  
- Paramètres de build  
- Artefacts de build  

Ce TP fonctionne sur **Jenkins v2.528.1** ou supérieure.

### 🪜 1. Préparation

#### 1.1 Plugins requis
- Pipeline  
- Credentials  
- Credentials Binding  
- Git  
- Workspace Cleanup (optionnel)

### 🔑 2. Gestion des Credentials

1. Menu principal → **Gérer Jenkins → Identifiants**  
2. Clique sur **System → (global)**  
3. Clique sur **Add Credentials**  
4. Remplis :  
   - Type : Username with password  
   - ID : `git_user`  
   - Description : Identifiant Git pour Jenkins  
   - Username / Password : ton compte Git ou ton token

### ⚙️ 3. Créer un projet Pipeline

1. Depuis le tableau de bord → **Nouveau Item**  
2. Nom : `TP1-Jenkins-Basics`  
3. Type : **Pipeline**  
4. Valide avec **OK**

### 📜 4. Script Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        APP_NAME = "MonApplication"
        BUILD_ENV = "DEV"
        GIT_CREDS = credentials('git_user')
    }

    parameters {
        string(name: 'VERSION', defaultValue: '1.0.0', description: 'Version du build')
        choice(name: 'DEPLOY_ENV', choices: ['DEV', 'TEST', 'PROD'], description: 'Environnement cible')
    }

    stages {
        stage('Init') {
            steps {
                echo "=== Initialisation ==="
                echo "Application : ${APP_NAME}"
                echo "Version : ${params.VERSION}"
                echo "Environnement : ${params.DEPLOY_ENV}"
                echo "Utilisateur Git : ${GIT_CREDS_USR}"
            }
        }

        stage('Build') {
            steps {
                sh '''
                    echo "=== Compilation ==="
                    mkdir -p build
                    echo "Fichier de build pour $APP_NAME ($VERSION)" > build/info.txt
                '''
            }
        }

        stage('Tests') {
            steps {
                sh 'echo "Tests OK" > build/tests.txt'
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'build/*.txt', fingerprint: true
            }
        }
    }

    post {
        always {
            echo "Pipeline terminé."
        }
    }
}
```

### 🧾 Résumé
Ce TP t’a appris à :
- Créer un pipeline Jenkins basique  
- Définir et utiliser des variables d’environnement  
- Paramétrer des builds  
- Manipuler des credentials sécurisés  
- Archiver des artefacts de build
``` 