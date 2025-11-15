
---
## 🧰 TP2 — Jenkins Avancé : Multi-environnements et Artefacts

### 🎯 Objectif
Créer deux pipelines interconnectés :
- `Build-App` : produit des artefacts
- `Deploy-App` : récupère et déploie ces artefacts

### 🔑 Credentials nécessaires
- `git_creds` → Username+Password
- `deploy_server` → SSH Username with private key

### 🧱 Job 1 : Build-App

```groovy
pipeline {
    agent any

    parameters {
        choice(name: 'BUILD_ENV', choices: ['DEV', 'TEST', 'PROD'], description: 'Environnement de build')
        string(name: 'VERSION', defaultValue: '2.0.0', description: 'Version de l’application')
    }

    environment {
        APP_NAME = "SuperApp"
        BUILD_DATE = "${new Date().format('yyyy-MM-dd_HHmm')}"
        GIT_CREDS = credentials('git_creds')
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Clonage du code (simulé)..."
                sh 'mkdir -p src && echo "print(\"Hello Jenkins!\")" > src/app.py'
            }
        }

        stage('Build') {
            steps {
                echo "Compilation pour ${params.BUILD_ENV}"
                sh '''
                    mkdir -p build
                    echo "Application: ${APP_NAME}" > build/info.txt
                    echo "Version: ${VERSION}" >> build/info.txt
                    echo "Env: ${BUILD_ENV}" >> build/info.txt
                    echo "Date: ${BUILD_DATE}" >> build/info.txt
                '''
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'build/*.txt', fingerprint: true
            }
        }
    }

    post {
        success {
            build job: 'Deploy-App',
                  parameters: [
                      string(name: 'SOURCE_BUILD', value: "${env.BUILD_NUMBER}"),
                      string(name: 'DEPLOY_ENV', value: "${params.BUILD_ENV}")
                  ],
                  wait: false
        }
    }
}
```

### 🧱 Job 2 : Deploy-App

```groovy
pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['DEV', 'TEST', 'PROD'], description: 'Environnement de déploiement')
        string(name: 'SOURCE_BUILD', defaultValue: 'lastSuccessfulBuild', description: 'Numéro du build à déployer')
    }

    environment {
        SERVER_CREDS = credentials('deploy_server')
    }

    stages {
        stage('Téléchargement artefacts') {
            steps {
                copyArtifacts(
                    projectName: 'Build-App',
                    selector: specific("${params.SOURCE_BUILD}"),
                    filter: 'build/*.txt'
                )
                sh 'cat build/info.txt'
            }
        }

        stage('Déploiement') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'deploy_server', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
                    sh '''
                        echo "Connexion SSH simulée : ${SSH_USER}@serveur"
                        echo "Déploiement de l’application sur ${DEPLOY_ENV}"
                    '''
                }
            }
        }
    }
}
```

### 🧾 Résumé
Tu as appris à :
- Chainer plusieurs pipelines  
- Partager des artefacts entre jobs  
- Utiliser plusieurs credentials  
- Gérer des environnements distincts (DEV, TEST, PROD)
