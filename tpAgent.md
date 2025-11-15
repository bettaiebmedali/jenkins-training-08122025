

# 📄 **tp-jenkins-agents.md**


# TP – Mise en place d’agents Jenkins en Docker (Java, Python, Node.js)

## 🎯 Objectifs du TP
Dans ce TP, vous allez :

- Installer et configurer **3 agents Jenkins Docker** :
  - un agent **Java**
  - un agent **Python**
  - un agent **Node.js**
- Effectuer des **tests de connectivité**
- Déployer un **pipeline Jenkins multi-agents**
- Vérifier l'exécution des jobs sur chaque environnement dédié



# 🧱 1. Pré-requis

### Logiciels nécessaires
- Jenkins fonctionnel (controller)
- Docker installé
- Plugin **Docker** ou **Docker Pipeline** (optionnel)
- Port **50000** ouvert pour les agents JNLP

### Vérifier la configuration JNLP dans Jenkins  
`Manage Jenkins → Security → Agents`

- Agent protocols :  
  ✔️ `JNLP4-connect`  
  ✔️ `Ping`

- **TCP port for inbound agents** :  
  ✔️ `Fixed 50000`

---

# 🧱 2. Création des agents Jenkins

Nous allons créer **3 nodes permanents** :

| Nom           | Rôle            | Label  |
|---------------|-----------------|---------|
| agent-java    | Build Java/JDK  | java    |
| agent-python  | Build Python    | python  |
| agent-node    | Build Node.js   | node    |

---

# 🟩 2.1 Création des nodes dans Jenkins

Pour chaque agent :

```

Manage Jenkins → Nodes → New Node

```

**Nom :**
```

agent-java   ou   agent-python   ou   agent-node

```

**Type :**
```

Permanent Agent

```

**Remote root directory :**
```

/home/debian/jenkins

```

**Labels :**
```

java   ou   python   ou   node

````

**Launch method :**  
✔️ **Launch agent by connecting it to the controller**

**Enregistrer.**

Ensuite Jenkins affiche :

- un **SECRET**
- le **nom du node**
- le lien pour télécharger `agent.jar`

Conservez le secret pour la suite.

---

# 🧱 3. Construction des images Docker

## 🟦 3.1 Création de l’agent Java

Créer `Dockerfile.java` :

```Dockerfile
FROM jenkins/inbound-agent:jdk17

# (Optional) set user, but the base image already defaults to `jenkins`
USER jenkins

````

Build :

```bash
docker build -t jenkins-agent-java -f Dockerfile.java .
```

---

## 🟧 3.2 Création de l’agent Python

Créer `Dockerfile.python` :

```Dockerfile
FROM jenkins/inbound-agent:latest

USER root
RUN apt-get update \
    && apt-get install -y python3 python3-pip \
    && apt-get clean

USER jenkins
```

Build :

```bash
docker build -t jenkins-agent-python -f Dockerfile.python .
```

---

## 🟩 3.3 Création de l’agent Node.js

Créer `Dockerfile.node` :

```Dockerfile
FROM jenkins/inbound-agent:latest

USER root
RUN apt-get update && apt-get install -y curl \
    && curl -fsSL https://deb.nodesource.com/setup_18.x | bash - \
    && apt-get install -y nodejs

USER jenkins
```

Build :

```bash
docker build -t jenkins-agent-node -f Dockerfile.node .
```

---

# 🧱 4. Lancement des agents Docker

Remplacez `<SECRET>` et `<NODE_NAME>` par les valeurs Jenkins.

### 🟦 Agent Java

```bash
docker run -d \
  --name agent-java \
  jenkins-agent-java \
  -url http://<IP_JENKINS>:8080 \
  <SECRET_JAVA> \
  agent-java
```

### 🟧 Agent Python

```bash
docker run -d \
  --name agent-python \
  jenkins-agent-python \
  -url http://<IP_JENKINS>:8080 \
  <SECRET_PYTHON> \
  agent-python
```

### 🟩 Agent Node.js

```bash
docker run -d \
  --name agent-node \
  jenkins-agent-node \
  -url http://<IP_JENKINS>:8080 \
  <SECRET_NODE> \
  agent-node
```

---

# 🧪 5. Tests de connectivité des agents

### Vérifier les logs Docker :

```bash
docker logs agent-java
docker logs agent-python
docker logs agent-node
```

Vous devez voir :

```
INFO: Connected to Jenkins controller
```

### Vérifier dans Jenkins :

```
Manage Jenkins → Nodes
```

Les trois agents doivent être **ONLINE**.

---

# 🧪 6. Pipeline Jenkins multi-agents (test final)

Créer un pipeline :

`Jenkinsfile`

```groovy
pipeline {
    agent none
    stages {

        stage('Test Java Agent') {
            agent { label 'java' }
            steps {
                sh 'echo "Java agent OK"'
                sh 'java -version'
            }
        }

        stage('Test Python Agent') {
            agent { label 'python' }
            steps {
                sh 'echo "Python agent OK"'
                sh 'python3 --version'
            }
        }

        stage('Test Node Agent') {
            agent { label 'node' }
            steps {
                sh 'echo "Node agent OK"'
                sh 'node -v'
            }
        }
    }
}
```

---

# 🏁 7. Résultat attendu

Dans la console Jenkins, vous devriez voir :

```
Java agent OK
openjdk version "17..."

Python agent OK
Python 3.x.x

Node agent OK
v18.x.x
```

Chaque stage doit s’exécuter sur **son agent spécialisé**.

---

# 📦 8. Nettoyage (optionnel)

```bash
docker rm -f agent-java agent-python agent-node
docker rmi jenkins-agent-java jenkins-agent-python jenkins-agent-node
```

---

# 🎉 TP terminé !

Vous avez maintenant :

✔️ 3 agents Jenkins en Docker
✔️ Java + Python + Node isolés
✔️ Pipeline multi-agents
✔️ Tests fonctionnels

Bravo ! 🚀

```

---

Si tu veux, je peux aussi te générer :

✅ Une version **PDF**  
✅ Une version **ZIP**  
✅ Une version **HTML** prête à imprimer  

Tu veux quel format supplémentaire ?
```
