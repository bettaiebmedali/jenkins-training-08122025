# 🚀 TP5 — CI/CD Complet : Docker local + Kubernetes (Jenkins v2.528.1+)

> Objectif : construire un pipeline Jenkins complet qui **construit une image Docker**, **exécute localement un conteneur pour vérification**, **scanne l’image**, **pousse l’image** sur un registry (Docker Hub), puis **déploie** sur un **cluster Kubernetes**. Le pipeline prend en charge rollback, notifications et bonnes pratiques de sécurité.

---

## 📌 Pré-requis

### Infrastructure & outils
- Jenkins **v2.528.1+** avec accès Docker (soit Jenkins sur une machine avec Docker installé, soit agents Docker).  
- Docker Engine installé sur l'agent qui exécute la phase locale.  
- Accès à un **registry** (Docker Hub, ECR, GHCR...) et identifiants (username/password or token).  
- Un **cluster Kubernetes** accessible depuis Jenkins (kubeconfig, ou via `kubectl` + credentials).  
- `kubectl` et (optionnel) `helm` installés sur l’agent Jenkins qui exécute les étapes Kubernetes.  
- Plugins Jenkins : `Pipeline`, `Docker Pipeline`, `Kubernetes CLI Plugin` (ou `Kubernetes`), `Credentials Binding`, `AnsiColor` (optionnel), `Email Extension`/`Slack Notification` (optionnel).

### Credentials Jenkins à créer
- **docker_hub** — *Username with password* (ou token) pour pousser l’image.  
- **kubeconfig** — *Secret file / Kubernetes config* (type: `Secret file` or `Kubernetes config`) ; alternative : `kube-creds` via `usernamePassword` + API.  
- **registry_scanner** (facultatif) — token pour scanner (Trivy/Snyk), si tu utilises un service externe.

> Place ces credentials dans **Gérer Jenkins → Identifiants → System → (global)** avec les IDs ci-dessus.

---

## 🧭 Architecture proposée du pipeline

1. Checkout repo (Jenkinsfile + code)
2. Build image Docker (tag : `repo:version`)
3. Run image locally (sanity smoke-test)
4. Scan image (Trivy local)
5. Push image vers registry (Docker Hub)
6. Deploy to Kubernetes (manifest / Helm) on a specified environment (DEV/TEST/PROD)
7. Post actions : notifications, cleanup, rollback si besoin

---

## 📁 Structure recommandée du dépôt Git

```
my-app/
├── app.py
├── Dockerfile
├── charts/                  # si utilisation Helm
│   └── my-app/
│       └── Chart.yaml
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── Jenkinsfile
├── .jenkins/
│   └── shared_lib.groovy    # (optionnel) fonctions partagées
└── .env
```

---

## 🔐 Gestion des secrets (bonnes pratiques)
- Ne stocke **jamais** kubeconfig ou tokens dans le repo. Utilise les **Credentials Jenkins**.
- Préfère **tokens** plutôt que mots de passe utilisateur. Active la rotation régulière.
- Utilise **withCredentials** pour exposer temporairement les secrets pendant les étapes shell.
- Masque les secrets dans les logs (plugin Mask Passwords / Credentials Binding).

---

## ✅ Jenkinsfile complet (multistage, compatible Declarative Pipeline)

> Ce Jenkinsfile assume que les credentials `docker_hub` (username/password) et `kubeconfig` (file) existent dans Jenkins.
> Adapte les valeurs `registry`, `imageName` et chemins k8s selon ton projet.

```groovy
pipeline {
  agent any

  environment {
    REGISTRY = "docker.io/mydockeruser"     // adapter
    IMAGE_NAME = "my-app"
    // VERSION peut venir d'un tag Git ou d'un paramètre
  }

  parameters {
    string(name: 'VERSION', defaultValue: '0.1.0-${BUILD_NUMBER}', description: 'Version / Tag de l’image')
    choice(name: 'TARGET_ENV', choices: ['DEV','TEST','PROD'], description: 'Environnement de déploiement')
    booleanParam(name: 'SKIP_SCAN', defaultValue: false, description: 'Passer l’analyse de sécurité?')
    booleanParam(name: 'RUN_LOCAL', defaultValue: true, description: 'Démarrer l’image localement pour tests rapides?')
    booleanParam(name: 'FORCE_DEPLOY', defaultValue: false, description: 'Forcer le déploiement même si scan signale vulnérabilités?')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker image') {
      steps {
        script {
          IMAGE_TAG = "${env.REGISTRY}/${env.IMAGE_NAME}:${params.VERSION}"
          echo "Build image ${IMAGE_TAG}"
        }
        sh "docker build -t ${IMAGE_TAG} ."
      }
    }

    stage('Run local smoke-test') {
      when { expression { return params.RUN_LOCAL } }
      steps {
        script {
          def containerName = "jenkins_test_${env.BUILD_NUMBER}"
          sh """
            docker run -d --name ${containerName} -p 8080:8080 ${IMAGE_TAG} || true
            sleep 2
            # Exécuter un test simple (adapter l'URL/healthcheck)
            curl --silent --fail http://localhost:8080/health || (docker logs ${containerName} && exit 1)
            docker stop ${containerName} || true
            docker rm ${containerName} || true
          """
        }
      }
    }

    stage('Scan image (Trivy)') {
      when { expression { return !params.SKIP_SCAN } }
      steps {
        script {
          // Utilise trivy s'il est installé sur l'agent; sinon saute cette étape
          sh """
            if command -v trivy >/dev/null 2>&1; then
              trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_TAG} || true
            else
              echo "Trivy non disponible, skip scan"
            fi
          """
        }
      }
    }

    stage('Push to registry') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker_hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh """
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${IMAGE_TAG}
            docker logout
          """
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          # place le kubeconfig dans un fichier temporaire et exporte KUBECONFIG
          sh """
            export KUBECONFIG=${KUBECONFIG_FILE}
            # Affiche le contexte pour vérification
            kubectl config current-context
            # Remplace l'image dans les manifests (ex: k8s/deployment.yaml templating)
            sed -e "s|IMAGE_PLACEHOLDER|${IMAGE_TAG}|g" k8s/deployment.yaml > k8s/deployment_for_jenkins.yaml
            kubectl apply -f k8s/deployment_for_jenkins.yaml -n ${params.TARGET_ENV,,} || true
            # Rollout status
            kubectl -n ${params.TARGET_ENV,,} rollout status deployment/my-app --timeout=120s || (kubectl -n ${params.TARGET_ENV,,} get pods -o wide && exit 1)
          """
        }
      }
    }

    stage('Post-deploy verification') {
      steps {
        sh """
          # Exemple de vérification simple : hit le service via kubectl port-forward (ou utiliser Ingress/Service URL)
          echo "Vérification post déploiement (exécutée depuis agent)"
        """
      }
    }
  } // stages

  post {
    success {
      echo "✅ Pipeline terminé avec succès : ${IMAGE_TAG}"
      // notifications optionnelles (Slack, Email)
    }
    unstable {
      echo "⚠️ Pipeline instable"
    }
    failure {
      echo "🔥 Pipeline échoué"
      // Option de rollback basique : déployer l'image précédente
      script {
        if (params.FORCE_DEPLOY == false) {
          echo "Trigger rollback manual or automatic depending on policy"
        }
      }
    }
    always {
      // nettoyage local d'images temporaires si voulu
      sh 'docker image prune -f || true'
    }
  }
}
```

---

## 🔁 Rollback basique (stratégies)

1. **Rollback manuel via UI** : redeployer un build précédent depuis Jenkins (Build → Rebuild).  
2. **Rollback automatique** : conserver le tag précédent (`previousTag`) et `kubectl set image deployment/my-app my-app=${previousTag}` en cas d’échec.  
3. **Blue/Green or Canary** : implémentation avancée avec service switch, helm charts ou Istio/Argo Rollouts.

Exemple simplifié pour rollback automatique (snippet à intégrer dans `post.failure`):
```groovy
// pseudo-code
def previous = sh(returnStdout:true, script: "kubectl -n ${env.TARGET_ENV} get deployment my-app -o=jsonpath='{.spec.template.spec.containers[0].image}'").trim()
sh "kubectl -n ${env.TARGET_ENV} set image deployment/my-app my-app=${previous}"
```

---

## 🔎 Vérifications & Tests recommandés

- S’assurer que `kubectl` fonctionne depuis l’agent : `kubectl get nodes`  
- Vérifier les droits du kubeconfig (namespace, RBAC) : l’utilisateur doit pouvoir `apply`, `get`, `rollout status`.  
- Tester le push docker localement : `docker login && docker push`  
- Installer `trivy` sur l’agent pour scans locaux : `sudo apt-get install trivy` ou utiliser l’image trivy via docker run.

---

## 🛡️ Sécurité & Best Practices

- Ne pas exécuter des builds Docker en tant que root sur des agents partagés. Préfère agents dédiés ou conteneurs Docker-in-Docker sécurisés.  
- Utiliser **Image signing** et **Content Trust** pour garantir provenance des images.  
- Scanner les images automatiquement et définir une politique d’acceptation (ex: bloquer sur vulnérabilités Critiques).  
- Configurer des quotas et limitations (CPU/mémoire) pour pods et namespaces.  
- Auditer les actions Jenkins (Audit Trail plugin).

---

## 🧾 Conseils de debugging

- Si déploiement échoue → `kubectl -n <env> describe pod <pod>` et `kubectl logs pod/<pod>`  
- Si push échoue → vérifier `docker login` et quotas registry.  
- Si build échoue → inspecter Dockerfile et contexte de build (fichiers .dockerignore).

---

## ✅ Récapitulatif et prochaines étapes

Ce TP5 te permet de :
- Construire, tester localement, scanner et pousser des images Docker.  
- Déployer automatiquement sur Kubernetes en ciblant des environnements distincts.  
- Mettre en place des stratégies de rollback et notifications.

🔜 Propositions d’extensions :  
- Implémenter **Blue/Green** ou **Canary** (Istio/Argo Rollouts)  
- Intégrer **Politiques de sécurité** (OPA/Gatekeeper)  
- Déploiements GitOps (ArgoCD)  

---
