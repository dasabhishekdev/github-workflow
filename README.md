#  CI/CD for Docker Production Deployments Using GitHub Actions

Automating deployments ensures that your production releases are consistent, secure, and fast. In this blog, we’ll walk through a GitHub Actions workflow that deploys a Dockerized application to a remote server, with several key **DevOps best practices**.


> ✅ **Note**: This process is designed for applications you want to **deploy and run as containers on a virtual machine (VM)**.  
> The app will be copied to the VM, built using Docker Compose, and then run as a containerized service on that machine.

---

##  Best Practices Implemented

1. **Environment-Specific Compose Files**  
   Use `docker-compose.prod.yaml`, `docker-compose.dev.yaml`, etc., to isolate configurations per environment.

2. **Temporary Rename During Build**  
   Rename `docker-compose.prod.yaml` to `docker-compose.yaml` only during CI to avoid hardcoding or committing it.

3. **Post-Copy Cleanup**  
   Remove sensitive config files like `docker-compose.yaml` after copying and running, both locally and on the server.

4. **Docker System Prune**  
   Run `docker system prune -f` before and after `docker compose up` to clean up unused images, networks, and volumes.

5. **Secure Credential Handling**  
   Use GitHub Secrets to store SSH credentials instead of hardcoding them into the workflow.

6. **No Cache Build**  
   Use `--no-cache` during builds to ensure fresh layers are built, avoiding stale behavior.

7. **Use `.dockerignore`**  
   Prevent unnecessary files (like `.git`, `node_modules`, etc.) from being copied and bloating Docker images.

8. **Use `docker compose down` Before Up**  
   Shuts down previous containers cleanly to avoid port conflicts and leftover states.

9. **Logging Deployments**  
   Pipe the deployment output to `deploy.log` for traceability and debugging.

10. **Use Latest Action Versions**  
   Keep action versions updated (e.g., `actions/checkout@v4`) to use the latest security and feature updates.

---

## 🔧 GitHub Actions Workflow (`.github/workflows/ams-production-deploy.yml`)

```yaml
name: ams-production-deploy

on:
  push:
    branches:
      - production

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Rename Docker Compose file for production
        run: mv docker-compose.prod.yaml docker-compose.yaml

      - name: Copy project files to remote server
        uses: appleboy/scp-action@v0.1.4
        with:
          source: "."
          target: "/home/services/ams"
          host: ${{ secrets.PROD_HOST_NAME }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          port: ${{ secrets.PROD_PORT }}
          passphrase: ${{ secrets.PROD_PASSPHRASE }}

      - name: Cleanup local compose file (security hygiene)
        run: rm -f docker-compose.yaml

      - name: SSH into server and deploy
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PROD_HOST_NAME }}
          port: ${{ secrets.PROD_PORT }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          passphrase: ${{ secrets.PROD_PASSPHRASE }}
          script: |
            echo "Moving to deployment directory"
            cd /home/services/ams

            echo "Clean previous containers and resources"
            docker compose down
            docker system prune -f

            echo "Rebuild and launch with no cache"
            docker compose build --no-cache
            docker compose up -d > deploy.log 2>&1

            echo "Post-deployment cleanup"
            docker system prune -f
            rm -f docker-compose.yaml
