# TP7 — Jenkins + Docker + Ansible + SonarQube (Simulation Dev / Qualif / Prod)

## 🎯 Objectif

Mettre en place une infrastructure CI/CD complète simulée avec Docker pour tester un pipeline Jenkins :
- Build et test d’une app (Flask par ex.)
- Analyse SonarQube
- Push sur un registry privé local
- Déploiement automatisé via Ansible sur des containers jouant les rôles de serveurs dev / qualif / prod.

---

## 🧱 Étapes réalisées

### 1️⃣ Stack Docker local

Fichier : `docker-compose.test.yml`

Lance :
- registry.local:5000 → registry privé  
- SonarQube + PostgreSQL  
- 3 serveurs simulés (dev-server, qualif-server, prod-server)

Commande :
```bash
docker compose -f docker-compose.test.yml up -d
```

---

### 2️⃣ Configuration SSH pour Ansible

1. Génération d’une clé :
   ```bash
   ssh-keygen -t ed25519 -C "ansible-deploy" -f ~/.ssh/ansible_deploy_key
   ```

2. Ajout automatique de la clé publique dans les 3 conteneurs :
   ```bash
   PUBKEY=$(cat ~/.ssh/ansible_deploy_key.pub)
   for container in dev-server qualif-server prod-server; do
     docker exec -u root $container bash -c "mkdir -p /root/.ssh && echo '$PUBKEY' >> /root/.ssh/authorized_keys && chmod 700 /root/.ssh && chmod 600 /root/.ssh/authorized_keys"
   done
   ```

3. Test SSH :
   ```bash
   ssh -i ~/.ssh/ansible_deploy_key root@127.0.0.1 -p 2222
   ```

---

### 3️⃣ Inventaire Ansible

`ansible/inventory.ini`
```ini
[dev]
dev-server ansible_host=127.0.0.1 ansible_port=2222 ansible_user=root ansible_ssh_private_key_file=~/.ssh/ansible_deploy_key

[qualif]
qualif-server ansible_host=127.0.0.1 ansible_port=2223 ansible_user=root ansible_ssh_private_key_file=~/.ssh/ansible_deploy_key

[prod]
prod-server ansible_host=127.0.0.1 ansible_port=2224 ansible_user=root ansible_ssh_private_key_file=~/.ssh/ansible_deploy_key

[app:children]
dev
qualif
prod
```

---

### 4️⃣ Test de connexion Ansible

`ansible/ping.yml`
```yaml
---
- name: Test de connexion SSH vers containers
  hosts: all
  gather_facts: no
  tasks:
    - name: Ping
      ansible.builtin.ping:
```

Exécution :
```bash
ansible-playbook -i ansible/inventory.ini ansible/ping.yml
```

---

### 5️⃣ Test de déploiement simple avec Ansible

`ansible/deploy.yml`
```yaml
---
- name: Déploiement simple sur environnements simulés
  hosts: all
  become: true
  tasks:
    - name: Vérifier Docker
      ansible.builtin.command: docker --version
      register: docker_version
      changed_when: false

    - debug:
        msg: "Docker sur {{ inventory_hostname }} : {{ docker_version.stdout }}"

    - name: Lancer un conteneur test nginx
      ansible.builtin.shell: |
        docker run -d --name test-nginx -p 8080:80 nginx:alpine || true
```

Exécution :
```bash
ansible-playbook -i ansible/inventory.ini ansible/deploy.yml --limit dev
```

---

### 6️⃣ SonarQube — Installation & Projet

- Accès : http://localhost:9000  
- Login : admin / admin  
- Créer projet `my-app`  
- Générer un token  
- Ajouter dans Jenkins credentials `sonar_token`

---

### 7️⃣ Structure du projet

```
my-app/
├── app.py
├── tests/
│   └── test_health.py
├── requirements.txt
├── Dockerfile
├── sonar-project.properties
├── ansible/
│   ├── inventory.ini
│   ├── ping.yml
│   └── deploy.yml
├── docker-compose.test.yml
└── Jenkinsfile  (à venir)
```

---

### ✅ Prochaines étapes

1. Valider la connexion Ansible (ping.yml)
2. Déployer un conteneur test via deploy.yml
3. Ajouter Jenkins et créer le pipeline CI/CD complet.

---

Auteur : **TP7 CI/CD — Jenkins + Docker + Ansible + SonarQube**
