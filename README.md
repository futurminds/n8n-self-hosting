# Self-Hosting SSL enabled N8N on a Linux Server

This guide provides step-by-step instructions to self-host [n8n](https://n8n.io), a free and open-source workflow automation tool, on a Linux server using Docker, Nginx, and Certbot for SSL with a custom domain name.

Youtube Video Explanation: https://www.youtube.com/watch?v=Temh_Ddxp24
![Screenshot 2024-08-03 125607](https://github.com/user-attachments/assets/908eaaf2-d66f-446e-8dd8-a3676da2bc89)


## Step 1: Installing Docker

1. **Update the Package Index:**
   ```bash
   sudo apt update

2. **Install Docker:**
    ```bash
    sudo apt install docker.io

3.  **Start Docker:**
    ```bash
    sudo systemctl start docker

4. **Enable Docker to Start at Boot:**
    ```bash
    sudo systemctl enable docker


## Step 2: Starting n8n in Docker

Run the following command to start n8n in Docker. Replace your-domain.com with your actual domain name:

    ```bash
    sudo docker run -d --restart unless-stopped -it \
    --name n8n \
    -p 5678:5678 \
    -e N8N_HOST="your-domain.com(or)externalip:port(5678)" \
    -e WEBHOOK_TUNNEL_URL="https://your-domain.com(or)https://externalip:port(5678)/" \
    -e WEBHOOK_URL="https://your-domain.com(or)https://externalip:port(5678)/" \
    -v ~/.n8n:/root/.n8n \
    n8nio/n8n
    ```

Or if you are using a subdomain, it should look like this:

    ```bash
    sudo docker run -d --restart unless-stopped -it \
    --name n8n \
    -p 5678:5678 \
    -e N8N_HOST="subdomain.your-domain.com" \
    -e WEBHOOK_TUNNEL_URL="https://subdomain.your-domain.com/" \
    -e WEBHOOK_URL="https://subdomain.your-domain.com/" \
    -v ~/.n8n:/root/.n8n \
    n8nio/n8n
    ```


This command does the following:

- Downloads and runs the n8n Docker image.
- Exposes n8n on port 5678.
- Sets environment variables for the n8n host and webhook tunnel URL.
- Mounts the n8n data directory for persistent storage.
- After executing the command, n8n will be accessible on your-domain.com:5678.

## Step 3: Installing Nginx

Nginx is used as a reverse proxy to forward requests to n8n and handle SSL termination.

1. **Install Nginx:**
    ```bash
    sudo apt install nginx

## Step 4: Configuring Nginx

Configure Nginx to reverse proxy the n8n web interface:

1. **Create a New Nginx Configuration File:**
    ```bash
    sudo nano /etc/nginx/sites-available/n8n

2. **Paste the Following Configuration:**
    ```bash
    server {
        listen 80;
        server_name your-domain.com; // subdomain.your-domain.com if you have a subdomain

        location / {
            proxy_pass http://localhost:5678;
            proxy_http_version 1.1;
            chunked_transfer_encoding off;
            proxy_buffering off;
            proxy_cache off;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
    ```
    Replace your-domain.com with your actual domain.

3. **Enable the Configuration:**
    ```bash
    sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/

    If you see the error saying /etc/nginx/sites-enabled/ doesn't exist. Create it by running: sudo mkdir /etc/nginx/sites-enabled/

4. **Test the Nginx Configuration and Restart:**
    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```

## Step 5: Setting up SSL with Certbot

Certbot will obtain and install an SSL certificate from Let's Encrypt.

1. **Install Certbot and the Nginx Plugin:**
    ```bash
    sudo apt install certbot python3-certbot-nginx

2. **Obtain an SSL Certificate:**
    ```bash
    sudo certbot --nginx -d your-domain.com
    // If you have a subdomain then it will be subdomain.your-domain.com

Follow the on-screen instructions to complete the SSL setup.
Once completed, n8n will be accessible securely over HTTPS at your-domain.com.

IMPORTANT: Make sure you follow the above steps in order. Step 5 will modify your /etc/nginx/sites-available/n8n.conf file to something like this:
![image](https://github.com/user-attachments/assets/344187ec-5bcf-4d97-ad35-21b6562182e5)
 

## How to update n8n:

You can follow the instructions here to update the version: https://docs.n8n.io/hosting/installation/docker/#updating

Important: Take a backup of ~/.n8n:/home/node/.n8n
To create a backup, you can copy ~/.n8n:/home/node/.n8n to your local or another directory on the same VM even before deleting the container. And then, after updating and spinning up a new container, if you see the data getting lost, you can replace ~/.n8n:/home/node/.n8n with the one you saved earlier.

Ensure that your n8n instance is using a persistent volume or a mapped directory for its data storage. This is crucial because the workflows, user accounts, and configurations are stored in the database file (typically database.sqlite), which should be located in a directory that remains intact even when the container is removed.
In your docker-compose.yml, you should have something like this:
```bash
volumes:
- ~/.n8n:/home/node/.n8n
```

This mapping ensures that the .n8n directory on your host machine is used for data storage, preserving your workflows and configurations across container updates.

When you stop and remove the n8n container, you are only deleting the container instance itself, not the data stored in the persistent volume. As long as the volume is correctly configured, your workflows and accounts should remain unaffected.

But to avoid any chance of data loss you should take a backup of ~/.n8n:/home/node/.n8n before removing the container.

## Important Notes
- Ensure your domain's DNS A record points to your server's IP address.
- Allow ports 80 (HTTP), 443 (HTTPS), and 5678 (n8n) in your server's firewall.
- Nginx handles SSL termination, so it forwards requests to the n8n instance over HTTP internally.

## Why Nginx and Certbot?

**Nginx:** It serves as a reverse proxy, forwarding client requests to n8n running on Docker. This setup enhances security, load balancing, and scalability.

**Certbot:** Certbot is a tool from the Electronic Frontier Foundation (EFF) that automates the process of obtaining and renewing SSL certificates from Let's Encrypt, a free and open Certificate Authority.

By using Nginx and Certbot, you ensure that your n8n instance is securely accessible over the internet with HTTPS.

## Troubleshooting

If you encounter issues with Nginx, check the logs located at /var/log/nginx/error.log for more details.
For Docker-related issues, ensure the Docker service is running: sudo systemctl status docker.
