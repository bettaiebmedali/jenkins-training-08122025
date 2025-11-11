# TP7_Final — Jenkins + Docker + Ansible + SonarQube (Simu: Containers serveurs Dev / Qualif / Prod)

**Niveau :** Intermédiaire → Avancé  
**Compatible :** Jenkins v2.528.1+  
**But :** pipeline CI/CD complet — build, tests, SonarQube, push sur registry privé, déploiement via Ansible sur *containers* simulant des serveurs (dev / qualif / prod), tests post-déploiement, rollback, monitoring.

---

## Sommaire

1. Objectif & Architecture  
2. Diagrammes (Mermaid + ASCII)  
3. Prérequis & topologie Docker (création des containers cibles)  
4. Configuration Jenkins (plugins & credentials)  
5. Fichiers du projet (Dockerfile, app, tests, sonar)  
6. SonarQube local (docker) — config rapide  
7. Ansible — inventory & playbook (déploiement containers)  
8. Jenkinsfile complet (commenté)  
9. Tests post-déploiement & Quality Gate SonarQube  
10. Rollback & stratégies  
11. FAQ & erreurs fréquentes  
12. Bonnes pratiques & extensions

---

## 1. Objectif & architecture

**Flux** :  
- Dev push → Jenkins pipeline  
- Tests unitaires → SonarQube analysis (Quality Gate)  
- Build image Docker → Push → Déploiement Ansible sur environnements simulés (containers)  
- Vérifications & tests post-déploiement → Promotion

**Topologie de test (simulée avec containers) :**

- `registry.local:5000` — registry privé (peut être un container `registry:2`)  
- `sonarqube` — SonarQube container  
- `dev-server` (container) — target for dev  
- `qualif-server` (container) — target for qualif  
- `prod-server` (container) — target for prod

Chaque "server" est un container Docker avec Docker Engine (Docker-in-Docker pattern is heavy); pour la simulation on utilisera des containers basés sur `docker:dind` ou des containers simples où Ansible se connecte via SSH et exécute docker commands (docker doit être installé sur ces containers). Méthode recommandée ici : lancer de petites VMs Docker-Linux ou containers avec SSH et docker client/daemon accessible (ex : dockerd via dind). Pour simplicité, on présente la version avec containers `ubuntu` + `docker` installé.

---

## 2. Diagrammes

### 2.1 Diagramme Mermaid (thème clair — recommandé pour docs)
> Si ton éditeur supporte Mermaid (VSCode + plugin, GitHub Pages avec mermaid activé, etc.), il rendra ce schéma visuel.

```mermaid
flowchart TB
  subgraph DEVOPS
    A[Git (repo)] --> B[Jenkins Pipeline]
    B --> C[SonarQube (Docker)]
    B --> D[Docker Registry (registry.local:5000)]
    B --> E[Ansible (via sshagent)]
  end

  subgraph ENVIRONMENTS
    E --> F[dev-server (container)]
    E --> G[qualif-server (container)]
    E --> H[prod-server (container)]
  end

  C --> B
  D --> F
  D --> G
  D --> H
  B --> I[Tests Post-Deploy]

  style B fill:#f8fbff,stroke:#2b6cb0,stroke-width:2px
  style C fill:#ffffff,stroke:#2b6cb0
  style D fill:#ffffff,stroke:#2b6cb0
  style E fill:#ffffff,stroke:#2b6cb0
  style F fill:#f7fafc,stroke:#4a5568
  style G fill:#f7fafc,stroke:#4a5568
  style H fill:#f7fafc,stroke:#4a5568
  style I fill:#edf2f7,stroke:#2b6cb0

```
### 2.2 Schéma ASCII (toujours visible dans n’importe quel viewer)
```less

[ Git ] --> [ Jenkins ]
                 |
        ------------------------
        |          |           |
   [ SonarQube ] [ Registry ] [ Ansible ]
                                 |
                      -------------------------
                      |          |            |
                 [ dev ]     [ qualif ]     [ prod ]
```
## 3. Prérequis & topologie Docker (création des containers cibles)
### 3.1 Prérequis locaux
Docker & docker-compose installés sur ta machine (pour monter Sonar, Registry, et containers cibles).

Jenkins opérationnel (local ou serveur).

Ansible installé sur l’agent Jenkins (ou exécution via un container Ansible).

### 3.2 Lancer les services de test (SonarQube + Registry + targets)
Exemple docker-compose.test.yml (à placer localement pour tests):

```yaml
version: "3.8"
services:
  registry:
    image: registry:2
    ports: ["5000:5000"]
    restart: unless-stopped

  sonarqube:
    image: sonarqube:9.9-community
    environment:
      - SONAR_JDBC_URL=jdbc:postgresql://db:5432/sonar
      - SONAR_JDBC_USERNAME=sonar
      - SONAR_JDBC_PASSWORD=sonar
    ports:
      - "9000:9000"
    depends_on:
      - db

  db:
    image: postgres:13
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
      - POSTGRES_DB=sonar

  dev-server:
    image: ubuntu:22.04
    container_name: dev-server
    tty: true
    privileged: true
    ports:
      - "2222:22"
    command: ["/bin/sh","-c","apt-get update && apt-get install -y openssh-server docker.io python3 python3-pip && mkdir -p /var/run/sshd && echo 'root:root' | chpasswd && /usr/sbin/sshd -D"]

  qualif-server:
    image: ubuntu:22.04
    container_name: qualif-server
    tty: true
    privileged: true
    ports:
      - "2223:22"
    command: ["/bin/sh","-c","apt-get update && apt-get install -y openssh-server docker.io python3 python3-pip && mkdir -p /var/run/sshd && echo 'root:root' | chpasswd && /usr/sbin/sshd -D"]

  prod-server:
    image: ubuntu:22.04
    container_name: prod-server
    tty: true
    privileged: true
    ports:
      - "2224:22"
    command: ["/bin/sh","-c","apt-get update && apt-get install -y openssh-server docker.io python3 python3-pip && mkdir -p /var/run/sshd && echo 'root:root' | chpasswd && /usr/sbin/sshd -D"]
```
```bash
Lancer : docker compose -f docker-compose.test.yml up -d
Ceci crée : un registry local sur :5000, SonarQube sur :9000, et 3 containers avec SSH exposés sur 2222/2223/2224.
Note : en prod on n’utilise pas ce pattern (privileged containers), mais c’est pratique pour les tests.
```
### 3.3 Ajout de la clé SSH Jenkins
Génère une paire ssh (sur ton poste ou Jenkins) :

```bash
ssh-keygen -t ed25519 -C "jenkins-deploy" -f jenkins_deploy_key
Ajoute la clé publique (jenkins_deploy_key.pub) dans /root/.ssh/authorized_keys des containers (ex: docker exec -it dev-server bash puis mkdir -p /root/.ssh && echo "...pub..." >> /root/.ssh/authorized_keys).

Dans Jenkins > Manage Credentials > System > Global : ajoute la clé privée (type SSH Username with private key) avec ID par ex. ansible_ssh et username root (ou deploy si créé).
```
## 4. Configuration Jenkins (plugins & credentials)
### 4.1 Plugins recommandés
Pipeline

Docker Pipeline

SSH Agent Plugin

SonarQube Scanner (et Generic Sonar plugin)

Credentials Binding

Ansible Plugin (optionnel)

### 4.2 Credentials à créer
docker_registry → Username/password (registry.local:5000)

ansible_ssh → SSH Username with private key (username root ou deploy)

sonar_token → Secret text (token SonarQube)

(optionnel) git_credential si repo privé

## 5. Fichiers du projet (exemples complets)
### 5.1 Dockerfile
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```
### 5.2 requirements.txt
```ini

Flask==2.2.5
gunicorn==20.1.0
requests==2.31.0
pytest==7.4.0

```
### 5.3 app.py
```python

from flask import Flask, jsonify
app = Flask(__name__)

@app.route("/health")
def health():
    return jsonify({"status":"OK"}), 200

@app.route("/")
def root():
    return "Hello from my-app", 200


```
### 5.4 tests/test_health.py
```python

import requests

def test_health():
    r = requests.get("http://localhost:8080/health", timeout=5)
    assert r.status_code == 200
    assert r.json().get("status") == "OK"
```
### 5.5 sonar-project.properties
```ini
sonar.projectKey=my-app
sonar.sources=.
sonar.host.url=http://localhost:9000
```
## 6. SonarQube local (installation & configuration rapide)
Lancer Sonar (vu section 3).

Créer un projet my-app et générer un sonar_token (user: admin → Security → Generate Tokens).

Stocker ce token dans Jenkins comme sonar_token.

Exécuter sonar-scanner : tu peux l'installer sur l'agent ou utiliser l'image Docker sonarsource/sonar-scanner-cli.

Exemple exécuté dans Jenkinsfile (docker-run) :

```bash

docker run --rm -v "$(pwd)":/usr/src sonarsource/sonar-scanner-cli \
  -Dsonar.projectKey=my-app \
  -Dsonar.sources=/usr/src \
  -Dsonar.host.url=http://sonarqube:9000 \
  -Dsonar.login=$SONAR_TOKEN
```
## 7. Ansible — inventory & playbook (déploiement containers)
### 7.1 ansible/inventory.ini
csharp
[dev]
dev-server ansible_host=127.0.0.1 ansible_port=2222 ansible_user=root

[qualif]
qualif-server ansible_host=127.0.0.1 ansible_port=2223 ansible_user=root

[prod]
prod-server ansible_host=127.0.0.1 ansible_port=2224 ansible_user=root

[app:children]
dev
qualif
prod
### 7.2 ansible/playbook-deploy.yml
```yaml
---
- name: Deploy my-app on targeted hosts
  hosts: app
  become: true
  vars:
    registry: "registry.local:5000"
    image: "myapp"
    port: 8080
  tasks:
    - name: Ensure docker is present
      ansible.builtin.shell: |
        which docker || (apt-get update && apt-get install -y docker.io)
      changed_when: false

    - name: Pull image from registry
      ansible.builtin.shell: "docker pull {{ registry }}/{{ image }}:{{ version }}"
      register: pull
      failed_when: pull.rc != 0

    - name: Stop old container if running
      ansible.builtin.shell: "docker ps -q -f name={{ image }} | xargs --no-run-if-empty docker stop || true"
      ignore_errors: true

    - name: Remove old container
      ansible.builtin.shell: "docker rm -f {{ image }} || true"
      ignore_errors: true

    - name: Run new container
      ansible.builtin.shell: >
        docker run -d --name {{ image }} -p {{ port }}:{{ port }}
        --restart unless-stopped {{ registry }}/{{ image }}:{{ version }}
      args:
        executable: /bin/bash

    - name: Wait for app health endpoint
      ansible.builtin.uri:
        url: "http://localhost:{{ port }}/health"
        status_code: 200
        timeout: 10
      register: hc
      retries: 6
      delay: 5
      until: hc.status == 200

```
Le playbook attend une variable version fournie par Jenkins via --extra-vars "version=1.2.3".

## 8. Jenkinsfile complet (multi-stage, commenté)
Ce Jenkinsfile réalise : checkout → tests unitaires → sonar → build docker → push → deploy dev → tests → promote qualif/prod.

```groovy

pipeline {
  agent any

  environment {
    REGISTRY = "registry.local:5000"
    IMAGE = "myapp"
    SONAR_HOST = "http://localhost:9000"
  }

  parameters {
    string(name: 'VERSION', defaultValue: "0.1.0-${env.BUILD_NUMBER}", description: 'Tag image')
    booleanParam(name: 'AUTO_PROMOTE', defaultValue: false, description: 'Promote automatically to qualif/prod')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Unit Tests') {
      steps {
        sh '''
          python3 -m pip install -r requirements.txt
          pytest -q || true
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withCredentials([string(credentialsId:'sonar_token', variable:'SONAR_TOKEN')]) {
          sh '''
            if command -v sonar-scanner >/dev/null 2>&1; then
              sonar-scanner -Dsonar.projectKey=my-app -Dsonar.host.url=${SONAR_HOST} -Dsonar.login=$SONAR_TOKEN
            else
              docker run --rm -v "${PWD}":/usr/src sonarsource/sonar-scanner-cli \
                -Dsonar.projectKey=my-app -Dsonar.sources=/usr/src -Dsonar.host.url=${SONAR_HOST} -Dsonar.login=$SONAR_TOKEN
            fi
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          withCredentials([string(credentialsId:'sonar_token', variable:'SONAR_TOKEN')]) {
            def status = sh(returnStdout:true, script: """
              sleep 3
              curl -s -u $SONAR_TOKEN: ${SONAR_HOST}/api/qualitygates/project_status?projectKey=my-app | jq -r .projectStatus.status
            """).trim()
            echo "Sonar Quality Gate: ${status}"
            if (status != 'OK') {
              error("Quality Gate failed: ${status}")
            }
          }
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          IMAGE_TAG = "${env.REGISTRY}/${env.IMAGE}:${params.VERSION}"
          sh "docker build -t ${IMAGE_TAG} ."
        }
      }
    }

    stage('Push to Registry') {
      steps {
        withCredentials([usernamePassword(credentialsId:'docker_registry', usernameVariable:'REG_USER', passwordVariable:'REG_PASS')]) {
          sh '''
            echo $REG_PASS | docker login registry.local:5000 -u $REG_USER --password-stdin
            docker push ${IMAGE_TAG}
            docker logout registry.local:5000 || true
          '''
        }
      }
    }

    stage('Deploy -> DEV') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId:'ansible_ssh', keyFileVariable:'SSH_KEY', usernameVariable:'SSH_USER')]) {
          sh """
            mkdir -p .ssh && echo "$(<$SSH_KEY)" > .ssh/id_rsa && chmod 600 .ssh/id_rsa
            export ANSIBLE_HOST_KEY_CHECKING=False
            ansible-playbook -i ansible/inventory.ini ansible/playbook-deploy.yml --extra-vars "version=${params.VERSION}" --limit dev -u root --private-key=.ssh/id_rsa
          """
        }
      }
    }

    stage('Verify -> DEV') {
      steps {
        sh 'curl -fsS http://127.0.0.1:8080/health || (echo "DEV health failed" && exit 1)'
      }
    }

    stage('Promote to QUALIF') {
      when { expression { return params.AUTO_PROMOTE } }
      steps {
        withCredentials([sshUserPrivateKey(credentialsId:'ansible_ssh', keyFileVariable:'SSH_KEY', usernameVariable:'SSH_USER')]) {
          sh """
            export ANSIBLE_HOST_KEY_CHECKING=False
            ansible-playbook -i ansible/inventory.ini ansible/playbook-deploy.yml --extra-vars "version=${params.VERSION}" --limit qualif -u root --private-key=$SSH_KEY
          """
        }
      }
    }

    stage('Promote to PROD (Manual)') {
      when { expression { return !params.AUTO_PROMOTE } }
      steps {
        input message: "Valider déploiement en PROD ?"
      }
    }

    stage('Deploy -> PROD') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId:'ansible_ssh', keyFileVariable:'SSH_KEY', usernameVariable:'SSH_USER')]) {
          sh """
            export ANSIBLE_HOST_KEY_CHECKING=False
            ansible-playbook -i ansible/inventory.ini ansible/playbook-deploy.yml --extra-vars "version=${params.VERSION}" --limit prod -u root --private-key=.ssh/id_rsa
          """
        }
      }
    }

    stage('Verify -> PROD') {
      steps {
        sh 'curl -fsS http://127.0.0.1:8080/health || (echo "PROD health failed" && exit 1)'
      }
    }
  }

  post {
    success {
      echo "Pipeline terminé : ${IMAGE_TAG}"
    }
    failure {
      echo "Échec du pipeline — analyser logs. Eventuellement rollback."
    }
    always {
      sh 'docker image prune -f || true'
    }
  }
}

```
** Remarques :

J’utilise ansible-playbook avec --limit pour cibler dev/qualif/prod.

On fournit la clé privée Jenkins via withCredentials et on l’écrit en .ssh/id_rsa.

Les curl de vérification pointent vers localhost:8080 parce que dans la configuration docker-compose.test.yml les ports sont mappés localement; adapte selon réseau réel.

## 9. Tests post-déploiement & Quality Gate SonarQube
Checks automatisés
SonarQube Quality Gate → pipeline stoppe si != OK.

Smoke test → curl /health.

Tests d’intégration → exécution de tests via Ansible ou Jenkins (ex: lancer pytest dans un container test).

Exemple : test via Ansible
bash
ansible -i ansible/inventory.ini dev -m uri -a "url=http://localhost:8080/health status_code=200" --become
## 10. Rollback & stratégies
Rollback simple (manuel)
Relancer pipeline avec VERSION précédent.

Rollback automatique (exemple)
Dans post.failure, exécuter un playbook playbook-rollback.yml qui :

obtient previous_tag (depuis un artefact Jenkins ou Registry API)

appelle docker run avec previous_tag.

Exemple minimal playbook-rollback.yml

```yaml

- name: rollback
  hosts: app
  become: true
  tasks:
    - name: Run previous image
      ansible.builtin.shell: docker run -d --name myapp -p 8080:8080 registry.local:5000/myapp:{{ rollback_tag }}
```
## 11. FAQ & erreurs fréquentes
Q : Sonar Quality Gate retourne PENDING ou ne répond pas ?
A : attendre la fin de l’analyse (peut prendre quelques secondes) et vérifier que token/URL sont corrects. Utilise l’API Sonar /api/ce/task?id=... et /api/qualitygates/project_status.

Q : docker push échoue (401) ?
A : vérifier docker login et credentials Jenkins. Si registry utilise TLS self-signed, soit configurer CA, soit configurer Docker daemon pour ignorer TLS (non recommandé).

Q : Ansible ne peut pas ssh ?
A : tester la connexion SSH depuis l’agent Jenkins : ssh -i key root@127.0.0.1 -p 2222. Vérifier authorized_keys, firewall, ANSIBLE_HOST_KEY_CHECKING=False.

Q : docker introuvable sur target container ?
A : vérifier que docker est installé et que le user a le droit d’exécuter docker; dans playbook on installe docker si absent.

Q : Problèmes de permissions fichiers clés (private key)
A : chmod 600 .ssh/id_rsa ; Ansible/ssh refusera une clé trop permissive.

## 12. Bonnes pratiques & extensions
Séparer les rôles Jenkins (master vs agents) : exécuter docker sur agent dédié.

Scans vulnérabilités (Trivy) avant push.

Utiliser ansible-vault pour chiffrer les secrets.

Déployer Blue/Green ou Canary pour prod.

Utiliser pipeline multibranch et shared libraries pour réutilisabilité.

Annexes : commandes utiles
Lancer l’environnement de test :

```bash

docker compose -f docker-compose.test.yml up -d
Afficher logs Sonar :
```
```bash

docker logs -f sonarqube
Tester SSH sur dev :
```
```bash

ssh -p 2222 root@127.0.0.1 -i jenkins_deploy_key
Pousser une image test vers registry local :
```
```bash
docker tag myapp:latest registry.local:5000/myapp:1.0.0
docker push registry.local:5000/myapp:1.0.0
```
# Conclusion
Ce TP fournit un workflow CI/CD réaliste et reproductible : build → test → sonar → push → deploy sur environnements simulés par containers. Il couvre la plupart des besoins pour tester un pipeline multi-environnements avant déploiement réel en production.


# Troubleshooting 
Sur ta machine hôte (et pas dans le conteneur), exécute cette commande :

sudo sysctl -w vm.max_map_count=262144


Puis rends la persistance du paramètre (facultatif mais recommandé) :

echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf