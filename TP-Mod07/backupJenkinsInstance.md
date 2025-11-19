# Backup Jenkins — Résumé

## 1. Sauvegarde complète du dossier JENKINS_HOME (recommandé)

### Dossier à sauvegarder
/var/lib/jenkins

bash
Copy code

### Commandes
```bash
sudo systemctl stop jenkins
sudo tar -czvf jenkins-backup-$(date +%F).tar.gz /var/lib/jenkins
sudo systemctl start jenkins
Restauration
bash
Copy code
sudo systemctl stop jenkins
sudo tar -xzvf jenkins-backup.tar.gz -C /
sudo systemctl start jenkins
2. Plugin ThinBackup (sauvegarde automatique)
Installation
nginx
Copy code
Manage Jenkins → Manage Plugins → Available → ThinBackup
Configuration
yaml
Copy code
Manage Jenkins → ThinBackup
Backup directory : /var/backups/jenkins/
Schedule : H 2 * * *
Include plugins : yes
Include credentials : yes
3. Backup minimal (fichiers essentiels)
Fichiers à sauvegarder
swift
Copy code
/var/lib/jenkins/config.xml
/var/lib/jenkins/users/
/var/lib/jenkins/jobs/
/var/lib/jenkins/credentials.xml
/var/lib/jenkins/secrets/
4. Script automatisé (optionnel)
bash
Copy code
#!/bin/bash
DATE=$(date +%F)
BACKUP=/opt/jenkins-backup-$DATE.tar.gz

sudo systemctl stop jenkins
sudo tar -czvf $BACKUP /var/lib/jenkins
sudo systemctl start jenkins
Cron
swift
Copy code
0 3 * * * root /usr/local/bin/backup-jenkins.sh