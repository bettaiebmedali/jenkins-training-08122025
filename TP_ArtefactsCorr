Correction complète — correction_maven.md
# Correction — TP3 Jenkins Maven : Build & Deploy

---

## 🧱 Pipeline 1 : Build-Maven-App

```groovy
pipeline {
    agent any

    parameters {
        choice(name: 'BUILD_ENV', choices: ['DEV', 'TEST', 'PROD'], description: 'Environnement de build')
        string(name: 'VERSION', defaultValue: '1.0.0', description: 'Version de l’application')
    }

    environment {
        APP_NAME = "MyMavenApp"
        BUILD_DATE = "${new Date().format('yyyy-MM-dd_HHmm')}"
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/spring-projects/spring-petclinic.git'
            }
        }

        stage('Build Maven') {
            steps {
                sh "mvn clean package -DskipTests"
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }

    post {
        success {
            build job: 'Deploy-Maven-App',
                  parameters: [
                      string(name: 'BUILD_ID', value: "${env.BUILD_NUMBER}"),
                      string(name: 'DEPLOY_ENV', value: "${params.BUILD_ENV}")
                  ],
                  wait: false
        }
    }
}

🧱 Pipeline 2 : Deploy-Maven-App
pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['DEV', 'TEST', 'PROD'], description: 'Environnement cible')
        string(name: 'BUILD_ID', defaultValue: 'lastSuccessfulBuild', description: 'Build source')
    }

    stages {
        stage('Téléchargement artefacts') {
            steps {
                script {
                    def artifactUrl = "${JENKINS_URL}/job/Build-Maven-App/${params.BUILD_ID}/artifact/target/*.jar"

                    echo "Téléchargement depuis : ${artifactUrl}"

                    sh """
                        mkdir -p build
                        curl -L -o build/app.jar "${artifactUrl}"
                    """

                    sh 'ls -lh build/'
                }
            }
        }

        stage('Déploiement') {
            steps {
                echo "Déploiement de l’application sur ${params.DEPLOY_ENV}"
            }
        }
    }
}

✔ Résultat final attendu

Le job Build-Maven-App :

clone un projet Maven

génère un JAR

archive le JAR

déclenche automatiquement le job Deploy-Maven-App

Le job Deploy-Maven-App :

télécharge le JAR du build précédent

affiche son contenu

simule un déploiement

Pipeline complet validé ✔
