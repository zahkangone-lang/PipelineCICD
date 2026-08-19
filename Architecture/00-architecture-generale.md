# Architecture générale

---


Le pipeline mis en place suit le flux suivant :

1. Le développeur pousse son code
2. Vers un serveur **GitLab** auto-hébergé sur un **VPS Hostinger**
3. GitLab déclenche un pipeline sur un serveur CI hébergé sur **Amazon EC2**
4. **SonarQube** analyse la qualité et la sécurité du code
5. **Jenkins** orchestre le **build** de l'application
6. **Trivy** scanne l'image Docker générée à la recherche de vulnérabilités
7. L'image validée est poussée vers un **registre d'images Docker**
8. **Argo CD** détecte la nouvelle image et déclenche le déploiement (GitOps)
9. Argo CD déploie vers un cluster **Kubernetes** (second EC2)
10. Les conteneurs **Docker** tournent dans les pods Kubernetes
11. Une notification de statut est envoyée sur **Slack**

---


---

⬅️ **Précédent** : [Sommaire du projet](../README.md)  
➡️ **Suivant** : [Étape 1 — Connexion et préparation du VPS Hostinger](./01-vps-hostinger.md)