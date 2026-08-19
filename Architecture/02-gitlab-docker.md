# Étape 2 — Déploiement de GitLab CE en conteneur Docker

[⬅️ Retour au sommaire](../README.md)

---


### 2.1 Objectif
Déployer GitLab Community Edition **dans un conteneur Docker** (image officielle `gitlab/gitlab-ce`) plutôt qu'en installation native Omnibus. Avantages : isolation, mises à jour simplifiées (changement de tag d'image), sauvegarde/restauration facilitée (volumes), portabilité.

> ℹ️ Le conteneur GitLab embarque en interne les composants suivant: Puma, Sidekiq, PostgreSQL, Redis et Nginx.

### 2.2 Choix des ports et point d'attention important sur SSH
Le VPS utilise déjà le **port 22** pour la connexion SSH d'administration. On ne peut donc pas laisser GitLab utiliser aussi le port 22 côté hôte pour le SSH Git.

➡️ On va mapper le SSH interne de GitLab (port 22 du conteneur) vers un **port hôte différent**, par exemple **2224**.

| Service | Port conteneur | Port hôte (VPS) |
|---|---|---|
| HTTP (web) | 80 | 80 |
| HTTPS (web) | 443 | 443 |
| SSH (git clone/push) | 22 | 2224 |

### 2.3 Création de l'arborescence de travail et des volumes

```bash
mkdir -p /srv/gitlab/config /srv/gitlab/logs /srv/gitlab/data
cd /srv/gitlab
```

Ces trois dossiers persisteront les données de GitLab même si le conteneur est supprimé/recréé.

> ⚠️ **NB** : dans les exemples ci-dessous, `<IP_DU_VPS>` est **placeholders à remplacer entièrement avec l'IP de votre VPS**, chevrons `< >` inclus.

### 2.4 Création du fichier `docker-compose.yml`

```bash
nano /srv/gitlab/docker-compose.yml
```

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: '<IP_DU_VPS>'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://<IP_DU_VPS>'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
        letsencrypt['enable'] = true
        letsencrypt['contact_emails'] = ['ton-email@example.com']
        nginx['redirect_http_to_https'] = true
    ports:
      - '80:80'
      - '443:443'
      - '2224:22'
    volumes:
      - '/srv/gitlab/config:/etc/gitlab'
      - '/srv/gitlab/logs:/var/log/gitlab'
      - '/srv/gitlab/data:/var/opt/gitlab'
    shm_size: '256m'
```

### 2.5 Ouverture des ports sur le pare-feu (si `ufw` actif)

```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 2224/tcp
ufw status
```

> N'oublie pas de vérifier aussi le **pare-feu/groupe de sécurité côté hPanel Hostinger** si un firewall réseau est activé en plus du firewall système. Pour notre cas il n'y a pas cela donc on continu.

### 2.6 Démarrage du conteneur

```bash
cd /srv/gitlab
docker compose up -d
```

Le premier démarrage peut prendre **5 à 10 minutes** : GitLab initialise sa base PostgreSQL, ses migrations, et ses services internes.

### 2.7 Suivi du démarrage et vérification

```bash
docker compose logs -f gitlab
```

Attends de voir un message du type `gitlab Reconfigured!` puis quitte le suivi des logs (`Ctrl + C`).

Vérifie que le conteneur tourne :
```bash
docker compose ps
```

Le statut doit être `Up` (ou `healthy` après quelques minutes, GitLab expose un healthcheck interne).

### 2.8 Récupération du mot de passe initial de l'utilisateur `root`
Le mot de passe temporaire est généré **à l'intérieur du conteneur**, valable 24h :

```bash
docker exec -it gitlab cat /etc/gitlab/initial_root_password
```

📌 **Note-le immédiatement**, ce fichier est supprimé automatiquement après 24h.

### 2.9 Accès à l'interface web
Ouvre un navigateur et va à l'adresse :
- `https://<IP_DU_VPS>`

Connecte-toi avec :
- **Utilisateur** : `root`
- **Mot de passe** : celui récupéré à l'étape 2.8

⚠️ **Change immédiatement le mot de passe** via l'icône de profil → *Edit Profile* → *Password*, puis active si possible l'authentification à deux facteurs (*Preferences → Account*).

### 2.10 Configuration Git côté client pour le port SSH personnalisé
Comme le SSH de GitLab tourne sur le port **2224** (et non 22), toute opération `git clone`/`push` en SSH doit en tenir compte.

**Option 1 — URL SSH explicite avec port :**
```bash
git clone ssh://git@<IP_DU_VPS>:2224/groupe/projet.git
```

**Option 2 — Configuration dans `~/.ssh/config` (méthode recommandé) mais nous ne l'avons pas appliquer ici :**
```
Host <IP_DU_VPS>
  Port 2224
  User git
```
Ensuite un simple `git clone git@<IP_DU_VPS>:groupe/projet.git` fonctionnera normalement.

### 2.11 Commandes utiles de gestion du conteneur

| Action | Commande |
|---|---|
| Voir les logs en direct | `docker compose logs -f gitlab` |
| Redémarrer GitLab | `docker compose restart gitlab` |
| Arrêter GitLab | `docker compose stop gitlab` |
| Reconfigurer après modif de `GITLAB_OMNIBUS_CONFIG` | `docker exec -it gitlab gitlab-ctl reconfigure` |
| Mettre à jour GitLab (nouvelle version) | modifier le tag d'image puis `docker compose up -d` |
| Sauvegarder | `docker exec -it gitlab gitlab-backup create` |

### 2.12 Créer le projet Git du pipeline
Une fois connecté à l'interface :
1. Clique sur **New project** → **Create blank project**
2. Nomme-le (pour l'exemple MyAppPipe)
3. Choisis la visibilité (Private recommandé)
4. Récupère l'URL de clone (SSH ou HTTPS). Elle servira à l'étape 3 (connexion vers le serveur CI/Jenkins)


---

⬅️ **Précédent** : [Étape 1 — Connexion et préparation du VPS Hostinger](./01-vps-hostinger.md)  
➡️ **Suivant** : [Étape 3 — Serveur CI (EC2) + Jenkins + Webhook GitLab](./03-jenkins-ci-ec2.md)