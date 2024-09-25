# Deploying a Node.js Application with MongoDB in Docker on Timeweb VPS

This tutorial describes the process of deploying a Node.js application using MongoDB in a Docker container on a Timeweb Virtual Private Server (VPS).

## Prerequisites

- Timeweb account
- VPS on Timeweb with installed OS (preferably Ubuntu)
- Docker installed on your VPS
- Node.js application with MongoDB
- GitHub repository with your application

## Step 1: Preparing the Dockerfile

1. In the root directory of your project, create a `Dockerfile`:

```Dockerfile
# Base Node.js image
FROM node:18-alpine

# Create working directory in the container
WORKDIR /usr/src/app

# Copy package.json and package-lock.json (if exists)
COPY package*.json ./

# Copy prisma schema file (if it exists)
COPY prisma ./prisma

# Install dependencies without running scripts
RUN npm install --ignore-scripts

# Generate Prisma client
RUN npx prisma generate

# Copy application source code
COPY . .

# Build the application
RUN npm run build

# Expose ports
EXPOSE 3000 8877

# Run the application
CMD ["node", "dist/main"]
```

2. Create a `.dockerignore` file:

```
node_modules
npm-debug.log
```

## Step 2: Setting up GitHub Actions

1. In your repository, create a `.github/workflows/` directory
2. Create a `docker-publish.yml` file in this directory:

```yaml
name: Docker

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push-image:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Log in to the Container registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

## Step 3: Configuring VPS on Timeweb

1. Connect to your VPS via SSH:

   ```
   ssh root@your_server_ip
   ```

2. Install Docker (if not already installed):

   ```bash
   sudo apt update
   sudo apt install apt-transport-https ca-certificates curl software-properties-common
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
   sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
   sudo apt update
   sudo apt install docker-ce
   ```

3. Verify Docker installation:
   ```bash
   sudo systemctl status docker
   ```

## Step 4: Deploying the Application

1. Pull and run the Docker container:

   ```bash
   docker pull ghcr.io/your-username/your-repo:main
   docker run -d -p 3000:3000 -p 8877:8877 -e DATABASE_URL="your_mongodb_connection_string" ghcr.io/your-username/your-repo:main
   ```

   Replace `your_mongodb_connection_string` with the actual URL of your MongoDB database.

   #### Note the ports:

   - Port 3000 is used for your application's API
   - Port 8877 is used for API documentation (Swagger)

2. Check that the container is running:
   ```bash
   docker ps
   ```

## Step 5: Updating the Application

When you make changes to your repository and want to update the application on the VPS, follow these steps:

1. Connect to your VPS via SSH:

   ```
   ssh root@your_server_ip
   ```

2. Stop the current container:

   ```bash
   docker stop $(docker ps -q --filter ancestor=ghcr.io/your-username/your-repo:main)
   ```

3. Remove the old container:

   ```bash
   docker rm $(docker ps -aq --filter ancestor=ghcr.io/your-username/your-repo:main)
   ```

4. Get the latest image version:

   ```bash
   docker pull ghcr.io/your-username/your-repo:main
   ```

5. Run the new container:

   ```bash
   docker run -d -p 3000:3000 -p 8877:8877 -e DATABASE_URL="your_mongodb_connection_string" ghcr.io/your-username/your-repo:main
   ```

6. Check that the new container is running:
   ```bash
   docker ps
   ```

For automation, you can create an update script:

1. Create a file `update_container.sh`:

   ```bash
   #!/bin/bash
   docker stop $(docker ps -q --filter ancestor=ghcr.io/your-username/your-repo:main)
   docker rm $(docker ps -aq --filter ancestor=ghcr.io/your-username/your-repo:main)
   docker pull ghcr.io/your-username/your-repo:main
   docker run -d -p 3000:3000 -p 8877:8877 -e DATABASE_URL="your_mongodb_connection_string" ghcr.io/your-username/your-repo:main
   ```

2. Make the script executable:

   ```bash
   chmod +x update_container.sh
   ```

3. Run the script to update:
   ```bash
   ./update_container.sh
   ```

## Step 6: Setting up Nginx (optional)

If you want to use a domain name and HTTPS, you need to set up Nginx as a reverse proxy.

1. Install Nginx:

   ```bash
   sudo apt install nginx
   ```

2. Create a configuration file for your domain:

   ```bash
   sudo nano /etc/nginx/sites-available/your-domain.com
   ```

3. Add the following configuration:

   ```nginx
    server {
        listen 80;
        server_name your-domain.com www.your-domain.com;

        location /api {
            proxy_pass http://localhost:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }

        location /docs {
            proxy_pass http://localhost:8877;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
   ```

## Step 7: Setting up SSL with Certbot (optional)

For security, it is recommended to set up HTTPS. Certbot allows you to easily obtain and install a free SSL certificate from Let's Encrypt.

1. Install Certbot and the Nginx plugin:

   ```bash
   sudo apt install certbot python3-certbot-nginx
   ```

2. Obtain an SSL certificate:

   ```bash
   sudo certbot --nginx -d your-domain.com -d www.your-domain.com
   ```

3. Follow the Certbot instructions. When asked about redirecting HTTP to HTTPS, select the option to automatically redirect (usually option 2).

4. After successfully installing Certbot, it will automatically update the Nginx configuration. Check the updated configuration file:

   ```bash
   sudo nano /etc/nginx/sites-available/your-domain.com
   ```

5. Ensure the configuration looks like this:

   ```nginx
   server {
       server_name your-domain.com www.your-domain.com;

       location /api {
           proxy_pass http://localhost:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }

       location /docs {
           proxy_pass http://localhost:8877;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }

       listen 443 ssl; # managed by Certbot
       ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem; # managed by Certbot
       ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem; # managed by Certbot
       include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
       ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
   }

   server {
       if ($host = www.your-domain.com) {
           return 301 https://$host$request_uri;
       } # managed by Certbot

       if ($host = your-domain.com) {
           return 301 https://$host$request_uri;
       } # managed by Certbot

       listen 80;
       server_name your-domain.com www.your-domain.com;
       return 404; # managed by Certbot
   }
   ```

6. If everything looks correct, restart Nginx:
   ```bash
   sudo systemctl restart nginx
   ```

Now your application should be available via HTTPS:

- API: `https://your-domain.com/api`
- API Documentation: `https://your-domain.com/docs`

Certbot will automatically set up certificate renewal. You can check and update the automatic renewal settings using the command:

## Conclusion

Now your Node.js application with MongoDB should be deployed in a Docker container on Timeweb VPS, accessible through the configured ports 3000 and 8877, and easily updatable when changes are made to the repository.

Don't forget to regularly check and update your application, and set up monitoring to ensure stable operation.
