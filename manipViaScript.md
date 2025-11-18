# TP — Manipulation des Jobs Jenkins via Script Console (Move, Scan, Folder)

Ce TP vous permet d’apprendre à :

- Exécuter un script Groovy dans **Jenkins Script Console**
- Déplacer automatiquement des jobs
- Créer des dossiers dynamiquement
- Faire un audit des jobs / folders
- Ajouter des use cases avancés (copie, renommage, verrouillage, suppression)

Compatible : Jenkins 2.528+

---

## 1. Pré-requis

- Accès à **Manage Jenkins → Script Console**
- Rôle administrateur Jenkins
- Des jobs existants à la racine
- Plugin **Folders** installé

---

## 2. Exécution du script (rappel)

Ouvrez :

Manage Jenkins → Script Console

kotlin
Copy code

Collez un script Groovy puis exécutez.

⚠️ Les scripts agissent **en direct** sur l’instance.

---

# 3. TP — Déplacer un job dans un folder (script utilisé)

## 🟦 Script : déplacer un job dans un folder

```groovy
import jenkins.model.*
import com.cloudbees.hudson.plugins.folder.*

def jenkins = Jenkins.instance
def jobName = "alljobs"
def folderName = "PROJETS-Formation-17112025"

// chercher le job
def job = jenkins.getItem(jobName)
if (!job) {
    println "❌ Job '${jobName}' introuvable."
    return
}

// chercher le folder
def folder = jenkins.getItem(folderName)
if (!folder) {
    println "❌ Folder '${folderName}' introuvable."
    return
}

// déplacer
println "Déplacement du job '${jobName}' vers '${folderName}'..."
jenkins.move(job, folder)
println "✔ Déplacement terminé."
4. Use Case #1 — Déplacer TOUS les jobs dont le nom commence par « poste »
🟦 Script
groovy
Copy code
import jenkins.model.*
import com.cloudbees.hudson.plugins.folder.*

def jenkins = Jenkins.instance
def folder = jenkins.getItem("POSTE-JOBS") ?: jenkins.createProject(com.cloudbees.hudson.plugins.folder.Folder, "POSTE-JOBS")

jenkins.items.each { item ->
    if (item.name =~ /^poste/) {
        println "Déplacement : ${item.name}"
        jenkins.move(item, folder)
    }
}
println "✔ Tous les jobs 'poste*' ont été déplacés."
5. Use Case #2 — Créer automatiquement un folder par équipe (Team1, Team2, …)
groovy
Copy code
import jenkins.model.*
import com.cloudbees.hudson.plugins.folder.*

def jenkins = Jenkins.instance
def teams = ["TEAM1", "TEAM2", "TEAM3"]

teams.each { t ->
    if (!jenkins.getItem(t)) {
        jenkins.createProject(Folder, t)
        println "✔ Folder créé : ${t}"
    }
}
6. Use Case #3 — Copier un job dans un folder (clone job)
groovy
Copy code
import jenkins.model.*
import hudson.model.*

def jenkins = Jenkins.instance
def origin = jenkins.getItem("job-template")
def folder = jenkins.getItem("TEAM1")

def newJob = Items.copy(origin, "job-template-copy")
folder.add(newJob, "job-template-copy")

println "✔ Job cloné dans TEAM1"
7. Use Case #4 — Renommer automatiquement des jobs
groovy
Copy code
import jenkins.model.*

def jenkins = Jenkins.instance

jenkins.items.each { job ->
    if (job.name.contains("dev-")) {
        def newName = job.name.replace("dev-", "develop-")
        println "Renommage : ${job.name} → ${newName}"
        job.renameTo(newName)
    }
}
println "✔ Opération terminée."
8. Use Case #5 — Auditer tous les jobs + leur emplacement
groovy
Copy code
import jenkins.model.*
import com.cloudbees.hudson.plugins.folder.*

def printTree
printTree = { item, level ->
    println ("  " * level) + "• " + item.name
    if (item instanceof com.cloudbees.hudson.plugins.folder.Folder) {
        item.items.each { sub ->
            printTree(sub, level + 1)
        }
    }
}

println "# Arborescence complète Jenkins :"
Jenkins.instance.items.each { rootItem ->
    printTree(rootItem, 0)
}
9. Use Case #6 — Supprimer tous les jobs marqués “deprecated”
groovy
Copy code
import jenkins.model.*

Jenkins.instance.items.each { job ->
    if (job.name.contains("deprecated")) {
        println "Suppression : ${job.name}"
        job.delete()
    }
}
println "✔ Tous les jobs 'deprecated' ont été supprimés."
10. Use Case #7 — Vérifier quels jobs n’ont pas été lancés depuis plus de 60 jours
groovy
Copy code
import jenkins.model.*

println "Jobs inactifs depuis +60 jours :"

Jenkins.instance.items.each { job ->
    def last = job.getLastBuild()?.getTimestamp()
    if (last) {
        long days = (System.currentTimeMillis() - last.timeInMillis) / (1000*60*60*24)
        if (days > 60) {
            println "• ${job.name} — ${days} jours"
        }
    }
}
11. Use Case #8 — Désactiver tous les jobs sauf production
groovy
Copy code
import jenkins.model.*

Jenkins.instance.items.each { job ->
    if (!job.name.contains("prod")) {
        println "Désactivation : ${job.name}"
        job.disable()
    }
}
println "✔ Tous les jobs non-prod sont désactivés."
12. Fin du TP
Vous savez désormais :

Exécuter un script dans Jenkins

Réorganiser automatiquement vos jobs

Gérer folders, copie, suppression, renommage

Faire des audits automatisés