# Étape 4 — SonarQube (analyse statique du code)

[⬅️ Retour au sommaire](../README.md)

---


### 4.1 Objectif
Déployer **SonarQube** sur le serveur CI (même instance EC2 que Jenkins) pour analyser automatiquement la qualité et la sécurité du code à chaque build, avec un **Quality Gate** qui peut bloquer le pipeline si le code ne respecte pas les critères définis.

### 4.2 Prérequis techniques importants
SonarQube embarque un moteur **Elasticsearch** qui nécessite des réglages système spécifiques sur l'hôte, sans quoi le conteneur plantera au démarrage.

Connecte-toi en SSH sur l'instance EC2 :

```bash
ssh -i ci-server-key.pem ubuntu@<IP_EC2>
```

Applique les réglages requis :

```bash
sudo sysctl -w vm.max_map_count=262144
sudo sysctl -w fs.file-max=131072
```

Rends ces réglages **permanents** (persistent après reboot) :

```bash
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
echo "fs.file-max=131072" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 4.3 Ouverture du port 9000 sur le groupe de sécurité EC2

1. Console AWS → **EC2** → **Security Groups** → sélectionne `ci-server-sg`
2. **Inbound rules** → **Edit inbound rules** → **Add rule** :

| Type | Port | Source |
|---|---|---|
| Custom TCP | 9000 | Ton IP publique (`Mon IP`) |

3. **Save rules**

### 4.4 Déploiement de SonarQube en conteneur Docker (avec PostgreSQL)

```bash
mkdir -p /srv/sonarqube
cd /srv/sonarqube
nano docker-compose.yml
```

```yaml
services:
  sonarqube_db:
    image: postgres:15
    container_name: sonarqube_db
    restart: always
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: changeMeSonarDbPwd
      POSTGRES_DB: sonarqube
    volumes:
      - sonar_db_data:/var/lib/postgresql/data

  sonarqube:
    image: sonarqube:lts-community
    container_name: sonarqube
    restart: always
    depends_on:
      - sonarqube_db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://sonarqube_db:5432/sonarqube
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: changeMeSonarDbPwd
    ports:
      - '9000:9000'
    volumes:
      - sonar_data:/opt/sonarqube/data
      - sonar_extensions:/opt/sonarqube/extensions
      - sonar_logs:/opt/sonarqube/logs
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
      nproc: 4096

volumes:
  sonar_db_data:
  sonar_data:
  sonar_extensions:
  sonar_logs:
```

> ⚠️ Remplace `changeMeSonarDbPwd` par un mot de passe robuste, **identique** dans les deux services (`sonarqube_db` et `sonarqube`).

Démarre les conteneurs :
```bash
docker compose up -d
docker compose logs -f sonarqube
```

Attends un message du type `SonarQube is operational`. Le premier démarrage peut prendre 2 à 3 minutes.

### 4.5 Première connexion à SonarQube

1. Va sur `http://<IP_EC2>:9000`
2. Connecte-toi avec les identifiants par défaut :
   - **Login** : `admin`
   - **Mot de passe** : `admin`
3. SonarQube te force immédiatement à **définir un nouveau mot de passe**, fais-le

### 4.6 Génération d'un token SonarQube pour Jenkins

1. Clique sur ton avatar (en haut à droite) → **My Account** → onglet **Security**
2. Section **Generate Tokens** :
   - Nom : `jenkins-integration`
   - Type : **Global Analysis Token**
   - Expiration : selon ta politique
3. Clique sur **Generate** → **copie immédiatement le token** (affiché une seule fois)

### 4.7 Installation du plugin SonarQube Scanner dans Jenkins

1. Jenkins → **Manage Jenkins** → **Plugins** → **Available plugins**
2. Recherche `SonarQube Scanner` → coche-le → **Install**
3. Redémarre Jenkins si demandé

### 4.8 Enregistrement du token SonarQube dans les Credentials Jenkins

1. **Manage Jenkins** → **Credentials** → **System** → **Global credentials** → **Add Credentials**
2. Type : **Secret text**
3. Secret : colle le token généré à l'étape 4.6
4. ID : `sonarqube-token`
5. **Create**

### 4.9 Configuration du serveur SonarQube dans Jenkins (Configure System)

1. **Manage Jenkins** → **System**
2. Section **SonarQube servers** :
   - Coche **Environment variables** (permet d'injecter les variables dans le pipeline)
   - Clique sur **Add SonarQube**
     - **Name** : `sonarqube-server` (nom qu'on réutilisera dans le `Jenkinsfile`)
     - **Server URL** : `http://sonarqube:9000` si Jenkins et SonarQube partagent un réseau Docker commun, sinon `http://<IP_EC2>:9000`
     - **Server authentication token** : sélectionne `sonarqube-token`
3. **Save**

> 💡 Pour que Jenkins puisse joindre SonarQube via `http://sonarqube:9000` (nom de service au lieu de l'IP), les deux conteneurs doivent être sur le **même réseau Docker**. Sinon, utilise simplement l'IP de l'EC2 avec le port 9000 — fonctionne également très bien.

### 4.10 Configuration de l'outil SonarQube Scanner (Global Tool Configuration)

1. **Manage Jenkins** → **Tools**
2. Section **SonarQube Scanner installations** → **Add SonarQube Scanner**
   - Name : `sonar-scanner`
   - Coche **Install automatically** (Jenkins téléchargera le scanner tout seul)
3. **Save**

### 4.11 Configuration du Webhook SonarQube → Jenkins (pour le Quality Gate)

Ce webhook permet à SonarQube de notifier Jenkins dès que l'analyse est terminée, sans que Jenkins ait à sonder (poll) en boucle.

1. Dans SonarQube : **Administration** → **Configuration** → **Webhooks**
2. **Create** :
   - Name : `jenkins`
   - URL : `http://<IP_EC2>:8080/sonarqube-webhook/`
3. **Create**

### 4.12 Ajout du fichier de configuration du projet (`sonar-project.properties`)

À la racine de ton dépôt Git (celui cloné depuis GitLab), crée :

```properties
sonar.projectKey=pipeline-cicd-demo
sonar.projectName=PIPELINE DEMO
sonar.sources=.
sonar.sourceEncoding=UTF-8
```

> Adapte `sonar.sources` selon la structure de ton projet (ex. `src/` si le code est dans un sous-dossier).

### 4.13 Mise à jour du `Jenkinsfile` (dans le dépôt Git) avec le stage SonarQube

Le job Jenkins étant configuré en **Pipeline script from SCM** (étape 3.12), le `Jenkinsfile` se modifie directement dans ton dépôt GitLab, pas dans l'interface Jenkins. Remplace le contenu du `Jenkinsfile` créé à l'étape 3.12ter par cette version incluant l'analyse SonarQube et l'attente du Quality Gate :

```groovy
pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
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
    }
}
```

> `abortPipeline: true` stoppe le pipeline si le Quality Gate est en échec (code non conforme aux critères qualité/sécurité définis). C'est volontaire : c'est le rôle de cette étape de bloquer les builds problématiques avant qu'ils n'avancent vers le build Docker (étape 5).

### 4.14 Test bout-en-bout

1. Fais un `git push` sur ton dépôt GitLab
2. Vérifie dans Jenkins que le build se déclenche et exécute les stages **Checkout → SonarQube Analysis → Quality Gate**
3. Va sur `http://<IP_EC2>:9000` → ton projet `pipeline-cicd-demo` doit apparaître avec les résultats d'analyse (bugs, vulnérabilités, code smells, couverture)
4. Vérifie le statut du **Quality Gate** (Passed/Failed) affiché à la fois dans SonarQube et dans les logs du build Jenkins


---

⬅️ **Précédent** : [Étape 3 — Serveur CI (EC2) + Jenkins + Webhook GitLab](./03-jenkins-ci-ec2.md)  
➡️ **Suivant** : [Étape 5 — Build de l'image Docker (application Django)](./05-build-image-docker.md)
