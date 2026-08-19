# Étape 3 — Serveur CI (EC2) + Jenkins + Webhook GitLab

[⬅️ Retour au sommaire](../README.md)

---


### 3.1 Objectif
Créer le serveur CI (Amazon EC2) qui hébergera Jenkins (puis SonarQube, Trivy et le registre Docker aux étapes suivantes), et connecter GitLab à Jenkins via un **webhook** : chaque `push` déclenchera automatiquement un job Jenkins.

### 3.2 Prérequis
- Compte AWS actif
- Projet GitLab déjà créé (étape 2.12)
- Le VPS GitLab doit être joignable depuis Internet

### 3.3 Création de l'instance EC2

1. Connecte-toi sur **console.aws.amazon.com**
2. Va dans le service **EC2** → région de ton choix (ex. `eu-west-3` Paris)
3. Clique sur **Launch Instance**
4. Configure :

| Paramètre | Valeur recommandée |
|---|---|
| Nom | `ci-server` |
| AMI | Ubuntu Server 22.04 LTS (64-bit x86) |
| Type d'instance | `t3.large` (2 vCPU / 8 Go RAM) — cette instance va accueillir Jenkins **+** SonarQube **+** Trivy **+** registre Docker aux prochaines étapes, prévoir large dès maintenant évite de redimensionner plus tard |
| Paire de clés | Créer une nouvelle paire (ex. `ci-server-key`), **télécharger le `.pem`** et le conserver précieusement (non récupérable ensuite) |
| Stockage | 40 Go minimum (gp3) |

5. **Groupe de sécurité** — crée-en un nouveau (`ci-server-sg`) avec ces règles entrantes initiales :

| Type | Port | Source |
|---|---|---|
| SSH | 22 | Ton IP publique uniquement (`Mon IP`) |
| Custom TCP | 8080 | Ton IP publique (accès Jenkins UI) — on pourra restreindre plus tard |

> D'autres ports (9000 pour SonarQube, 5000 pour le registre Docker, etc.) seront ajoutés aux étapes suivantes.

6. Clique sur **Launch Instance**
7. Une fois l'instance en état `running`, note son **adresse IP publique** (EC2 → Instances → sélectionner l'instance → onglet Détails)

### 3.4 Connexion SSH à l'instance EC2

Sur ta machine locale, sécurise la clé et connecte-toi :

```bash
chmod 400 ci-server-key.pem
ssh -i ci-server-key.pem ubuntu@<IP_EC2>
```

> Vidéo Réf. : https://www.youtube.com/watch?v=Rj6jWsfLb4g&list=PLJuTqSmOxhNtS9Z37Uut14fPwpdtz6Fdh&index=9(A partir de la minute 29:50)

### 3.5 Installation de Docker sur l'instance EC2

Même procédure qu'à l'étape 1.8 (réappliquée ici car nouvelle machine) :

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
```

> ⚠️ Déconnecte-toi (`exit`) puis reconnecte-toi en SSH pour que l'ajout au groupe `docker` prenne effet (évite d'avoir à taper `sudo` avant chaque commande `docker`).

### 3.6 Déploiement de Jenkins en conteneur Docker

```bash
mkdir -p /srv/jenkins
cd /srv/jenkins
nano docker-compose.yml
```

```yaml
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
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

> Le montage de `/var/run/docker.sock` permet à Jenkins de piloter Docker sur l'hôte (utile pour les étapes de build d'image Docker à l'étape 5). Le `group_add: '999'` correspond en général au GID du groupe `docker`, vérifiable avec `getent group docker` sur l'hôte ; ajuste la valeur si différente.

Démarre le conteneur :
```bash
docker compose up -d
docker compose logs -f jenkins
```

Attends de voir dans les logs une ligne du type :
```
Jenkins is fully up and running
```

### 3.7 Déverrouillage initial de Jenkins

1. Récupère le mot de passe initial :
```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
2. Va sur `http://<IP_EC2>:8080` dans ton navigateur
3. Colle le mot de passe dans **Unlock Jenkins**
4. Choisis **Install suggested plugins** (installation des plugins standards)
5. Crée ton **utilisateur administrateur** (nom, mot de passe, email)
6. Valide l'URL de l'instance Jenkins (garde `http://<IP_EC2>:8080/`)

### 3.8 Installation du plugin GitLab dans Jenkins

1. **Manage Jenkins** → **Plugins**
2. Onglet **Available plugins**
3. Recherche `GitLab` → coche le plugin **GitLab Plugin**
4. Recherche également `Git` → vérifie qu'il est bien installé (généralement inclus dans les plugins suggérés)
5. Clique sur **Install**, puis redémarre Jenkins si demandé

### 3.9 Création d'un Personal Access Token GitLab (pour Jenkins → GitLab)

1. Connecte-toi à ton GitLab (`https://<IP_DU_VPS>`)
2. Clique sur ton avatar → **Edit profile** → **Access Tokens**
3. Crée un token :
   - Nom : `jenkins-integration`
   - Scopes : coche `api`, `read_repository`, `write_repository`
   - Expiration : selon ta politique (ex. 1 an)
4. Clique sur **Create personal access token**
5. **Copie immédiatement le token** (affiché une seule fois)

### 3.10 Enregistrement du token dans Jenkins (Credentials)

1. Dans Jenkins : **Manage Jenkins** → **Credentials** → **System** → **Global credentials** → **Add Credentials**
2. Type : **GitLab API token**
3. Colle le token GitLab récupéré à l'étape 3.9
4. ID : `gitlab-api-token` (facilite les références futures)
5. Clique sur **Create**

### 3.11 Connexion Jenkins ↔ GitLab (Configure System)

1. **Manage Jenkins** → **System**
2. Section **GitLab** :
   - **Connection name** : `gitlab-vps`
   - **GitLab host URL** : `https://<IP_DU_VPS>`
   - **Credentials** : sélectionne `gitlab-api-token` créé à l'étape 3.10
3. Clique sur **Test Connection** → doit afficher un message de succès
4. **Save**

### 3.12 Création du Job/Pipeline Jenkins

1. Sur le tableau de bord Jenkins : **New Item**
2. Nom : `pipeline-cicd-demo`
3. Type : **Pipeline**
4. Dans la configuration du job :
   - Section **Build Triggers** → coche **Build when a change is pushed to GitLab. GitLab webhook URL:**
   - Jenkins affiche alors une URL du type :
     ```
     http://<IP_EC2>:8080/project/pipeline-cicd-demo
     ```
     **Copie cette URL**, elle servira à l'étape 3.13
   - Coche également les événements souhaités : **Push Events** (et **Merge Request Events** si tu veux déclencher sur MR)
   - Dans **Advanced**, génère un **Secret token** (bouton *Generate*) — **copie-le aussi**
5. Section **Pipeline** — ⚠️ **ne pas utiliser un script inline** (`Pipeline script`), car les futures étapes utiliseront `checkout scm`, qui exige que le pipeline soit défini via SCM. Configure plutôt :
   - **Definition** : **Pipeline script from SCM**
   - **SCM** : `Git`
   - **Repository URL** : `https://<IP_DU_VPS>/MyAppPipe.git` (URL HTTPS de ton projet GitLab)
   - **Credentials** : voir étape 3.12bis ci-dessous pour les créer
   - **Branch Specifier** : `*/main` (adapte selon ta branche par défaut)
   - **Script Path** : `Jenkinsfile`
6. **Save**

### 3.12bis Credentials Jenkins pour cloner le dépôt GitLab (HTTPS)

Comme le dépôt est privé, Jenkins a besoin d'identifiants pour le cloner :

1. **Manage Jenkins** → **Credentials** → **System** → **Global credentials** → **Add Credentials**
2. Type : **Username with password**
3. Username : ton nom d'utilisateur GitLab
4. Password : le token GitLab créé à l'étape 3.9 (`jenkins-integration`, scope `read_repository`)
5. ID : `gitlab-clone-creds`
6. **Create**

### 3.12ter Création du fichier `Jenkinsfile` dans le dépôt

À la racine de ton projet GitLab, crée un fichier `Jenkinsfile` avec le contenu du pipeline. Pour l'instant, un script minimal de test :

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Test connexion') {
            steps {
                echo 'Webhook GitLab -> Jenkins fonctionnel !'
            }
        }
    }
}
```

Commit et push ce fichier, il sera enrichi avec les stages SonarQube à l'étape 4.13, puis build/Trivy/registry aux étapes suivantes.

### 3.13 Configuration du Webhook côté GitLab

1. Va sur ton projet GitLab → **Settings** → **Webhooks**
2. **URL** : colle l'URL récupérée à l'étape 3.12 (ex. `http://<IP_EC2>:8080/project/pipeline-cicd-demo`)
3. **Secret token** : colle le token généré à l'étape 3.12
4. **Trigger** : coche **Push events** (branche : all branches, ou restreindre à `main`)
5. Laisse **Enable SSL verification** décoché si tu es en HTTP simple, sinon coché
6. Clique sur **Add webhook**

### 3.14 Test du webhook

1. Dans la liste des webhooks GitLab, clique sur **Test** → **Push events**
2. Va vérifier dans Jenkins : le job `pipeline-cicd-demo` doit apparaître en cours d'exécution (ou dans l'historique des builds)
3. Test réel : fais un `git push` sur ton dépôt GitLab et vérifie qu'un nouveau build se déclenche automatiquement dans Jenkins


---

⬅️ **Précédent** : [Étape 2 — Déploiement de GitLab CE en conteneur Docker](./02-gitlab-docker.md)  
➡️ **Suivant** : [Étape 4 — SonarQube (analyse statique du code)](./04-sonarqube.md)