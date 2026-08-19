# Étape 8 — Cluster Kubernetes + Argo CD (GitOps)

[⬅️ Retour au sommaire](../README.md)

---


### 8.1 Objectif
Provisionner le second serveur EC2 qui hébergera le **cluster Kubernetes**, puis y installer **Argo CD** (Argo CD tourne lui-même comme une application Kubernetes, à l'intérieur du cluster qu'il pilote). Argo CD surveillera ensuite le dépôt GitLab et synchronisera automatiquement les déploiements.

> 💡 **Choix technique : k3s plutôt que kubeadm.** Pour un projet de cette taille, [k3s](https://k3s.io/) (distribution Kubernetes légère de Rancher) s'installe en une seule commande, consomme beaucoup moins de ressources que kubeadm, et reste 100% compatible avec l'API Kubernetes standard (donc avec Argo CD, kubectl, etc.). C'est un bon choix pour notre démonstration.

### 8.2 Création de l'instance EC2 pour Kubernetes

1. Console AWS → **EC2** → **Launch Instance**
2. Configure :

| Paramètre | Valeur recommandée |
|---|---|
| Nom | `k8s-node` |
| AMI | Ubuntu Server 22.04 LTS (64-bit x86) |
| Type d'instance | `t3.medium` (2 vCPU / 4 Go RAM) minimum |
| Paire de clés | Réutilise `ci-server-key`, ou crée-en une nouvelle (`k8s-node-key`) |
| Stockage | 30 Go minimum (gp3) |

3. **Groupe de sécurité** — crée-en un nouveau (`k8s-node-sg`) avec ces règles :

| Type | Port | Source |
|---|---|---|
| SSH | 22 | Ton IP publique |
| Custom TCP | 6443 | Ton IP publique (API Kubernetes, pour kubectl à distance si besoin) |
| Custom TCP | 30000–32767 | Ton IP publique (plage NodePort, pour exposer Argo CD et l'application) |

4. **Launch Instance**, puis note l'**adresse IP publique** une fois l'état `running`

### 8.3 Connexion SSH et installation de k3s

```bash
ssh -i k8s-node-key.pem ubuntu@<IP_K8S>
sudo apt update && sudo apt upgrade -y
curl -sfL https://get.k3s.io | sh -
```

**Vérification :**
```bash
sudo k3s kubectl get nodes
```
Le nœud doit apparaître avec le statut `Ready`.

**Simplifie l'usage de `kubectl`** (évite de préfixer `sudo k3s` à chaque commande) :
```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
kubectl get nodes
```

### 8.4 Autoriser l'accès au registre GitLab (HTTP) depuis containerd

⚠️ Point important : k3s utilise **containerd**, pas le démon Docker classique, le réglage `insecure-registries` de `/etc/docker/daemon.json` (vu aux étapes 3 et 7) **ne s'applique pas ici**. Il faut déclarer le registre HTTP dans la configuration de containerd propre à k3s.

```bash
sudo nano /etc/rancher/k3s/registries.yaml
```

```yaml
mirrors:
  "<IP_DU_VPS>:5050":
    endpoint:
      - "http://<IP_DU_VPS>:5050"
configs:
  "<IP_DU_VPS>:5050":
    tls:
      insecure_skip_verify: true
```

Redémarre k3s pour appliquer :
```bash
sudo systemctl restart k3s
kubectl get nodes
```

### 8.5 Création du secret Kubernetes pour l'authentification au registre

Le registre GitLab est protégé par le Deploy Token (étape 7.6) donc Kubernetes doit s'authentifier pour tirer (`pull`) les images :

```bash
kubectl create secret docker-registry gitlab-registry-secret \
  --docker-server=<IP_DU_VPS>:5050 \
  --docker-username=<username_du_deploy_token> \
  --docker-password='<deploy_token>' \
  --docker-email=noreply@my-app.local
```

> Ce secret sera référencé dans le manifeste de déploiement de l'application (`imagePullSecrets`) à l'étape 9.

### 8.6 Installation d'Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Attends que tous les pods soient prêts :
```bash
kubectl get pods -n argocd -w
```
(`Ctrl+C` une fois que tous affichent `Running`/`Completed`)

### 8.7 Exposition de l'interface Argo CD (NodePort)

Par défaut, le service `argocd-server` est en `ClusterIP` (accessible uniquement depuis l'intérieur du cluster). Comme on n'a pas d'Ingress/domaine, on l'expose en `NodePort` :

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
kubectl get svc argocd-server -n argocd
```

Note le port attribué pour `443` (colonne `PORT(S)`, format `443:XXXXX/TCP`), c'est le port à utiliser dans l'URL.

### 8.8 Récupération du mot de passe admin initial

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### 8.9 Connexion à l'interface Argo CD

Ouvre un navigateur : `https://<IP_K8S>:<NodePort_noté_à_l'étape_8.7>`

> Si ton navigateur affichera un avertissement de certificat (Argo CD génère un certificat auto-signé par défaut), c'est normal ici en l'absence de domaine, clique sur **Advanced → Continue**.

Connecte-toi :
- **Username** : `admin`
- **Password** : celui récupéré à l'étape 8.8

⚠️ **Change le mot de passe admin** immédiatement après la première connexion (menu **User Info** → **Update Password**).

### 8.10 (Optionnel) Installation du CLI Argo CD
Pratique pour les commandes en ligne de commande plutôt que l'interface web :

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
argocd version --client
```

### 8.11 Connexion d'Argo CD au dépôt GitLab

Argo CD a besoin d'accéder au dépôt Git contenant les manifestes Kubernetes de l'application (on créera ces manifestes à l'étape 9). Comme le dépôt est privé et en HTTP (pas de domaine), utilisons le token créé à l'étape 3.9 (`jenkins-integration`, scope `read_repository`) ou crée-en un dédié en lecture seule.

**Via le CLI** (depuis l'EC2 `k8s-node`, après login CLI) :
```bash
argocd login <IP_K8S>:<NodePort> --username admin --password '<mot_de_passe_admin>' --insecure

argocd repo add http://<IP_DU_VPS>/MyAppPipe.git \
  --username <ton_username_gitlab> \
  --password '<token_read_repository>'
```

**Ou via l'interface web** : **Settings** → **Repositories** → **Connect Repo** → renseigne l'URL HTTPS/HTTP, le username et le token.


---

⬅️ **Précédent** : [Étape 7 — Registre d'images Docker intégré à GitLab (Container Registry)](./07-container-registry-gitlab.md)  
➡️ **Suivant** : [Étape 9 — Manifestes Kubernetes + Application Argo CD](./09-manifests-argocd-application.md)