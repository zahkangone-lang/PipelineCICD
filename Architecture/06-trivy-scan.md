# Étape 6 — Scan de vulnérabilités avec Trivy

[⬅️ Retour au sommaire](../README.md)

---


### 6.1 Objectif
Ajouter au pipeline un stage qui scanne l'image Docker construite à l'étape 5 avec **Trivy**, pour détecter les vulnérabilités connues (CVE) dans les dépendances système et packages Python embarqués. Le pipeline sera bloqué si des vulnérabilités **HIGH** ou **CRITICAL** sont détectées.

### 6.2 Installation de Trivy dans l'image Jenkins personnalisée

Comme pour Docker CLI à l'étape 5.3, Trivy doit être installé **dans le conteneur Jenkins** puisque le pipeline tourne en `agent any` (le nœud intégré, c'est-à-dire ce même conteneur).

```bash
nano /srv/jenkins/Dockerfile
```

Ajoute l'installation de Trivy après celle du client Docker :

```dockerfile
FROM jenkins/jenkins:lts-jdk17
USER root

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl gnupg \
    && install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
    && chmod a+r /etc/apt/keyrings/docker.asc \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
       > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y docker-ce-cli \
    && rm -rf /var/lib/apt/lists/*

RUN curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
    | sh -s -- -b /usr/local/bin

USER jenkins
```

> 💡 Le script officiel `install.sh` installe le binaire directement dans `/usr/local/bin`, sans dépôt APT à gére. C'est plus simple et fiable.

### 6.3 Reconstruction du conteneur Jenkins

```bash
cd /srv/jenkins
docker compose up -d --build
```

**Vérification :**
```bash
docker exec -it jenkins trivy --version
```

> ✅ Le cache de la base de vulnérabilités Trivy (`~/.cache/trivy`, soit `/var/jenkins_home/.cache/trivy` dans le conteneur) est automatiquement conservé entre les builds grâce au volume `jenkins_home` déjà persistant.

### 6.4 Ajout du stage "Trivy Scan" dans le `Jenkinsfile`

```groovy
pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME   = 'my-app-pipe'
        IMAGE_TAG    = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                  trivy image \
                    --severity HIGH,CRITICAL \
                    --ignore-unfixed \
                    --exit-code 1 \
                    --format table \
                    -o trivy-report.txt \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
        }
    }
}
```

**Ce que fait ce stage :**
- `--severity HIGH,CRITICAL` : ne remonte que les vulnérabilités importantes (ignore LOW/MEDIUM pour éviter le bruit)
- `--ignore-unfixed` : **ignore les vulnérabilités sans correctif disponible** sur une image Debian/Python, une bonne partie des CVE système (kernel headers, Perl embarqué pour les scripts de paquets, etc.) n'ont pas encore de patch publié ; bloquer le pipeline dessus n'a aucune valeur actionnable.
- `--exit-code 1` : fait échouer le stage (donc bloque le pipeline avant le push vers le registre) si au moins une vulnérabilité HIGH/CRITICAL **avec correctif disponible** est trouvée.
- `-o trivy-report.txt` : écrit le rapport dans un fichier
- Le bloc `post { always { archiveArtifacts ... } }` **conserve le rapport dans Jenkins** (onglet du build → *Artifacts*) même si le pipeline échoue, pour pouvoir consulter le détail des vulnérabilités

> 📋 **Retour d'expérience (premier scan réel du projet utiliser lors des tests du pipleline)** : sur 77 CVE système détectées, la quasi-totalité concernait des paquets sans lien avec le runtime applicatif (kernel headers `linux-libc-dev`, Perl utilisé uniquement par les scripts de gestion de paquets Debian, utilitaires `util-linux`/`libblkid`) et sans correctif publié, donc `--ignore-unfixed` les élimine du blocage. Restaient **7 CVE Python réellement actionnables** : Django 5.1.6 (CVE-2025-64459, injection SQL **CRITICAL**, + 3 CVE HIGH) et Pillow 11.1.0 (3 CVE dont exécution de code arbitraire). Correction appliquée : mise à jour de `requirements.txt` vers `Django==5.1.14` et `Pillow==12.2.0` (versions corrigées les plus proches, sans saut de version majeure pour Django).

### 6.5 Commit et test

```bash
git add Jenkinsfile
git commit -m "Ajout du scan de vulnérabilités Trivy"
git push
```

Vérifie dans Jenkins :
1. Le stage **Trivy Scan** apparaît dans le **Pipeline Overview**
2. Le rapport `trivy-report.txt` est accessible depuis la page du build → **Artifacts**
3. Si le build échoue à cause de vulnérabilités détectées, consulte le rapport pour voir le détail (package concerné, CVE, sévérité, version corrigée disponible)


---

⬅️ **Précédent** : [Étape 5 — Build de l'image Docker (application Django)](./05-build-image-docker.md)  
➡️ **Suivant** : [Étape 7 — Registre d'images Docker intégré à GitLab (Container Registry)](./07-container-registry-gitlab.md)