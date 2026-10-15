# GitHub Actions CI/CD Pipeline

Automated CI/CD pipeline using GitHub Actions — runs tests, builds Docker image, and deploys to AWS EC2 on every push to main.

> Pattern used in production at Navat Technology

---

## Pipeline Flow

```
Push to main
    │
    ▼
GitHub Actions triggered
    │
    ├── 1. Run tests
    ├── 2. Build Docker image
    ├── 3. Push to Docker Hub
    └── 4. SSH into EC2 → pull image → restart service
```

---

## Workflow File

Create `.github/workflows/deploy.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v3

      - name: Build Docker image
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/myapp:latest .

      - name: Login to Docker Hub
        run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

      - name: Push Docker image
        run: docker push ${{ secrets.DOCKER_USERNAME }}/myapp:latest

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v0.1.10
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            docker pull ${{ secrets.DOCKER_USERNAME }}/myapp:latest
            docker stop myapp || true
            docker rm myapp || true
            docker run -d --name myapp -p 3000:3000 --env-file /home/ubuntu/.env ${{ secrets.DOCKER_USERNAME }}/myapp:latest
```

---

## GitHub Secrets Setup

Go to repo → Settings → Secrets and variables → Actions → New repository secret

| Secret | Value |
|--------|-------|
| `DOCKER_USERNAME` | Your Docker Hub username |
| `DOCKER_PASSWORD` | Your Docker Hub password |
| `EC2_HOST` | EC2 public IP address |
| `EC2_SSH_KEY` | Contents of your `.pem` key file |

---

## Dockerfile

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

---

## Key Concepts Demonstrated

- **Branch-based deployment** — only deploys on push to `main`, not on PRs
- **Job dependencies** — deploy job only runs if test job passes (`needs: test`)
- **Secrets management** — all credentials in GitHub Secrets, never in code
- **Least-privilege IAM** — EC2 deploy user has only the permissions it needs
- **Docker** — consistent environment between dev and production

---

## Author

**Abishek S** — DevOps Engineer  
[LinkedIn](https://linkedin.com/in/abishek-iam) · [GitHub](https://github.com/abisheksug)
