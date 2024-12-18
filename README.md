# Nextjs Continuous deployment

- Continuous deployment of nextjs to ec2 using docker, github actions
- This is a [Next.js](https://nextjs.org) project bootstrapped with [`create-next-app`](https://nextjs.org/docs/app/api-reference/cli/create-next-app).

## Description

### server spec

- hardware: aws ec2(t2.micro instance)
- os: ubuntu-noble-24.04 x86_64



### .dockerignore의 용량 차이

- node18 버전 기준 210.67MB -> 190.91MB (19.76MB 감소)
- 패키지들이 많아질 경우 차이는 더 커질 수 있음



### Reference

- install docker on ubuntu
  - https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository
- github actions
  - build and push docker image
    - https://github.com/docker/build-push-action
  - connect ssh
    - https://github.com/appleboy/ssh-action



## Github Actions

### Docker

```yaml
name: deploy to docker hub
run-name: Deploy to Docker Hub

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          # 리포지토리 이름을 태그로 사용
          tags: ${{ vars.DOCKERHUB_USERNAME }}/next-docker-githubaction:latest

  deploy:
    name: Deploy to Server
    runs-on: ubuntu-latest
    needs: build-and-push

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: 22
          script: |
            sudo apt-get update && sudo apt upgrade -y
            if ! command -v docker &> /dev/null;
            then
              echo "Docker not found, installing Docker..."
              sudo apt-get update
              sudo apt-get install ca-certificates curl
              sudo install -m 0755 -d /etc/apt/keyrings
              sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
              sudo chmod a+r /etc/apt/keyrings/docker.asc
              echo \
                "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
                $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
                sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
              sudo apt-get update
              sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
              sudo systemctl enable docker
              sudo systemctl start docker
            fi
            sudo docker pull ${{ vars.DOCKERHUB_USERNAME }}/next-docker-githubaction:latest
            sudo docker stop next-docker-githubaction || true
            sudo docker rm next-docker-githubaction || true
            sudo docker run -d -p 80:3000 --name next-docker-githubaction ${{ vars.DOCKERHUB_USERNAME }}/next-docker-githubaction:latest

```



### With Nginx

```yaml
name: deploy to docker hub
run-name: Deploy to Docker Hub

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          # 리포지토리 이름을 태그로 사용
          tags: ${{ vars.DOCKERHUB_USERNAME }}/next-docker-githubaction:latest

  deploy:
    name: Deploy to Server
    runs-on: ubuntu-latest
    needs: build-and-push

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: 22
          script: |
            sudo apt-get update && sudo apt upgrade -y
            if ! command -v docker &> /dev/null;
            then
              echo "Docker not found, installing Docker..."
              sudo apt-get update
              sudo apt-get install ca-certificates curl
              sudo install -m 0755 -d /etc/apt/keyrings
              sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
              sudo chmod a+r /etc/apt/keyrings/docker.asc
              echo \
                "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
                $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
                sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
              sudo apt-get update
              sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
              sudo systemctl enable docker
              sudo systemctl start docker
            fi
            if ! command -v nginx &> /dev/null;
            then
              echo "Nginx not found, installing Nginx..."
              sudo apt-get update
              sudo apt-get install nginx -y
              sudo systemctl stop nginx
              sudo ufw allow 'Nginx Full'
              sudo ufw allow 'OpenSSH'
              sudo ufw enable y
            sudo bash -c "cat > /etc/nginx/sites-available/default" <<EOL
            server {
              listen 80;
              server_name _;
              location / {
                proxy_pass http://localhost:3000;
                proxy_http_version 1.1;
                proxy_set_header Upgrade \$http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host \$host;
                proxy_cache_bypass \$http_upgrade;
              }
            }
            EOL
              sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default
              sudo systemctl enable nginx
              sudo systemctl restart nginx
            fi
            sudo docker pull ${{ vars.DOCKERHUB_USERNAME }}/next-docker-githubaction:latest
            sudo docker stop next-docker-githubaction || true
            sudo docker rm next-docker-githubaction || true
            sudo docker run -d -p 3000:3000 --name next-docker-githubaction ${{ vars.DOCKERHUB_USERNAME }}/next-docker-githubaction:latest

```

