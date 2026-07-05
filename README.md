1. Credential trong Jenkins:
- dockerhub-credentials — username/password Docker Hub
- gitops-credentials — GitHub token/USername+password để push yas-gitops
2. Pipeline job trỏ vào yas/Jenkinsfile trên source repo
3. Webhook GitHub → Jenkins (push events + tag push events)