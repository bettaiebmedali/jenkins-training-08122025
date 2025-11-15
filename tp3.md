
---
## 🚀 TP3 — Jenkins Expert : CI/CD Complet

### 🎯 Objectif
Créer un pipeline multibranch Git lisant un `.env`, construisant et poussant une image Docker.

### 🧩 Structure du projet
```
my-app/
├── .env
├── Dockerfile
├── app.py
└── Jenkinsfile
```

### Exemple de `.env`
```
APP_NAME=MicroApp
APP_PORT=8080
APP_VERSION=3.0.0
```

### Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        DOCKER_CREDS = credentials('docker_hub')
        ENV_FILE = readFile('.env').split("\n")
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${env.BRANCH_NAME}", url: 'https://github.com/monrepo/my-app.git'
            }
        }

        stage('Charger .env') {
            steps {
                script {
                    for (line in ENV_FILE) {
                        if (line.trim() && !line.startsWith('#')) {
                            def parts = line.tokenize('=')
                            env[parts[0]] = parts[1]
                        }
                    }
                    echo "Application : ${APP_NAME}"
                    echo "Version : ${APP_VERSION}"
                }
            }
        }

        stage('Build Docker') {
            steps {
                script {
                    docker.build("${APP_NAME}:${APP_VERSION}")
                }
            }
        }

        stage('Push Docker') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker_hub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                        echo "$PASS" | docker login -u "$USER" --password-stdin
                        docker push ${APP_NAME}:${APP_VERSION}
                    '''
                }
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                echo "Déploiement de ${APP_NAME}:${APP_VERSION} sur PROD..."
            }
        }
    }

    post {
        success {
            echo "✅ Build réussi pour ${APP_NAME}:${APP_VERSION}"
        }
        failure {
            echo "❌ Échec du build ${APP_NAME}"
        }
    }
}
```

### 🧾 Résumé
- Jenkinsfile multibranch Git  
- Variables d’environnement depuis `.env`  
- Construction et push Docker automatisé  
- Conditions par branche (`main`)  
- Sécurisation via credentials
