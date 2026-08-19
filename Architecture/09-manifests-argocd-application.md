# Étape 9 — Manifestes Kubernetes + Application Argo CD

[⬅️ Retour au sommaire](../README.md)

---


### 9.1 Objectif
Créer les manifestes Kubernetes de l'application (Deployment + Service) dans le dépôt GitLab, puis déclarer une **Application Argo CD** qui surveille ce dépôt et synchronise automatiquement le cluster à chaque changement. C'est le cœur du GitOps : le dépôt Git devient la source de vérité de l'état désiré du cluster.

### 9.2 Création du namespace applicatif

```bash
kubectl create namespace my-app-pipe
```

### 9.3 (Manuel, hors GitOps) Secrets des variables d'environnement

⚠️ **Bonne pratique** : les secrets (clé Django, identifiants base de données) ne doivent **jamais être commités en clair dans Git**, même privé. Ces secrets sont donc créés **manuellement** dans le cluster, en dehors du dépôt synchronisé par Argo CD.



**Les secret sont creer avec cette commande**
```bash
  kubectl create secret generic
```

### 9.3bis Déploiement de PostgreSQL dans le cluster : Si vous utilisez Postgres

Contrairement au `docker-compose.yml` local où PostgreSQL est un service parmi d'autres, il doit exister comme **ressource Kubernetes à part entière** dans le cluster cible, avec stockage persistant.

**`k8s/postgres.yaml`** (à ajouter au dépôt, synchronisé par Argo CD comme le reste) :
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: my-app-pipe
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: my-app-pipe
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          envFrom:
            - secretRef:
                name: postgres-env
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "$(POSTGRES_USER)"]
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: my-app-pipe
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

> 💡 Le Service `postgres-service` fournit une résolution DNS interne automatique dans le cluster (grâce à CoreDNS) : n'importe quel pod du namespace `my-app-pipe` peut joindre PostgreSQL simplement via le nom `postgres-service`, exactement comme `db` le permettait dans Docker Compose.

```bash
kubectl get secret gitlab-registry-secret -o yaml \
  | sed 's/namespace: default/namespace: my-app-pipe/' \
  | kubectl apply -f -
```

### 9.4 Création des manifestes de l'application (Deployment + Service) dans le dépôt GitLab

Dans ton dépôt local (`MyAppPipe`), crée un dossier `k8s/` :

```bash
mkdir -p k8s
```

**`k8s/deployment.yaml`** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-pipe-app
  namespace: my-app-pipe
spec:
  replicas: 4
  selector:
    matchLabels:
      app: my-app-pipe-app
  template:
    metadata:
      labels:
        app: my-app-pipe-app
    spec:
      imagePullSecrets:
        - name: gitlab-registry-secret
      containers:
        - name: my-app-pipe-app
          image: <IP_DU_VPS>:5050/my-app-pipe:1
          ports:
            - containerPort: 8000
          envFrom:
            - secretRef:
                name: django-env
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /
              port: 8000
              httpHeaders:
                - name: Host
                  value: "<IP_DU_VPS>"
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 8000
              httpHeaders:
                - name: Host
                  value: "<IP_DU_VPS>"
            initialDelaySeconds: 20
            periodSeconds: 20
```

> `replicas: 4` correspond aux 4 conteneurs Docker illustrés dans le schéma d'architecture initial. Ajuste selon la charge réelle attendue.
>
> 📌 **Le tag `:1` de l'image est une valeur de démarrage.** Une fois le pipeline Jenkins opérationnel (étape 11), c'est lui qui **met à jour automatiquement** cette ligne à chaque build réussi  donc inutile de la modifier manuellement par la suite.

**`k8s/service.yaml`** :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-pipe-service
  namespace: my-app-pipe
spec:
  type: NodePort
  selector:
    app: my-app-pipe-app
  ports:
    - port: 8000
      targetPort: 8000
      nodePort: 30080
```

> Pas de domaine disponible, donc exposition directe en `NodePort` (port fixé à `30080`, dans la plage 30000-32767 déjà ouverte au security group/firewall).

### 9.5 Commit et push des manifestes

```bash
git add k8s/
git commit -m "Ajout des manifestes Kubernetes (PostgreSQL + Deployment + Service applicatifs)"
git push
```

### 9.6 Ouverture du port NodePort applicatif sur le firewall

```bash
ufw allow 30080/tcp
ufw status
```
(+ vérifier le pare-feu réseau hPanel Hostinger pour ce port, comme pour les précédents)

### 9.7 Création de l'Application Argo CD

**Via le CLI :**
```bash
argocd app create my-app-pipe-app \
  --repo http://<IP_DU_VPS>/my-app-pipe.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace my-app-pipe \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

**Ou via l'interface web** (**+ New App**) :
- **Application Name** : `my-app-pipe-app`
- **Project** : `default`
- **Sync Policy** : `Automatic` + coche `PRUNE RESOURCES` et `SELF HEAL`
- **Repository URL** : celle connectée à l'étape 8.11
- **Path** : `k8s`
- **Cluster URL** : `https://kubernetes.default.svc` (cluster local, celui où Argo CD lui-même tourne)
- **Namespace** : `my-app-pipe`

> `--auto-prune` supprime automatiquement les ressources retirées du dépôt Git ; `--self-heal` restaure automatiquement toute modification manuelle du cluster qui diverge du dépôt (c'est le principe central du GitOps : le cluster ne doit jamais dériver de ce qui est déclaré dans Git).

### 9.8 Vérification de la synchronisation

```bash
argocd app get my-app-pipe-app
argocd app sync my-app-pipe-app
```

Ou dans l'interface web : l'application doit passer au statut **Healthy** / **Synced**.

Vérifie les pods :
```bash
kubectl get pods -n my-app-pipe
kubectl get svc -n my-app-pipe
```

### 9.9 Test d'accès à l'application

```bash
curl http://<IP_DU_VPS>:30080/
```

Ou directement dans un navigateur : `http://<IP_DU_VPS>:30080/`

### 9.10 Test du cycle GitOps complet

1. Modifie une valeur dans `k8s/deployment.yaml` (ex. `replicas: 4` → `replicas: 5`)
2. `git commit` + `git push`
3. Observe dans Argo CD (UI ou `argocd app get my-app-pipe-app`) : le statut passe à **OutOfSync**, puis se resynchronise automatiquement (grâce à `--sync-policy automated`) en quelques dizaines de secondes
4. `kubectl get pods -n my-app-pipe` doit refléter le nouveau nombre de replicas


---

⬅️ **Précédent** : [Étape 8 — Cluster Kubernetes + Argo CD (GitOps)](./08-kubernetes-argocd.md)  
➡️ **Suivant** : [Étape 10 — Notifications Slack](./10-notifications-slack.md)