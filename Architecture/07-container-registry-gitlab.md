# Étape 7 — Registre d'images Docker intégré à GitLab (Container Registry)

[⬅️ Retour au sommaire](../README.md)

---


### 7.1 Objectif
Plutôt qu'un registre Docker séparé, nous utiliserons le **Container Registry intégré à GitLab** : chaque projet GitLab dispose nativement de son propre registre d'images (`Packages and Registries → Container Registry`), avec authentification basée sur les comptes/tokens GitLab déjà en place.

> ⚠️ **Contrainte technique liée à l'absence de nom de domaine.** Le Container Registry de GitLab est conçu pour fonctionner en HTTPS. Sans domaine, on va l'exposer en **HTTP simple**, ce qui n'est pas la configuration recommandée par GitLab en production, mais reste tout à fait fonctionnel pour un projet démo comme le notre. Cela implique de déclarer le registre en "insecure-registry" sur chaque moteur Docker qui devra le contacter.

### 7.2 Activation du Container Registry dans le `docker-compose.yml` de GitLab

Sur le **VPS Hostinger**, édite la configuration de GitLab :

```bash
cd /srv/gitlab
nano docker-compose.yml
```

Ajoute la ligne `registry_external_url` dans le bloc `GITLAB_OMNIBUS_CONFIG`, et mappe le port `5050` :

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: '<IP_DU_VPS>'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://<IP_DU_VPS>'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
        registry_external_url 'http://<IP_DU_VPS>:5050'
        gitlab_rails['registry_enabled'] = true
    ports:
      - '80:80'
      - '443:443'
      - '2224:22'
      - '5050:5050'
    volumes:
      - '/srv/gitlab/config:/etc/gitlab'
      - '/srv/gitlab/logs:/var/log/gitlab'
      - '/srv/gitlab/data:/var/opt/gitlab'
    shm_size: '256m'
```

Applique les changements :

```bash
docker compose up -d
docker exec -it gitlab gitlab-ctl reconfigure
docker compose logs -f gitlab
```

### 7.3 Ouverture du port 5050 sur le VPS Hostinger

```bash
ufw allow 5050/tcp
ufw status
```

> Vérifie aussi le pare-feu réseau côté **hPanel Hostinger** (en plus d'`ufw`) pour ce port, comme fait pour 80/443/2224 à l'étape 2.

### 7.4 Vérification dans l'interface GitLab

1. Va sur ton projet GitLab (`MyAppPipe`)
2. Menu de gauche : **Deploy** → **Container Registry**
3. La page doit s'afficher sans erreur, avec les instructions de connexion Docker propres à ton projet (commande `docker login` avec l'adresse exacte du registre)

### 7.5 Autoriser le registre en HTTP (insecure-registry) sur le moteur Docker de l'EC2

Comme à l'étape 7 précédente, il faut déclarer le registre comme "insecure" sur le moteur Docker de l'EC2 `ci-server` (celui qui exécute le `docker push` depuis Jenkins) :

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "insecure-registries": ["<IP_DU_VPS>:5050"]
}
```

```bash
sudo systemctl restart docker
```

> ⚠️ Ce redémarrage relance aussi Jenkins et SonarQube (conteneurs `restart: always`) donc attends qu'ils redémarrent puis vérifie `docker ps`.

> 💡 Plus tard, le(s) nœud(s) Kubernetes devront aussi avoir ce même réglage `insecure-registries` pour pouvoir `pull` les images (voir étape 9).

### 7.6 Génération d'un token GitLab dédié au registre (Deploy Token)

Plutôt que de réutiliser le token `jenkins-integration` (scopes API génériques), on crée un **Deploy Token** dédié, limité aux droits nécessaires sur le registre. C'est une bonne pratique de moindre privilège.

1. Sur ton projet GitLab : **Settings** → **Repository** → section **Deploy tokens**
2. **Add token** :
   - Name : `jenkins-registry-token`
   - Scopes : coche uniquement `read_registry` et `write_registry`
3. **Create deploy token**
4. **Copie immédiatement le username généré et le token** (affichés une seule fois)

### 7.7 Test manuel depuis l'EC2 `ci-server`

```bash
docker login <IP_DU_VPS>:5050 -u <username_du_deploy_token> -p '<deploy_token>'
```

> 📌 Format d'image GitLab : `<IP>:5050/<groupe>/<projet>[/<sous-image>]:<tag>`.

**Vérifie dans l'interface GitLab** (Deploy → Container Registry) que l'image builder apparaît bien.

### 7.8 Enregistrement du Deploy Token dans Jenkins (Credentials)

1. **Manage Jenkins** → **Credentials** → **System** → **Global credentials** → **Add Credentials**
2. Type : **Username with password**
3. Username : le username du deploy token (étape 7.6)
4. Password : le deploy token
5. ID : `gitlab-registry-creds`
6. **Create**

### 7.9 Ajout du stage "Push to Registry" dans le `Jenkinsfile`

```groovy
pipeline {
    agent any

    environment {
        SCANNER_HOME   = tool 'sonar-scanner'
        IMAGE_NAME     = 'my-app-pipe'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        REGISTRY       = '<IP_DU_VPS>:5050'
        REGISTRY_PATH  = 'my-app-pipe'
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

        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'gitlab-registry-creds',
                    usernameVariable: 'REG_USER',
                    passwordVariable: 'REG_PASS'
                )]) {
                    sh """
                      echo "\$REG_PASS" | docker login ${REGISTRY} -u "\$REG_USER" --password-stdin
                      docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${REGISTRY_PATH}:${IMAGE_TAG}
                      docker push ${REGISTRY}/${REGISTRY_PATH}:${IMAGE_TAG}
                      docker logout ${REGISTRY}
                    """
                }
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

**Points clés :**
- Le chemin d'image `${REGISTRY_PATH}` reprend le **namespace GitLab** (`groupe/projet`), c'est ce qui permet à GitLab de rattacher automatiquement les images au bon projet dans son Container Registry
- Le tag `${IMAGE_TAG}` (numéro de build Jenkins) est **l'unique tag poussé**. Chaque image est ainsi identifiée sans ambiguïté par son numéro de build, ce qui permet de savoir exactement quelle version tourne en production à tout moment (voir étape 11.4 pour vérifier le tag déployé depuis Argo CD)
- Ce stage n'est atteint que si SonarQube (Quality Gate) et Trivy (scan CVE) ont réussi

### 7.10 Commit et test

```bash
git add Jenkinsfile
git commit -m "Push vers le Container Registry GitLab au lieu d'un registre autonome"
git push
```

Vérifie dans Jenkins que le stage **Push to Registry** réussit, puis va sur GitLab → ton projet → **Deploy → Container Registry** : le tag `${BUILD_NUMBER}` doit apparaître.

### 7.11 (Optionnel) Nettoyage automatique des anciennes images

GitLab propose une politique de rétention pour éviter d'accumuler indéfiniment des images :
1. Projet → **Settings** → **Packages and registries** → **Container Registry** → **Cleanup policies**
2. Configure par exemple : garder les 10 dernières images par tag, expirer après 90 jours


---

⬅️ **Précédent** : [Étape 6 — Scan de vulnérabilités avec Trivy](./06-trivy-scan.md)  
➡️ **Suivant** : [Étape 8 — Cluster Kubernetes + Argo CD (GitOps)](./08-kubernetes-argocd.md)