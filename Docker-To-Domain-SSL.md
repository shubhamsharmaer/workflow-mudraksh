# Workflow for Deploying Project to Live Domain with SSL

This guide explains how to deploy a web application (a Vite/React frontend and a backend API) from local development to production using GitHub, Docker, Cloudflare, and Let's Encrypt SSL certificates. It includes step-by-step instructions, commands, and troubleshooting tips for common issues encountered along the way.


## Table of Contents

| S.NO | Section                                                                  | Description                                                       |
|------|--------------------------------------------------------------------------|-------------------------------------------------------------------|
| 1    | [Prerequisites](#prerequisites)                                          | Tools, accounts, and server prerequisites                         |
| 2    | [Local Development Setup](#local-development-setup)                      | Setting up your local environment                                 |
| 3    | [Dockerizing the Application](#dockerizing-the-application)              | Containerizing your application with Docker                       |
| 4    | [Deploying on a Public IP](#deploying-on-a-public-ip)        | Deploying your application on a server with a public IP           |
| 5    | [Configuring Cloudflare DNS](#configuring-cloudflare-dns)                  | Setting up Cloudflare DNS records                                 |
| 6    | [Setting Up Nginx Reverse Proxy](#setting-up-nginx-reverse-proxy) | Configuring Nginx as your reverse proxy                           |
| 7    | [Obtaining and Deploying SSL Certificates](#obtaining-and-deploying-ssl-certificates) | Using Certbot and Let's Encrypt for SSL                           |
| 8    | [Common Issues and Troubleshooting](#common-issues-and-troubleshooting)    | Tips for troubleshooting common deployment issues                 |
| 9    | [Summary](#summary)                                                      | Recap of the deployment workflow                                  |


## Prerequisites

- **Nginx:**  Must be installed on remote SSH.
- **Access/Accounts:** GitHub, Cloudflare.
- **Remote SSH:** A cloud or VPS server (e.g., Ubuntu 20.04) with a public IP.
- **Certbot:** Installed on your server if you’re obtaining SSL directly (or use a companion container).



## Local Development Setup

1. **Clone Your Repository:**
   ```bash
   git clone https://github.com/yourusername/yourproject.git
   cd yourproject

2. **Configure Environment Files:**
   For .env 
   ```bash
   VITE_CLIENT_AUTH_API_URL=http://localhost:7135 ## localhost can be Dabba or Public IP (124.12.135.128)
   ```

   Create .env.production 
   ```bash
   VITE_CLIENT_AUTH_API_URL=http://'SUB-DOMAIN-NAME'.quanttiphy.com/api ## SUB-DOMAIN your CloudFlare DNS Name (e.g; 'client' in client.quanttiphy.com)
   ```


## Dockerizing the Application

#### Create files at root: `docker-compose.yaml`, `Dockerfile`, `nginx.conf` 

- docker-compose.yaml
    ```bash
    version: "3.8"

    services:
    vite-react-app:
        container_name: client-dashboard-container
        build:
        context: .
        dockerfile: Dockerfile
        ports:
        - "DABBA PORT:80"
        environment:
        - .env
    ```
    
- Dockerfile
    ```bash
    # Use an official Node.js image as a base
    FROM node:20-alpine AS build

    # Set the working directory inside the container
    WORKDIR /app

    # Copy package.json and package-lock.json (or yarn.lock) to the working directory
    COPY package*.json ./

    # Install dependencies
    RUN npm install

    # Copy the rest of the application code
    COPY . .

    # Build the application
    RUN npm run build

    # Create a production image
    FROM nginx:alpine AS production

    # Copy the built app from the previous stage to the NGINX web server directory
    COPY --from=build /app/dist /usr/share/nginx/html

    #copy the nginx configuration file
    COPY nginx.conf /etc/nginx/conf.d/default.conf

    # Expose port 80 for the NGINX server
    EXPOSE 80

    # Set the timezone of the container to match the host machine's timezone
    RUN ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime

    # Start NGINX when the container starts
    CMD ["nginx", "-g", "daemon off;"]
    ```

- nginx.conf
    ```bash
        server {
        listen 80 default_server;
        server_name _;

        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
    }
    ```

    `NOTE: During development, use .env; in production, the build process will use .env.production (make sure these files are in your build context).`


## Deploying on a Public IP

1. Open SSH and Clone project repo:
    ```bash
    git clone https://github.com/yourusername/yourproject.git
    cd yourproject
    ```
2. Run Docker Compose: 
    ```bash
    docker-compose up -d
    ```
    `Verify: Your frontend app should now be accessible locally on the server (e.g., via http://127.0.0.1:DABBA-PORT)`



## Configuring Cloudflare DNS

1. Request access from TL
2. Create DNS Records:
    - Go to dns records 
    - Add a record for `SUB-DOMAIN` (e.g; client in client.quanttiphy.com)
    - TYPE: `A` NAME: `SUB-DOMAIN` IP: `YOUR DABBA IP` PROXY MODE: `DNS ONLY`


## Setting Up Nginx Reverse Proxy

1. Open nginx in SSH:
    ```bash
    code /etc/nginx
    ```
2. Create a Custome file for serving:
     - Go to `/etc/nginx/sites_available/`
     - create with any name (e.g; quanttiphy.com)

3. Configure the file:
    ```bash
        server {
        listen 80;
        server_name `SUB-DOMAIN`.quanttiphy.com www.`SUB-DOMAIN`.quanttiphy.com;
        # Redirect HTTP to HTTPS
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name `SUB-DOMAIN`.quanttiphy.com www.`SUB-DOMAIN`.quanttiphy.com;
        
        location / {
            proxy_pass http://127.0.0.1:8156/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /api {
            proxy_pass http://127.0.0.1:7135/api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```

4. Symlinked Your above file:
    ```bash
    sudo ln -s /etc/nginx/sites_available/quanttiphy.com /etc/nginx/sites_enabled/quanttiphy.com
    ```
    
    `This will enable nginx to serve your project/files over domain`



## Obtaining and Deploying SSL Certificates:

1. Install CertBot:
    ```bash
    sudo apt install certbot python3-certbot-nginx
    ```
2. Request Certificate:
    ```bash
    sudo certbot --nginx -d `SUB-DOMAIN`.quanttiphy.com -d www.`SUB-DOMAIN`.quanttiphy.com
    ```
    `It will then ask for some info email enter them.`

3. Verify Certificate Deployment: 
    - List Certificates Issued by Certbort
    ```bash
    sudo certbot certificates
    ```
    `This command lists all the certificates managed by Certbot, along with their paths and expiration dates.`

4. Make Changes into `etc/nginx/sites_available/YOUR_CUSTOME_FILE`:
    - Open the file and paste below code inside server

    ```bash
    server {
        ## Make changes into that server which starts with
        #listen 443 ssl;

        ssl_certificate /etc/letsencrypt/live/`SUB-DOMAIN`.quanttiphy.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/`SUB-DOMAIN`.quanttiphy.com/privkey.pem;
        
    }
    ```

5. Test & Reload NGINX
    ```bash
    sudo nginx -t
    sudo systemctl reload nginx
    ```



## Common Issues and Troubleshooting

### Mixed Content Errors
    
- Issue: Frontend loaded over HTTPS calls Backend API over HTTP (e.g., hard-coded http://164.52.195.138:PORT or http://127.0.0.1:PORT).

- Solution: Change ENPOINT in your `.env.production` or `.env` if you have only .env. API_URL=`https://`SUB-DOMAIN`.quanttiphy.com/api`

### 403 Forbidden Erros

- Solution: Remove the default file configuration from `/etc/nginx/sites_enabled` Run below cmd:
    ```bash
    sudo rm /etc/nginx/sites_enabled/default
    ```

### Duplicate Server Name Warnings

- Solution: Keep your custom configuration in `/etc/nginx/ sites_available/` and disable the default configuration in `/etc/nginx/sites_enabled/`.
