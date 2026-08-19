# Étape 11 — Finalisation : suivi des versions et notifications enrichies

[⬅️ Retour au sommaire](../README.md)

---


### 11.1 Objectif
Deux finitions pour clore le pipeline :
1. Mettre à jour **automatiquement** le tag d'image dans le manifeste Kubernetes à chaque build réussi (au lieu de le modifier à la main)
2. Enrichir la notification Slack : nom + numéro de build de l'image, liens directs (GitLab, Jenkins, rapport Trivy, Argo CD, application), mise en forme soignée (Slack Block Kit)

### 11.2 Nouveau stage "Update Kubernetes Manifest" (mise à jour automatique du tag)

Ce stage fait ce qu'on faisait manuellement à la main jusqu'ici : il modifie `k8s/deployment.yaml` pour y inscrire le tag du build qui vient d'être poussé, puis commite et pousse ce changement sur GitLab. Argo CD (grâce à `--sync-policy automated`) détecte ce commit et redéploie automatiquement. C'est le principe du **GitOps piloté par CI**.

> ⚠️ Prérequis : le credential `gitlab-clone-creds` (étape 3.12bis) doit avoir un token avec le scope **`write_repository`** en plus de `read_repository`, sinon le `git push` de ce stage échouera avec une erreur d'autorisation.

> ⚠️ **Risque de boucle infinie à corriger avant d'appliquer ce stage.** Le `git push` fait par le stage `Update Kubernetes Manifest` redéclenche le webhook GitLab → Jenkins relance aussitôt un nouveau build → qui repousse un commit → nouveau webhook... boucle sans fin. Le `Jenkinsfile` ci-dessous inclut donc un **garde-fou intégré directement au stage `Checkout` existant** (pas de stage supplémentaire dans le pipeline) : un bloc `script` juste après `checkout scm` lit le message du dernier commit et positionne `env.CI_SKIP`. Chaque stage suivant porte une condition `when { expression { env.CI_SKIP != 'true' } }` qui le fait sauter proprement si le commit contient `[ci skip]` (marqueur ajouté par `Update Kubernetes Manifest` à son propre commit).

**`Jenkinsfile` complet mis à jour :**
```groovy
pipeline {
    agent any

    environment {
        SCANNER_HOME   = tool 'sonar-scanner'
        IMAGE_NAME     = 'my-app-pipe'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        REGISTRY       = '<IP_DU_VPS>:5050'
        REGISTRY_PATH  = 'my-app-pipe'
        GITLAB_PROJECT_URL  = 'http://<IP_DU_VPS>/my-app-pipe'
        ARGOCD_APP_URL      = 'https://<IP_DU_VPS>:31667/applications/my-app-pipe-app'
        APP_URL             = 'http://<IP_DU_VPS>:30080/'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def commitMsg = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    env.CI_SKIP = commitMsg.contains('[ci skip]') ? 'true' : 'false'
                    if (env.CI_SKIP == 'true') {
                        echo "Commit '[ci skip]' détecté (poussé par le stage Update Kubernetes Manifest) ,  les stages suivants seront sautés, pas de re-boucle."
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            when { expression { env.CI_SKIP != 'true' } }
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner"
                }
            }
        }

        stage('Quality Gate') {
            when { expression { env.CI_SKIP != 'true' } }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            when { expression { env.CI_SKIP != 'true' } }
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Trivy Scan') {
            when { expression { env.CI_SKIP != 'true' } }
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
            when { expression { env.CI_SKIP != 'true' } }
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

        stage('Update Kubernetes Manifest') {
            when { expression { env.CI_SKIP != 'true' } }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'gitlab-clone-creds',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh """
                      git config user.email "jenkins@my-app-pipe.local"
                      git config user.name "Jenkins CI"
                      sed -i "s#image: .*my-app-pipe:.*#image: ${REGISTRY}/${REGISTRY_PATH}:${IMAGE_TAG}#" k8s/deployment.yaml
                      git add k8s/deployment.yaml
                      git commit -m "Deploy build #${IMAGE_TAG} [ci skip]" || echo "Aucun changement à commiter"
                      git push http://\$GIT_USER:\$GIT_PASS@<IP_DU_VPS>/my-app-pipe.git HEAD:develop
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                if (env.CI_SKIP != 'true') {
                    archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
                }
            }
        }
        success {
            script {
                if (env.CI_SKIP != 'true') {
                    withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                        sh '''
                          curl -X POST -H "Content-type: application/json" --data '{
                            "attachments": [{
                              "color": "#2eb67d",
                              "blocks": [
                                {
                                  "type": "header",
                                  "text": { "type": "plain_text", "text": "✅ Déploiement réussi ,  MyAppPipe", "emoji": true }
                                },
                                {
                                  "type": "section",
                                  "fields": [
                                    { "type": "mrkdwn", "text": "*Image :*\\nmy-app-pipe:'"${IMAGE_TAG}"'" },
                                    { "type": "mrkdwn", "text": "*Build Jenkins :*\\n#'"${IMAGE_TAG}"'" },
                                    { "type": "mrkdwn", "text": "*Branche :*\\n'"${GIT_BRANCH}"'" },
                                    { "type": "mrkdwn", "text": "*Statut :*\\n:white_check_mark: Succès" }
                                  ]
                                },
                                { "type": "divider" },
                                {
                                  "type": "actions",
                                  "elements": [
                                    { "type": "button", "action_id": "gitlab_link", "text": { "type": "plain_text", "text": "📦 GitLab" }, "url": "'"${GITLAB_PROJECT_URL}"'" },
                                    { "type": "button", "action_id": "jenkins_link", "text": { "type": "plain_text", "text": "🛠️ Jenkins" }, "url": "'"${BUILD_URL}"'" },
                                    { "type": "button", "action_id": "trivy_link", "text": { "type": "plain_text", "text": "🔒 Rapport Trivy" }, "url": "'"${BUILD_URL}"'artifact/trivy-report.txt" },
                                    { "type": "button", "action_id": "argocd_link", "text": { "type": "plain_text", "text": "🚀 Argo CD" }, "url": "'"${ARGOCD_APP_URL}"'" },
                                    { "type": "button", "action_id": "app_link", "text": { "type": "plain_text", "text": "🌐 Application" }, "url": "'"${APP_URL}"'" }
                                  ]
                                }
                              ]
                            }]
                          }' "$SLACK_WEBHOOK"
                        '''
                    }
                }
            }
        }
        failure {
            script {
                if (env.CI_SKIP != 'true') {
                    withCredentials([string(credentialsId: 'slack-webhook-url', variable: 'SLACK_WEBHOOK')]) {
                        sh '''
                          curl -X POST -H "Content-type: application/json" --data '{
                            "attachments": [{
                              "color": "#e01e5a",
                              "blocks": [
                                {
                                  "type": "header",
                                  "text": { "type": "plain_text", "text": "❌ Pipeline échoué ,  MyAppPipe", "emoji": true }
                                },
                                {
                                  "type": "section",
                                  "fields": [
                                    { "type": "mrkdwn", "text": "*Image :*\\nmy-app-pipe:'"${IMAGE_TAG}"'" },
                                    { "type": "mrkdwn", "text": "*Build Jenkins :*\\n#'"${IMAGE_TAG}"'" },
                                    { "type": "mrkdwn", "text": "*Statut :*\\n:x: Échec" }
                                  ]
                                },
                                { "type": "divider" },
                                {
                                  "type": "actions",
                                  "elements": [
                                    { "type": "button", "action_id": "gitlab_link_fail", "text": { "type": "plain_text", "text": "📦 GitLab" }, "url": "'"${GITLAB_PROJECT_URL}"'" },
                                    { "type": "button", "action_id": "jenkins_logs_link", "text": { "type": "plain_text", "text": "🛠️ Voir les logs" }, "url": "'"${BUILD_URL}"'console" }
                                  ]
                                }
                              ]
                            }]
                          }' "$SLACK_WEBHOOK"
                        '''
                    }
                }
            }
        }
    }
}
```

**Ce qui a changé par rapport à l'étape 10 :**
- Le stage **`Checkout`** existant intègre désormais un bloc `script` qui détecte les commits `[ci skip]` (poussés par Jenkins lui-même) et positionne `env.CI_SKIP`, **aucun stage supplémentaire visible** dans le pipeline, chaque stage suivant porte simplement `when { expression { env.CI_SKIP != 'true' } }` pour sauter proprement si besoin, éliminant le risque de boucle infinie
- Nouveau stage **`Update Kubernetes Manifest`** : réécrit la ligne `image:` du manifeste avec le tag exact venant d'être poussé, commit (avec le marqueur `[ci skip]`) + push automatique
- Le message Slack passe d'un simple `attachment` texte à un vrai **Block Kit** : en-tête (`header`), champs structurés (`section` avec `fields` ,  image, build, branche, statut), séparateur (`divider`), et une **rangée de boutons cliquables** (`actions`) pointant vers GitLab, Jenkins, le rapport Trivy, Argo CD et l'application
- Les blocs `post` sont enveloppés dans `script { if (env.CI_SKIP != 'true') { ... } }` pour ne **pas** notifier Slack ni archiver d'artefact sur les builds "fantômes" arrêtés par le garde-fou
- La couleur de la barre latérale (`"color"`) reste verte en succès / rouge en échec, combinée aux blocs pour un rendu à la fois structuré et visuellement clair

> 💡 **Pourquoi `sh '''...'''` (triple guillemets simples) plutôt que `sh """..."""`** pour les blocs Slack : le JSON contient déjà beaucoup d'accolades et de guillemets doubles ; les triple guillemets simples Groovy ne font **aucune interpolation côté Groovy**, le texte est envoyé tel quel au shell. Les variables sont donc résolues **côté shell**, via la syntaxe `'"${VAR}"'` (fermeture du `'` shell → `"$VAR"` interprété par le shell → réouverture du `'`) ,  et non côté Groovy. ⚠️ Piège rencontré en pratique : `${env.GIT_BRANCH}` ou `${env.BUILD_URL}` ne fonctionnent **pas** ici car un point (`.`) est invalide dans un nom de variable shell (`Bad substitution`) ,  il faut utiliser `${GIT_BRANCH}` et `${BUILD_URL}` (sans le préfixe `env.`), ces variables étant déjà exportées nativement par Jenkins vers l'environnement du `sh`.

### 11.4 Accès direct à l'application pour vérifier les changements

L'application est exposée en permanence à cette adresse (aucune action nécessaire, déjà en place depuis l'étape 9) :

```
http://<IP_DU_VPS>:30080/
```

Après chaque déploiement réussi, ce lien est aussi inclus directement dans la notification Slack (bouton **🌐 Application**) ,  un simple clic depuis Slack permet de vérifier immédiatement que les modifications ont bien été prises en compte, sans avoir à retaper l'URL.

> 💡 Pour être certain de voir la nouvelle version et non une page mise en cache par le navigateur, un rafraîchissement forcé (`Ctrl+Maj+R` / `Cmd+Maj+R`) est recommandé juste après un déploiement.

### 11.5 Commit et test final

```bash
git add Jenkinsfile
git commit -m "Etape 11: mise a jour auto du manifeste + notifications Slack enrichies"
git push
```

Vérifie dans l'ordre :
1. Le nouveau stage **Update Kubernetes Manifest** réussit (le `git push` interne au stage doit aboutir, pas d'erreur d'autorisation)
2. Le message Slack reçu affiche bien le nom de l'image + numéro de build, et les 5 boutons fonctionnent
3. Le tag affiché dans Argo CD (étape 11.4) correspond bien au numéro de build Jenkins qui vient de tourner


---

⬅️ **Précédent** : [Étape 10 ,  Notifications Slack](./10-notifications-slack.md)  
➡️ **Suivant** : [Retour au sommaire](../README.md)

LeTrex