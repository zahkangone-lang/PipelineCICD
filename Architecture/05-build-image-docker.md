# Étape 5 — Build de l'image Docker (application Django)

[⬅️ Retour au sommaire](../README.md)

---


### 5.1 Objectif
Ajouter au pipeline Jenkins un stage qui construit l'image Docker de l'application `MyAppPipe` (soit Python/Django ou Laravel/PHP ou autres) après validation du Quality Gate SonarQube.

### 5.2 Prérequis important — Docker CLI absent de l'image Jenkins
L'image `jenkins/jenkins:lts-jdk17` utilisée à l'étape 3.6 ne contient **pas** le client Docker (`docker`). Le socket `/var/run/docker.sock` est bien monté, mais sans le binaire `docker`, impossible d'exécuter `docker build`. Il faut donc construire une image Jenkins personnalisée qui ajoute le client Docker.

### 5.3 Création d'une image Jenkins personnalisée avec Docker CLI

Sur l'instance EC2 `ci-server` :

```bash
cd /srv/jenkins
nano Dockerfile
```

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

USER jenkins
```

### 5.4 Mise à jour du `docker-compose.yml` de Jenkins pour utiliser cette image personnalisée

```bash
nano /srv/jenkins/docker-compose.yml
```

Remplace la ligne `image: jenkins/jenkins:lts-jdk17` par un `build` sur le `Dockerfile` local :

```yaml
services:
  jenkins:
    build: .
    container_name: jenkins
    restart: always
    ports:
      - '8080:8080'
      - '50000:50000'
    volumes:
      - 'jenkins_home:/var/jenkins_home'
      - '/var/run/docker.sock:/var/run/docker.sock'
    group_add:
      - '999'

volumes:
  jenkins_home:
```

Reconstruis et relance :
```bash
docker compose up -d --build
docker compose logs -f jenkins
```

> ✅ Le volume `jenkins_home` est conservé (jobs, credentials, plugins déjà configurés ne sont pas perdus).

**Vérification :**
```bash
docker exec -it jenkins docker --version
```

### 5.5 Création du `Dockerfile` de l'application (Django pour notre cas)

À la racine du dépôt `MyAppPipe` :

```bash
nano Dockerfile
```

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential \
      libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

COPY --chown=appuser:appuser entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

EXPOSE 8000

ENTRYPOINT ["/app/entrypoint.sh"]
```

> ⚠️ Assure-toi que `gunicorn` figure bien dans ton `requirements.txt` (`pip install gunicorn` puis `pip freeze` si besoin).
> ⚠️ Adapte `libpq-dev` uniquement si tu utilises PostgreSQL comme base de données ; retire-le sinon (par ex. si tu es en MySQL, remplace par `default-libmysqlclient-dev`).

### 5.6 Création du script `entrypoint.sh`

`collectstatic` et `migrate` nécessitent les variables d'environnement (base de données, `SECRET_KEY`, etc.), on les exécute donc **au démarrage du conteneur**, pas pendant le build (où ces variables ne sont pas encore disponibles) :

```bash
nano entrypoint.sh
```

```bash
#!/bin/sh
set -e

echo "Collecte des fichiers statiques..."
python manage.py collectstatic --noinput

echo "Application des migrations..."
python manage.py migrate --noinput

echo "Démarrage de Gunicorn..."
exec gunicorn <NOM_DU_PROJET_DJANGO>.wsgi:application --bind 0.0.0.0:8000 --workers 3
```

> ⚠️ Remplace `<NOM_DU_PROJET_DJANGO>` par le nom réel du dossier contenant ton fichier `wsgi.py` (celui créé par `django-admin startproject <nom>`).

### 5.7 Création du `.dockerignore`

```bash
nano .dockerignore
```

```
__pycache__/
*.pyc
.git
.gitignore
.env
venv/
env/
db.sqlite3
staticfiles/
media/
*.log
```

### 5.8 Ajout du stage "Build Docker Image" dans le `Jenkinsfile`

> 💡 **Pourquoi `agent any` ?** Ce projet n'utilise qu'une seule machine CI (l'EC2 `ci-server`) : Jenkins n'a donc qu'un seul nœud disponible, le **nœud intégré** (le conteneur Jenkins lui-même). `agent any` exécute donc tout le pipeline directement dans ce conteneur, qui dispose du client Docker (installé à l'étape 5.3) et du socket `/var/run/docker.sock`, suffisant pour `docker build` et, bientôt, `trivy`. Une architecture avec plusieurs agents dédiés (`agent { label '...' }`, agents Docker/Kubernetes) ne serait utile qu'en cas de parallélisation de builds ou d'isolation stricte par outil mais pas nécessaire ici.

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
    }
}
```

### 5.9 Commit et test

```bash
git add Dockerfile entrypoint.sh .dockerignore Jenkinsfile requirements.txt
git commit -m "Ajout Dockerfile et stage de build Docker"
git push
```

Vérifie dans Jenkins que le stage **Build Docker Image** passe au vert, puis sur l'EC2 :

```bash
docker images | grep my-app-pipe
```

L'image doit apparaître avec le tag correspondant au numéro de build.


---

⬅️ **Précédent** : [Étape 4 — SonarQube (analyse statique du code)](./04-sonarqube.md)  
➡️ **Suivant** : [Étape 6 — Scan de vulnérabilités avec Trivy](./06-trivy-scan.md)