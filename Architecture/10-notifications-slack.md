# Étape 10 — Notifications Slack

[⬅️ Retour au sommaire](../README.md)

---


### 10.1 Objectif
Notifier automatiquement un canal Slack du **statut de chaque exécution du pipeline** (succès ou échec), pour que l'équipe soit informée sans avoir à surveiller Jenkins en continu.

> 💡 **Choix technique : Incoming Webhook plutôt que le plugin Slack Notification.** Un Webhook entrant Slack + un simple appel `curl` en Groovy évite de dépendre d'un plugin Jenkins supplémentaire (gestion de version, configuration OAuth complexe) c'est plus simple à documenter et à maintenir pour ce projet.

### 10.2 Création d'un Incoming Webhook Slack

1. Va sur **[api.slack.com/apps](https://api.slack.com/apps)** et connecte-toi à ton compte Slack
2. **Create New App** → **From scratch**
   - App Name : `Jenkins CI/CD - My APP PIPE`
   - Workspace : sélectionne ton workspace Slack
3. Dans le menu de gauche de l'app créée : **Incoming Webhooks**
4. Active le toggle **Activate Incoming Webhooks**
5. En bas de page : **Add New Webhook to Workspace**
6. Choisis le canal cible (ex. `#cicd-test-app`, à créer au préalable si besoin) → **Allow**
7. Une **Webhook URL** est générée, au format :
   ```
   https://hooks.slack.com/services/T000000/B000000/XXXXXXXXXXXXXXXXXXXXXXXX
   ```
   **Copie-la immédiatement.**

### 10.3 Enregistrement du Webhook dans les Credentials Jenkins

1. **Manage Jenkins** → **Credentials** → **System** → **Global credentials** → **Add Credentials**
2. Type : **Secret text**
3. Secret : colle l'URL du webhook (étape 10.2)
4. ID : `slack-webhook-url`
5. **Create**

### 10.4 Ajout des notifications dans le `Jenkinsfile`

Ajoute un bloc `post` avec des étapes `success`/`failure` qui envoient un message formaté à Slack :

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
        success {
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                sh """
                  curl -X POST -H 'Content-type: application/json' \
                  --data '{
                    "attachments": [{
                      "color": "#36a64f",
                      "title": "✅ Pipeline réussi — ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                      "text": "Image poussée : ${REGISTRY}/${REGISTRY_PATH}:${IMAGE_TAG}\\nVoir le build : ${env.BUILD_URL}"
                    }]
                  }' "\$SLACK_WEBHOOK"
                """
            }
        }
        failure {
            withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                sh """
                  curl -X POST -H 'Content-type: application/json' \
                  --data '{
                    "attachments": [{
                      "color": "#e01e5a",
                      "title": "❌ Pipeline échoué — ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                      "text": "Étape en échec — voir les logs : ${env.BUILD_URL}console"
                    }]
                  }' "\$SLACK_WEBHOOK"
                """
            }
        }
    }
}
```

**Points clés :**
- `post { success { ... } }` et `post { failure { ... } }` s'exécutent respectivement uniquement si le pipeline entier a réussi ou échoué (peu importe à quel stage l'échec s'est produit, Quality Gate, Trivy, etc.)
- `withCredentials` protège l'URL du webhook (jamais affichée en clair dans les logs Jenkins)
- Le format `attachments` avec `color` affiche une barre latérale colorée dans Slack (vert pour succès, rouge pour échec), plus lisible qu'un simple message texte
- Le lien `${env.BUILD_URL}console` permet d'accéder directement aux logs complets depuis Slack en cas d'échec

### 10.5 Commit et test

```bash
git add Jenkinsfile
git commit -m "Ajout des notifications Slack (succès/échec)"
git push
```

Vérifie dans le canal Slack configuré (`#cicd-test-app`) que le message apparaît après le prochain build.

**Test du cas d'échec** (optionnel, pour valider les deux chemins) : casse temporairement une étape (ex. modifie une valeur du `Jenkinsfile` pour provoquer une erreur), pousse, observe la notification rouge, puis annule le changement (`git revert`).


---

⬅️ **Précédent** : [Étape 9 — Manifestes Kubernetes + Application Argo CD](./09-manifests-argocd-application.md)  
➡️ **Suivant** : [Étape 11 — Finalisation : suivi des versions et notifications enrichies](./11-finalisation-pipeline.md)