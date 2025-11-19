# TP — Pipeline Jenkins Complet  
GitFlow + Maven + SonarQube + Docker + VM distante + Webhooks GitHub + Choix de Pipeline

---

# 1. Introduction

Ce TP avancé vous guide pour mettre en place un pipeline CI/CD complet sur Jenkins incluant :

- Un workflow complet **GitFlow**
- Un **Multibranch Pipeline** qui détecte automatiquement feature/release/main
- Un pipeline Jenkins scripté ou déclaratif (au choix)
- Exécution de tests unitaires et build **Maven**
- Analyse qualité **SonarQube**
- Construction d’image Docker et push dans un **registry privé**
- Déploiement automatique sur une **VM distante**
- Vérification automatique des **Pull Requests GitHub**
- Intégration des **webhooks GitHub** pour déclencher Jenkins
- Paramètres d'entrée pour version, environnement, etc.

Objectif : reproduire un pipeline moderne de production avec validation avant merge des PR et déploiement automatique après validation.

---

# 2. Choix du type de Pipeline Jenkins

Jenkins propose deux stratégies possibles pour gérer un projet GitFlow :

---

## Option A — Pipeline Freestyle + Webhooks (déconseillé)

- Jenkins exécute un job unique basé sur le Jenkinsfile interne.
- Webhook GitHub déclenche le job.
- Fonctionne mal avec GitFlow (feature/*, release/*, hotfix/*).

**À utiliser uniquement pour petits projets.**

---

## Option B — Pipeline Multibranch (RECOMMANDÉ)

- Jenkins va automatiquement :
  - Scanner toutes les branches
  - Détecter les PR GitHub
  - Générer un job pour chaque branche
- Chaque branche utilise le Jenkinsfile du repo
- Compatible avec GitFlow :
  - `feature/*` → build + tests + sonar + PR validation
  - `develop` → build + tests + sonar + docker + deploy
  - `main` → build final + tag version + publication image stable

**Option RECOMMANDÉE pour GitFlow.**

---

# 3. Configuration GitHub Webhooks

Dans GitHub :

```
Settings → Webhooks → Add Webhook
```

Mettre :

```
Payload URL : https://VOTRE_JENKINS/github-webhook/
Content type : application/json
Secret : (facultatif)
```

Sélectionner les événements suivants :

- ✔ Push
- ✔ Pull Request
- ✔ Create (pour tags release)
- ✔ Delete
- ✔ Workflow run (optionnel)

GitHub enverra désormais automatiquement :

- un webhook à chaque commit
- un webhook à chaque PR
- un webhook à chaque merge

---

# 4. Structure GitFlow

Vos développeurs devront suivre la structure suivante :

```
main
develop
feature/<feature_name>
release/<version>
hotfix/<fix>
```

---

# 5. Préparation Jenkins

### Installer plugins nécessaires :

- GitHub Integration
- GitHub Branch Source
- Pipeline
- Pipeline: Multibranch
- Docker Pipeline
- SSH Agent
- SonarQube Scanner
- Credentials Binding
- Generic Webhook Trigger (optionnel)

### Configurer outils Jenkins :

```
Manage Jenkins → Global Tool Configuration
```

- Maven : `maven3`
- JDK : `jdk17` ou `jdk21`

### Variables globales recommandées (Manage Jenkins → Configure System)

```
SONARQUBE_URL = http://vps-xxxx:9000
REGISTRY_URL = http://vps-xxxx:5000
```

---

# 6. Configuration des Credentials Jenkins

Ajouter :

| ID | Type | Description |
|----|------|-------------|
| `github_token` | Secret Text | Accès GitHub API |
| `vm_ssh` | SSH Username/Private Key | Login VM distante |
| `docker_registry` | Username/Password | Connexion registry privé |
| `sonarqube_token` | Secret Text | SonarQube access token |

---

# 7. Création du Multibranch Pipeline

Dans Jenkins :

```
New Item → Multibranch Pipeline
```

Configurer :

### **Branch Sources → GitHub**

- Repo : `bitboxtraining-web/jenkins-tps`
- Credentials : `github_token`

### **Build Triggers**

✔ “Scan by webhook”  
✔ Enter webhook URL :  
```
https://VOTRE_JENKINS/github-webhook/
```

### **Orphaned Item Strategy**

- Delete after 3 days

---

# 8. Jenkinsfile — VERSION LONGUE ET DOCUMENTÉE

Créer un fichier `Jenkinsfile` dans le repo :

```groovy



pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    environment {
        REGISTRY = "vps-36f602ea.vps.ovh.net:5000"
        SONAR_URL = "http://vps-36f602ea.vps.ovh.net:9000"
    }

    stages {

        /* ----- 1. QUALITY GATE pour TOUT sauf main ----- */
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    dir('backend-demo') {
                        sh """
                            mvn clean verify sonar:sonar \
                                -Dsonar.projectKey=demo-backend \
                                -Dsonar.host.url=$SONAR_URL \
                                -DskipTests
                        """
                    }
                }
            }
        }

        /* ----- 2. BUILD pour TOUT ----- */
        stage('Build Maven') {
            steps {
                dir('backend-demo') {
                    sh "mvn clean package -DskipTests"
                }
            }
        }

        /* ----- 3. TESTS : seulement develop + feature/* ----- */
        stage('Unit Tests') {
            when { 
                expression { 
                    env.BRANCH_NAME == "develop" ||
                    env.BRANCH_NAME.startsWith("feature/")
                }
            }
            steps {
                dir('backend-demo') {
                    sh "mvn test"
                }
            }
        }

        /* ----- 4. BUILD DOCKER : develop, release/*, main ----- */
        stage('Docker Build') {
            when {
                anyOf {
                    branch "develop"
                    branch "main"
                    expression { env.BRANCH_NAME.startsWith("release/") }
                }
            }
            steps {
                dir('backend-demo') {
                    sh """
                        docker build -t ${REGISTRY}/backend:${BRANCH_NAME} .
                        docker push ${REGISTRY}/backend:${BRANCH_NAME}
                    """
                }
            }
        }

        /* ----- 5. DEPLOY : seulement main ----- */
        stage('Deploy to Production') {
            when { 
                branch "main" 
            }
            steps {
                sh """
                ssh -o StrictHostKeyChecking=no debian@217.182.207.167 '
                    sudo docker pull ${REGISTRY}/backend:main &&
                    sudo docker stop backend || true &&
                    sudo docker rm backend || true &&
                    sudo docker run -d --name backend -p 8080:8080 ${REGISTRY}/backend:main
                '
                """
            }
        }
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }
}




```

---

# 9. Synthèse GitFlow vs Pipeline

| Branche | Action |
|--------|--------|
| `feature/*` | Build + Tests + Sonar + Validation PR |
| `develop` | Build + Sonar + Docker Build + Push + Deploy |
| `release/*` | Build + Tests + Sonar |
| `main` | Build Final + Docker Image stable |
| PR Github | Analyse complète + blocage merge si erreur |

---

# 10. Tests du pipeline

## Tester une branche feature :
```
git checkout -b feature/login
git push origin feature/login
```

## Créer une Pull Request :
→ GitHub déclenche un webhook  
→ Jenkins valide automatiquement

## Merge dans develop :
→ Build complet + Déploiement VM

---

# 11. Bonus : Webhook personnalisé (optionnel)

Si vous voulez déclencher Jenkins depuis GitHub manuellement :

GitHub → Settings → Webhook → Add → URL :

```
https://jenkins.example.com/generic-webhook-trigger/invoke
```

Exemple payload JSON :

```json
{
  "branch": "develop",
  "action": "deploy"
}
```

---

# 12. Conclusion

Vous disposez maintenant d’un pipeline Jenkins **ultra-complet**, aligné sur les pratiques professionnelles modernes, incluant :

- GitFlow automatisé
- Multibranch intelligent
- Analyse qualité avancée
- Docker CI/CD
- Déploiement distant automatisé
- Validation stricte des PR
- Webhooks GitHub pour intégration native

