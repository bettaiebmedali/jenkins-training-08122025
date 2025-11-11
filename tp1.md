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
