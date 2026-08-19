# Étape 1 — Connexion et préparation du VPS Hostinger

[⬅️ Retour au sommaire](../README.md)

---


### 1.1 Objectif
Préparer le VPS Hostinger qui va héberger l'instance GitLab (rôle : gestion de code source + déclencheur de pipeline).

### 1.2 Prérequis
| Élément | Recommandation |
|---|---|
| VPS actif | Hostinger VPS (KVM) |
| OS | Ubuntu 22.04 LTS (recommandé pour GitLab Omnibus) |
| RAM minimale | 4 Go (8 Go recommandé pour GitLab CE) |
| CPU | 2 vCPU minimum |
| Stockage | 20 Go minimum |
| Accès | Adresse IP publique + accès SSH (clé ou mot de passe) |

> ⚠️ Si ton VPS a moins de 4 Go de RAM, GitLab risque de ne pas démarrer correctement (Puma/Sidekiq). Dans ce cas on pourra ajouter un swap ou limiter les workers.

### 1.3 Où récupérer les informations de connexion
1. Va sur **hpanel.hostinger.com** (panneau Hostinger)
2. Section **VPS** → sélectionne ton serveur
3. Onglet **Vue d'ensemble** : récupère l'**adresse IP**
4. Onglet **Paramètres SSH** (ou "Accès SSH") : récupère le mot de passe root ou configure ta clé SSH publique

### 1.4 Connexion en SSH
Depuis ton terminal local (Linux/Mac) ou PowerShell/WSL (Windows) :

```bash
ssh root@<IP_DU_VPS>
```

Si c'est la première connexion, accepte l'empreinte de l'hôte (`yes`).

### 1.5 Mise à jour du système
Une fois connecté :

```bash
apt update && apt upgrade -y
```

### 1.6 Installation des dépendances nécessaires à GitLab

```bash
apt install -y curl openssh-server ca-certificates tzdata perl postfix
```

### 1.7 Installation de Docker et Docker Compose
GitLab va être déployé en conteneur Docker plutôt qu'en installation native (Omnibus). Il faut donc installer Docker Engine et le plugin Docker Compose sur le VPS.

**Suppression d'éventuels anciens paquets Docker :**
```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do apt remove -y $pkg; done
```

**Ajout du dépôt officiel Docker :**
```bash
apt install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
```

**Installation de Docker Engine + Compose plugin :**
```bash
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Vérification :**
```bash
docker --version
docker compose version
```

**Activation au démarrage du VPS :**
```bash
systemctl enable docker
systemctl start docker
```

---

⬅️ **Précédent** : [Architecture générale](./00-architecture-generale.md)  
➡️ **Suivant** : [Étape 2 — Déploiement de GitLab CE en conteneur Docker](./02-gitlab-docker.md)