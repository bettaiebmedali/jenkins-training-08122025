# TP Jenkins — Housekeeping (Maintenance Système)

## Objectif
Mettre en place un ensemble d’actions de maintenance afin de :
- nettoyer les workspaces
- limiter la rétention des builds
- supprimer les anciens artefacts
- vérifier les plugins
- sauvegarder la configuration Jenkins
- automatiser la maintenance avec un job dédié

---

# 1. Configuration de la rétention des builds

## 1.1. Interface Freestyle
Activer dans le job :
- **Discard Old Builds**
- `Max # of builds to keep = 30`
- `Days to keep builds = 30`

## 1.2. Pipeline
Ajouter dans le Jenkinsfile :

```groovy
options {
  buildDiscarder(logRotator(numToKeepStr: '30', daysToKeepStr: '30'))
}
2. Nettoyage du workspace
2.1. Freestyle
Cocher :

Delete workspace before build starts

ou Delete workspace when build is done

2.2. Pipeline
groovy
Copy code
wsCleanup()
ou (si plugin non installé) :

groovy
Copy code
sh 'rm -rf *'
3. Suppression des anciens artefacts (script shell)
Créer un job "maintenance" (type Script Shell) avec :

bash
Copy code
#!/bin/bash
JENKINS_HOME=/var/lib/jenkins

# supprimer artefacts zip de plus de 30 jours
find "${JENKINS_HOME}/jobs" -type f -name '*.zip' -mtime +30 -print -delete

# supprimer dossiers archive anciens (+60j)
find "${JENKINS_HOME}/jobs" -type d -name "archive" -mtime +60 -print -exec rm -rf {} \;
4. Nettoyage automatique des workspaces sur les nodes (Groovy)
À exécuter dans :
Manage Jenkins → Script Console

groovy
Copy code
import jenkins.model.*
def jenkins = Jenkins.instance
def cutoffDays = 30
def cutoff = System.currentTimeMillis() - (cutoffDays * 24L*60*60*1000)

jenkins.getNodes().each { node ->
  println("Node: ${node.displayName}")
  def wsRoot = node.getWorkspaceRoot()
  if (wsRoot) {
    def dir = new File(wsRoot)
    if (dir.exists()) {
      dir.eachDir { d ->
        if (d.lastModified() < cutoff) {
          println("Deleting old workspace: ${d}")
          d.deleteDir()
        }
      }
    }
  }
}
5. Sauvegarde automatique de Jenkins
Créer un job "backup" :

bash
Copy code
#!/bin/bash
JENKINS_HOME=/var/lib/jenkins
BACKUP_DIR=/backups/jenkins

mkdir -p $BACKUP_DIR

tar czf $BACKUP_DIR/jenkins-$(date +%F).tar.gz -C $(dirname $JENKINS_HOME) $(basename $JENKINS_HOME)

# suppression des backups > 14 jours
find $BACKUP_DIR -type f -mtime +14 -delete
6. Vérification des plugins obsolètes
6.1. Via Script Console
groovy
Copy code
def pm = Jenkins.instance.pluginManager
pm.plugins.each { plugin ->
  if (plugin.hasUpdate()) {
    println "${plugin.getShortName()} : ${plugin.getVersion()} → update available"
  }
}
7. Suppression des jobs obsolètes
Lister les jobs via CLI :

bash
Copy code
java -jar jenkins-cli.jar -s http://JENKINS_URL list-jobs
Supprimer job (ex) :

bash
Copy code
java -jar jenkins-cli.jar -s http://JENKINS_URL delete-job old-job-name
8. Pipeline complet "jenkins-housekeeping"
Créer un job Pipeline :

groovy
Copy code
pipeline {
  agent any
  triggers {
    // chaque lundi matin
    cron('H H(2-4) * * 1')
  }
  options {
    timestamps()
  }
  stages {

    stage('Backup Jenkins') {
      steps {
        sh '''
        JENKINS_HOME=/var/lib/jenkins
        BACKUP_DIR=/backups/jenkins
        mkdir -p $BACKUP_DIR
        tar czf $BACKUP_DIR/jenkins-$(date +%F).tar.gz -C $(dirname $JENKINS_HOME) $(basename $JENKINS_HOME)
        '''
      }
    }

    stage('Cleanup Artifacts') {
      steps {
        sh '''
        find /var/lib/jenkins/jobs -type f -name "*.zip" -mtime +30 -delete
        find /var/lib/jenkins/jobs -type d -name "archive" -mtime +60 -exec rm -rf {} \\;
        '''
      }
    }

    stage('Workspace Cleanup') {
      steps {
        wsCleanup()
      }
    }

    stage('Plugin Check') {
      steps {
        echo "Plugins obsolètes visibles dans Manage Jenkins → Plugin Manager → Updates"
      }
    }
  }

  post {
    success {
      echo "Maintenance terminée avec succès."
    }
    failure {
      mail to: 'admin@example.com',
           subject: 'Housekeeping Jenkins — échec',
           body: 'Vérifier le job Jenkins-housekeeping.'
    }
  }
}
9. Questions d’évaluation
Pourquoi limiter la rétention des builds ?

Quelle est la différence entre workspace et artefacts ?

Quels risques présente un plugin non mis à jour ?

Pourquoi externaliser les artefacts (Nexus, S3…) ?

Pourquoi vaut-il mieux tester les mises à jour Jenkins sur une instance de staging ?

10. Résultats attendus
Rétention des builds configurée (30 builds / 30 jours)

Workspaces nettoyés automatiquement

Artefacts anciens supprimés

Sauvegarde Jenkins générée dans /backups/

Liste des plugins obsolètes obtenue

Job "jenkins-housekeeping" exécuté avec succès
