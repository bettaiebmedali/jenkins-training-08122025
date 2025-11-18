# TP : Pipeline Jenkins complet avec SonarQube externe

## 🎯 Objectif

Mettre en place un pipeline Jenkins suivant GitFlow permettant : - Build
Maven - Analyse SonarQube sur une VM externe - Quality Gate -
Préparation au déploiement

------------------------------------------------------------------------

## 🧱 1. Pré-requis

### ✔️ Jenkins

Version recommandée : **2.528.2**

Plugins à installer : - **Pipeline** - **Git** - **SonarQube Scanner** -
**Workspace Cleanup** - **BlueOcean** (optionnel)

### ✔️ SonarQube

URL d'accès : `http://<ip-sonar>:9000`

Créer un token : 1. Login SonarQube 2. **My Account → Security** 3.
Generate token

------------------------------------------------------------------------

## ⚙️ 2. Configuration Jenkins

### 2.1. Déclarer SonarQube dans Jenkins

`Manage Jenkins → System → SonarQube Servers → Add SonarQube`

  Champ        Valeur
  ------------ ---------------------------------
  Name         sonar-server
  Server URL   http://`<ip-sonar>`{=html}:9000
  Token        celui généré dans SonarQube

------------------------------------------------------------------------

### 2.2. Configurer Maven

`Manage Jenkins → Global Tool Configuration`

Ajouter :

    Name : maven3
    Version : Install automatically

------------------------------------------------------------------------

## 🐙 3. GitFlow utilisé

-   **develop** : branche d'intégration dev
-   **release** : futurs déploiements
-   **main** : production

Le pipeline démarre sur **develop**.

------------------------------------------------------------------------

## 📝 4. Structure projet

    jenkins-tps/
     ├── demo-backend/
     │    ├── pom.xml
     │    └── src/...
     └── Jenkinsfile

------------------------------------------------------------------------

## 🔧 5. Jenkinsfile complet (avec SonarQube externe)

``` groovy
pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'develop', url: 'https://github.com/bitboxtraining-web/jenkins-tps.git'
            }
        }

        stage('Build Backend') {
            steps {
                dir('demo-backend') {
                    sh "mvn clean install -DskipTests"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    dir('demo-backend') {
                        sh '''
                            mvn clean verify sonar:sonar                               -Dsonar.projectKey=demo-backend                               -Dsonar.host.url=$SONAR_HOST_URL                               -DskipTests
                        '''
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    timeout(time: 3, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
    }
}
```

------------------------------------------------------------------------

## 🧪 6. Vérifications

### Vérifier que Jenkins accède à Sonar :

``` bash
curl http://<ip-sonar>:9000/api/system/status
```

Doit retourner :

    {"status":"UP"}

------------------------------------------------------------------------

## 🚀 7. Résultat attendu

-   Jenkins build avec Maven → OK\
-   Analyse Sonar envoyée sur VM externe → OK\
-   Quality Gate valide → Pipeline continue

------------------------------------------------------------------------

## 🎓 Fin du TP

Vous disposez maintenant d'un pipeline CI complet GitFlow + Maven +
Sonar.
