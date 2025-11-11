Super — je vais te fournir un TP complet et détaillé (format Markdown dans le chat) qui :

construit une image Docker de ton appli,

exécute des tests unitaires,

lance un scan SonarQube (dans Docker) et vérifie la Quality Gate,

pousse l’image vers un registry privé,

déploie via Ansible des containers (simulation d’un vrai déploiement) sur trois environnements : dev, qualif, prod (chaque environnement représenté par un(s) hôte(s) ou des containers distants),

effectue des tests post-déploiement (smoke / intégration),

gère rollback, logs, sécurité et bonnes pratiques — avec explications complètes à chaque étape.

Copie-colle tout ce qui suit dans un fichier TP6_Ansible_Docker_Sonar_Jenkins.md ou utilise-le tel quel.

# TP6 bis — Jenkins + Docker + Ansible + SonarQube : déploiement réel simulé (dev / qualif / prod)

> Niveau : intermédiaire → avancé  
> Compatible : Jenkins **v2.528.1+**  
> But : pipeline CI/CD complet avec build, tests, SonarQube scan, push vers registry privé, déploiement via Ansible sur environnements dev/qualif/prod (chaque environnement représenté par une machine/VM/container accessible en SSH), tests post-déploiement et rollback.

---

## Table des matières
1. Vue d’ensemble & architecture  
2. Pré-requis (infrastructure & outils)  
3. Credentials Jenkins nécessaires  
4. Structure du dépôt & fichiers fournis  
5. Dockerfile & application d’exemple  
6. SonarQube (mode Docker) — installation & configuration locale  
7. Playbooks Ansible (déploiement par environnement)  
8. Inventory Ansible (dev / qualif / prod)  
9. Jenkinsfile complet (stages : checkout, build, test, sonar, push, deploy, verify)  
10. Tests post-déploiement (smoke / intégration)  
11. Rollback & stratégies  
12. Sécurité, bonnes pratiques, troubleshooting  
13. Extensions possibles

---

## 1) Vue d’ensemble & architecture

- Jenkins (master/agents) exécute le pipeline.  
- SonarQube (server) tourne en Docker (local ou central). Le pipeline déclenche l'analyse Sonar via Scanner.  
- Registry privé (ex. `registry.local:5000`) reçoit les images.  
- Trois environnements de déploiement : **dev**, **qualif**, **prod** — chacun est une ou plusieurs machines (ou containers VM) accessibles via SSH. Ansible orchestre `docker pull` + `docker run` sur ces hôtes.  
- Pipeline : build → tests → sonar (quality gate) → push → deploy dev → tests → promotion → deploy qualif → tests → (manuelle/automatique selon policy) → deploy prod.

Schéma logique :


Git repo -> Jenkins -> build/test -> SonarQube -> push image -> Ansible -> dev/qualif/prod


---

## 2) Pré-requis (infrastructure & outils)

### Serveurs / services
- Jenkins (v2.528.1+) avec agents (ou master capable de docker + ansible).  
- Registre privé accessible (ex : `registry.local:5000`) — peut être un simple Registry Docker.  
- Hôtes cibles pour dev / qualif / prod (ex : `dev-server`, `qualif-server`, `prod-server`) avec SSH et Docker installés.  
- SonarQube server (Docker) accessible depuis Jenkins.

### Logiciels & plugins Jenkins requis
- Docker (engine) sur agent d’exécution.  
- Ansible (soit installé sur l'agent, soit exécuté via container).  
- Plugin: Pipeline, Docker Pipeline, SSH Agent, Credentials Binding, Ansible (optionnel).  
- Sonar Scanner CLI installé sur agent (ou utiliser `sonarscanner` docker image).

### Commandes utiles à avoir
- `docker`, `docker-compose` (pour tests locaux), `ansible`, `ansible-playbook`, `ssh`, `curl`, `jq` (pour parser JSON).

---

## 3) Credentials Jenkins nécessaires

Ajoute dans **Gérer Jenkins → Identifiants → System → (global)** :

- `docker_registry_creds` — **Username / Password** du registry privé.  
- `ssh_deploy_key` — **SSH Username with private key** (user : `deploy`) pour les hôtes dev/qualif/prod.  
- `sonar_token` — **Secret text** : token d’accès à SonarQube (pour le projet).  
- (Optionnel) `git_creds` — si repo Git privé.  
- (Optionnel) `slack_webhook` ou `email_creds` pour notifications.

> **Explication** : utiliser des IDs cohérents permet d'appeler les credentials via `withCredentials` ou `sshagent` dans le Jenkinsfile.

---

## 4) Structure recommandée du dépôt



my-app/
├── app.py
├── requirements.txt
├── Dockerfile
├── docker-compose.yml # pour tests locaux
├── jenkins/
│ └── Jenkinsfile
├── ansible/
│ ├── inventory.ini
│ ├── playbook-deploy.yml
│ ├── roles/
│ │ └── deploy_app/ # facultatif (role)
│ └── group_vars/
│ ├── dev.yml
│ ├── qualif.yml
│ └── prod.yml
├── sonar-project.properties
└── tests/
└── test_health.sh


---

## 5) Dockerfile & example app (Flask)

**Dockerfile**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV FLASK_APP=app.py
EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]


requirements.txt

Flask==2.2.5
gunicorn==20.1.0
requests==2.31.0


app.py (endpoints / et /health)

from flask import Flask, jsonify
app = Flask(__name__)

@app.route('/health')
def health():
    return jsonify(status="OK"), 200

@app.route('/')
def root():
    return "Hello from my-app", 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)


tests/test_health.sh (script simple)

#!/bin/bash
set -e
curl --fail http://localhost:8080/health | grep -q OK
echo "Health ok"

6) SonarQube (mode Docker) — installation locale rapide

docker-compose-sonar.yml (exemple)

version: '3.7'
services:
  db:
    image: postgres:13
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonar
  sonarqube:
    image: sonarqube:9.9-community
    depends_on:
      - db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
    ports:
      - "9000:9000"


Démarrer :

docker compose -f docker-compose-sonar.yml up -d


Crée ton projet et récupère le token (utilisé comme sonar_token).

Le pipeline exécutera sonar-scanner ou mvn sonar:sonar selon stack.

7) Playbook Ansible (ansible/playbook-deploy.yml)

Objectif : sur chaque host, pull l'image depuis registry privé, arrêter le container existant, le remplacer par le nouveau tag, vérifier /health.

---
- name: Deploy application container from private registry
  hosts: app_servers
  become: true
  vars:
    registry: "registry.local:5000"
    image_name: "my-app"
    container_name: "my-app"
    port: 8080
    version: "latest"   # remplacé via --extra-vars
  tasks:
    - name: Ensure docker is installed (Debian/Ubuntu)
      ansible.builtin.apt:
        name: docker.io
        state: present
      become: true
      when: ansible_facts['os_family'] == 'Debian'

    - name: Login to private registry (use creds passed via environment / config if needed)
      ansible.builtin.shell: |
        docker login registry.local:5000 -u "{{ registry_user }}" -p "{{ registry_pass }}"
      args:
        warn: false
      no_log: true
      when: registry_user is defined and registry_pass is defined

    - name: Pull image
      ansible.builtin.shell: "docker pull {{ registry }}/{{ image_name }}:{{ version }}"
      register: pull_result
      failed_when: pull_result.rc != 0

    - name: Stop existing container if running
      ansible.builtin.shell: "docker ps -q -f name={{ container_name }} | xargs --no-run-if-empty docker stop || true"
      ignore_errors: true

    - name: Remove existing container if present
      ansible.builtin.shell: "docker rm -f {{ container_name }} || true"
      ignore_errors: true

    - name: Run new container
      ansible.builtin.shell: >
        docker run -d --name {{ container_name }} -p {{ port }}:{{ port }}
        --restart unless-stopped {{ registry }}/{{ image_name }}:{{ version }}
      args:
        executable: /bin/bash

    - name: Wait for health endpoint
      ansible.builtin.uri:
        url: "http://localhost:{{ port }}/health"
        status_code: 200
        timeout: 5
      register: health_check
      retries: 6
      delay: 5
      until: health_check.status == 200


Explication :

registry_user et registry_pass peuvent être fournis par Jenkins (via --extra-vars ou via group_vars et sshagent), mais attention : ne pas exposer en clair ; utiliser ansible-vault ou variables d’environnement (ANSIBLE_REMOTE_USER non recommandé pour secrets).

no_log: true sur login empêche l'affichage des credentials.

8) Inventory et group_vars

ansible/inventory.ini

[dev]
dev-server ansible_host=192.0.2.11 ansible_user=deploy

[qualif]
qualif-server ansible_host=192.0.2.12 ansible_user=deploy

[prod]
prod-server ansible_host=192.0.2.13 ansible_user=deploy

[app_servers:children]
dev
qualif
prod


ansible/group_vars/dev.yml

registry_user: "jenkins"
registry_pass: "PLACEHOLDER"   # mieux : passer depuis Jenkins via --extra-vars ou ansible-vault


Important : ne mets pas de secrets dans repo. Utilise Jenkins pour injecter via --extra-vars ou ansible-vault chiffré.

9) Jenkinsfile complet (déclarative) — pipeline multi-étapes et multi-environnements

Explication globale avant le Jenkinsfile :

Le pipeline effectue : Checkout → Build → Tests unitaires → SonarQube scan (quality gate) → Build image → Push → Deploy sur dev → tests → si ok, promotion (manuelle ou automatique) vers qualif puis prod.

Les credentials sont repris via withCredentials et sshagent.

Le pipeline arrête si la Quality Gate Sonar échoue (fail build).

Les tests post-déploiement sont exécutés via Ansible (ou via SSH/curl).

Jenkinsfile

pipeline {
  agent any

  environment {
    REGISTRY = "registry.local:5000"
    IMAGE_NAME = "my-app"
    SONAR_HOST = "http://sonarqube:9000"    // adapter
    // Les tokens/creds utilisés via withCredentials
  }

  parameters {
    string(name: 'VERSION', defaultValue: "0.1.0-${env.BUILD_NUMBER}", description: 'Tag image')
    booleanParam(name: 'PROMOTE_AUTOMATICALLY', defaultValue: false, description: 'Promote automatically if tests OK')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Unit Tests') {
      steps {
        sh 'pytest -q || true'   // ou adapter selon ton framework
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withCredentials([string(credentialsId: 'sonar_token', variable: 'SONAR_TOKEN')]) {
          sh """
            # utilise sonar-scanner CLI (doit être installé sur agent ou via docker)
            if command -v sonar-scanner >/dev/null 2>&1; then
              sonar-scanner \
                -Dsonar.projectKey=${env.JOB_NAME} \
                -Dsonar.sources=. \
                -Dsonar.host.url=${env.SONAR_HOST} \
                -Dsonar.login=$SONAR_TOKEN
            else
              echo "sonar-scanner non installé; exécution via docker"
              docker run --rm -v "${pwd()}":/usr/src sonarsource/sonar-scanner-cli \
                -Dsonar.projectKey=${env.JOB_NAME} \
                -Dsonar.sources=/usr/src \
                -Dsonar.host.url=${env.SONAR_HOST} \
                -Dsonar.login=$SONAR_TOKEN
            fi
          """
        }
      }
    }

    stage('Quality Gate') {
      steps {
        // Vérifier la Quality Gate via API Sonar
        script {
          withCredentials([string(credentialsId: 'sonar_token', variable: 'SONAR_TOKEN')]) {
            def projectKey = env.JOB_NAME
            // attente du serveur Sonar pour finaliser l'analyse
            timeout(time: 2, unit: 'MINUTES') {
              waitUntil {
                def qgStatus = sh(returnStdout: true, script: """
                  curl -s -u $SONAR_TOKEN: ${env.SONAR_HOST}/api/qualitygates/project_status?projectKey=${projectKey} | jq -r .projectStatus.status
                """).trim()
                echo "Quality Gate status: ${qgStatus}"
                return (qgStatus == 'OK' || qgStatus == 'ERROR' || qgStatus == 'WARN')
              }
            }
            // Récupérer le status final
            def status = sh(returnStdout: true, script: "curl -s -u $SONAR_TOKEN: ${env.SONAR_HOST}/api/qualitygates/project_status?projectKey=${projectKey} | jq -r .projectStatus.status").trim()
            if (status != 'OK') {
              error "Quality Gate failed: ${status}"
            } else {
              echo "Quality Gate OK"
            }
          }
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          IMAGE_TAG = "${env.REGISTRY}/${env.IMAGE_NAME}:${params.VERSION}"
        }
        sh "docker build -t ${IMAGE_TAG} ."
      }
    }

    stage('Push Image to Registry') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker_registry_creds', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
          sh """
            echo "$REG_PASS" | docker login registry.local:5000 -u "$REG_USER" --password-stdin
            docker push ${IMAGE_TAG}
            docker logout registry.local:5000 || true
          """
        }
      }
    }

    stage('Deploy to DEV') {
      steps {
        script {
          env.EXTRA_VARS = "version=${params.VERSION} registry_user=${REG_USER} registry_pass=${REG_PASS}"
        }
        sshagent(['ssh_deploy_key']) {
          sh """
            cd ansible
            ansible-playbook -i inventory.ini playbook-deploy.yml --limit dev --extra-vars "version=${params.VERSION}"
          """
        }
      }
    }

    stage('Verify DEV') {
      steps {
        // tests simples via Ansible (uri) ou SSH
        sshagent(['ssh_deploy_key']) {
          sh """
            cd ansible
            ansible -i inventory.ini dev -m uri -a 'url=http://localhost:8080/health status_code=200' --become
          """
        }
      }
    }

    stage('Promote to QUALIF') {
      when { expression { return params.PROMOTE_AUTOMATICALLY } }
      steps {
        sshagent(['ssh_deploy_key']) {
          sh """
            cd ansible
            ansible-playbook -i inventory.ini playbook-deploy.yml --limit qualif --extra-vars "version=${params.VERSION}"
          """
        }
      }
    }

    stage('Verify QUALIF') {
      when { expression { return params.PROMOTE_AUTOMATICALLY } }
      steps {
        sshagent(['ssh_deploy_key']) {
          sh """
            cd ansible
            ansible -i inventory.ini qualif -m uri -a 'url=http://localhost:8080/health status_code=200' --become
          """
        }
      }
    }

    stage('Manual Approval for PROD') {
      when { expression { return params.PROMOTE_AUTOMATICALLY == false } }
      steps {
        input message: "Valider le déploiement en PROD ?"
      }
    }

    stage('Deploy to PROD') {
      steps {
        sshagent(['ssh_deploy_key']) {
          sh """
            cd ansible
            ansible-playbook -i inventory.ini playbook-deploy.yml --limit prod --extra-vars "version=${params.VERSION}"
          """
        }
      }
    }

    stage('Verify PROD') {
      steps {
        sshagent(['ssh_deploy_key']) {
          sh """
            cd ansible
            ansible -i inventory.ini prod -m uri -a 'url=http://localhost:8080/health status_code=200' --become
          """
        }
      }
    }
  } // stages

  post {
    success {
      echo "Pipeline terminé : ${IMAGE_TAG}"
      // envoyer notifications si désiré
    }
    failure {
      echo "Pipeline échoué : vérifiez logs et rollback si nécessaire"
      // Optionnel: lancer playbook rollback automatique
    }
    always {
      sh 'docker image prune -f || true'
    }
  }
}


Remarques & explications :

sonar-scanner peut être exécuté en docker si pas installé sur l'agent.

Le Quality Gate est vérifié via l'API Sonar; jq est utilisé pour parser JSON — installe jq sur ton agent.

Utilise --limit pour cibler l’environnement (dev/qualif/prod) dans Ansible.

sshagent injecte la clé privée pour les connexions SSH durant l’exécution d’Ansible.

10) Tests post-déploiement

Smoke test : endpoint /health ou page home. Implémenté via Ansible uri ou via script curl exécuté sur la machine distante.

Tests d'intégration : appeler d'autres services, exécuter scripts de test depuis Jenkins (si accès réseau), ou via Ansible exécuter des scripts sur la cible.

Automatisation : si tests échouent → rollback (manuel ou automatique).

11) Rollback & stratégies
Option 1 — rollback manuel

Relancer le pipeline en spécifiant VERSION = tag précédent (par ex. 1.2.3) puis déployer.

Option 2 — rollback automatique (basic)

Dans post.failure du Jenkinsfile, exécuter un playbook Ansible playbook-rollback.yml :

récupérer previous_tag : soit via registry API, soit stocké dans un fichier last_successful_tag.txt sur Jenkins/Artefacts.

lancer docker run avec previous_tag.

Exemple minimal (inside post.failure):

post {
  failure {
    sshagent(['ssh_deploy_key']) {
      sh """
        cd ansible
        ansible-playbook -i inventory.ini playbook-rollback.yml --extra-vars "rollback_tag=${PREVIOUS_TAG}"
      """
    }
  }
}

Option 3 — Blue/Green ou Canary (préféré en prod)

Mettre en place deux sets de services (blue/green) et router le trafic via loadbalancer/ingress; switch instantané quand green OK.

12) Sécurité & bonnes pratiques

Ne pas stocker registry_pass / tokens en clair : utilise Jenkins Credentials.

No_log dans Ansible pour masquer les tâches sensibles.

Restreindre SSH (firewall) aux IP de Jenkins.

Utiliser tags immuables (versionnés).

Scanner les images (Trivy) avant push — ici Sonar est pour la qualité du code, pas vulnérabilités d’OS/packages.

Séparer les environnements réseau/sécurité (dev ≠ prod).

Monitorer logs & mettre en place alerting.

13) Debugging courant

Sonar : si Quality Gate non retourné, vérifier si Sonar a fini l'analyse / token correct / URL correcte.

Docker push : vérifier docker login & certificats (si registry TLS) ; pour registry privé avec certificat self-signed, configure CA ou --insecure selon besoin.

Ansible SSH : tester ssh -i key deploy@host depuis agent Jenkins.

Health checks : consulter docker logs du container sur la cible.

Timeouts : augmenter les retries/delays dans playbook si démarrage de l'app long.

14) Fichiers d’exemple supplémentaires (optionnel)

Tu peux ajouter dans le repo :

docker-compose.yml pour exécuter app + Sonar localement pour tests.

playbook-rollback.yml pour stratégie de rollback automatique.

roles/deploy_app/tasks/main.yml pour modulariser le déploiement.

15) Checklist avant exécution

Jenkins a accès à Docker & Ansible.

SonarQube accessible et sonar_token configuré.

Registry privé en place et docker_registry_creds créés.

Clé SSH ajoutée au authorized_keys des cibles et dans Jenkins (ssh_deploy_key).

Inventory Ansible correct et variables d’environnement configurées.

Tests unitaires passent localement.

Lancer pipeline sur branch de test, vérifier chaque étape.

Conclusion

Ce TP simule un déploiement « réel » multi-environnements en utilisant des containers Docker, orchestrés par Ansible (qui effectue les opérations sur hôtes distants via SSH), avec intégration SonarQube pour la qualité du code, et Jenkins comme orchestrateur CI/CD.
Il couvre build, test, quality gate, push, déploiement, vérification et rollback.

Si tu veux la suite, je peux aussi :

te fournir un playbook de rollback complet ;

te donner un docker-compose incluant SonarQube + Postgres + registry pour tester en local ;

générer un repository zip contenant tous les fichiers d’exemple (Dockerfile, app.py, Jenkinsfile, ansible/...).

Dis-moi ce que tu veux en plus : rollback, docker-compose, repo zip ou autre.


---

Souhaites-tu que je te génère aussi :  
- le `playbook-rollback.yml` prêt à l’emploi,  
- **ou** un `docker-compose.yml` pour monter SonarQube + registry local + app en test,  
- ou que je transforme tout ça en un fichier `.md` téléchargeable (je peux te fournir le contenu comme ci-d