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


## Troubleshoot
# 📝 Résumé + Actions à faire côté SonarQube pour résoudre le problème `waitForQualityGate`

Lorsque ton pipeline affiche :

Timeout set to expire in 3 min 0 sec
Checking status of SonarQube task 'AZqV0W...'
Status is 'IN_PROGRESS'


👉 **Cela signifie que Jenkins interroge SonarQube, mais SonarQube ne renvoie pas encore le statut final (OK / ERROR)** même si ton analyse semble terminée côté interface.

Ce problème est **fréquent** et lié au fait que SonarQube n'a pas encore "finalisé" le Quality Gate dans son API, même si l’analyse apparaît comme terminée visuellement.

---

# 🚨 Pourquoi ça arrive ?
- SonarQube peut avoir fini d’analyser **mais pas encore calculé le Quality Gate**.
- Ou bien Jenkins n’arrive pas à récupérer le résultat via l’API SonarQube.
- Ou un problème de configuration empêche le retour du statut.

---

# ✅ Actions à faire côté SonarQube

### 1️⃣ **Vérifier que le webhook "sonarqube → jenkins" existe**
Sans webhook, Jenkins NE PEUT PAS recevoir le Quality Gate final.

Dans SonarQube :

Administration > Configuration > Webhooks


Créer un webhook :

Name: jenkins
URL : http://<jenkins-url>/sonarqube-webhook/


✔ Obligatoire  
✔ Respecter exactement `/sonarqube-webhook/`  
✔ Redémarrer le projet et retester

---

### 2️⃣ **Tester le webhook**
SonarQube → Webhooks → Cliquer sur ton webhook → **Recent deliveries**

Vérifier que la dernière livraison est :

Status: 200 (success)


Si tu vois :
- `500` → Jenkins a rejeté l'appel  
- `404` → Mauvaise URL  
- `403` → Crumb problem  
- `ECONNREFUSED` → Jenkins inaccessible  

---

### 3️⃣ **Vérifier la clé du projet**
Dans SonarQube :

Project Settings > General > Project Key


Doit être **exactement le même** que dans ton Jenkinsfile :

Exemple :

```groovy
withSonarQubeEnv('sonar-server') {
    sh "mvn sonar:sonar -Dsonar.projectKey=my-app"
}

4️⃣ Vérifier ton SonarQube token

Dans Jenkins → Credentials :

Kind: Secret Text
ID : sonar-token

Et dans pipeline :

-Dsonar.login=${SONAR_TOKEN}

5️⃣ Donner plus de temps avant timeout

Si Sonar met plus que 3 minutes :

timeout(time: 6, unit: 'MINUTES') {
    waitForQualityGate abortPipeline: true
}

🟢 En résumé simple
Problème	Solution
Jenkins reste en IN_PROGRESS	Créer / vérifier webhook Sonar → Jenkins
SonarQube ne répond pas	Vérifier URL + statut 200
Mauvais projectKey	Vérifier clé Sonar / Jenkinsfile
Analyse trop longue	Augmenter timeout
✔️ Avec ça, ton waitForQualityGate fonctionnera correctement.