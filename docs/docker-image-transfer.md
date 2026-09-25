# Transfer the Docker Image to Another PC

This guide exports the built WACRM image as a file, copies it to another
computer, and runs it there without rebuilding the application.

## Build and Export

Run these commands on the computer where the image is built:

```bash
docker compose --env-file .env.local build
docker save wacrm:0.8.0 | gzip > wacrm-0.8.0.tar.gz
```

If Docker requires `sudo` on the build computer:

```bash
sudo docker compose --env-file .env.local build
sudo docker save wacrm:0.8.0 | gzip > wacrm-0.8.0.tar.gz
```

The Compose service is configured with the stable image name
`wacrm:0.8.0`, so no separate `docker tag` command is needed. If the image
version changes, update the `image:` value in `docker-compose.yml` and the
matching tag in this guide.

The `NEXT_PUBLIC_*` Supabase values are embedded into the client bundle during
this build. Make sure `.env.local` contains the correct values before building.

## Copy the Files

Copy the image archive and the runtime environment file to the destination
computer. For example:

```bash
scp wacrm-0.8.0.tar.gz .env.local user@OTHER_PC:/path/to/wacrm/
```

Keep `.env.local` private. It contains server-side secrets required at runtime.

## Extract and Load the Image

Run these commands on the destination computer:

1. Extract the archive if it was transferred as a gzip-compressed tar file:

   ```bash
   gunzip wacrm-0.8.0.tar.gz
   ```

   This creates `wacrm-0.8.0.tar`.

2. Load the image into Docker:

   ```bash
   sudo docker load --input wacrm-0.8.0.tar
   ```

3. Remove an older container with the same name, if one exists:

   ```bash
   sudo docker rm -f wacrm 2>/dev/null || true
   ```

## Run the Container for Nginx

Run WACRM on localhost only. Docker does not publish port 3000 publicly;
Nginx will receive public HTTP requests and forward them to this container.

```bash
sudo docker run -d \
   --name wacrm \
   --restart unless-stopped \
   --env-file .env.local \
   -e PORT=3000 \
   -p 127.0.0.1:3000:3000 \
   wacrm:0.8.0
```

Check that the container is running:

```bash
sudo docker ps
sudo docker logs --tail 100 wacrm
```

## Configure Nginx

Install Nginx on the destination VM:

```bash
sudo apt update
sudo apt install -y nginx
```

Create a site configuration:

```bash
sudo nano /etc/nginx/sites-available/wacrm
```

Paste this configuration. The domain must already resolve to the VM's public
IP address before continuing:

```nginx
server {
   listen 80;
   listen [::]:80;
   server_name crm.inveh.in;

   client_max_body_size 25M;

   location / {
      proxy_pass http://127.0.0.1:3000;
      proxy_http_version 1.1;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
   }
}
```

Enable the site, disable the default site, test the configuration, and reload
Nginx:

```bash
sudo ln -sfn /etc/nginx/sites-available/wacrm /etc/nginx/sites-enabled/wacrm
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl enable --now nginx
sudo systemctl reload nginx
```

Allow public HTTP traffic before testing the domain:

```bash
sudo ufw allow 80/tcp
sudo ufw status
```

Also add an inbound TCP port 80 rule in the cloud provider's network
security rules. For a temporary test, allow `0.0.0.0/0`; restrict the source
range for production use.

Verify each layer on the VM:

```bash
sudo docker ps
curl -i http://127.0.0.1:3000
sudo nginx -t
sudo systemctl status nginx --no-pager
curl -i -H 'Host: crm.inveh.in' http://127.0.0.1
```

The first `curl` must return the WACRM page. The second must return the same
page through Nginx. If the first fails, restart the container and inspect its
logs. If the first succeeds but the second fails, fix Nginx. If both local
checks succeed but `http://crm.inveh.in` fails from your computer, fix the
cloud firewall or UFW rules.

## Enable HTTPS

Do not test `https://crm.inveh.in` yet. The HTTP-only Nginx configuration
listens on port 80; port 443 starts listening only after Certbot installs the
certificate and HTTPS configuration.

Install Certbot and its Nginx plugin:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Request and install a certificate for the domain:

```bash
sudo certbot --nginx -d crm.inveh.in
```

When prompted, choose the option to redirect HTTP traffic to HTTPS. Certbot
will update the Nginx configuration and restart TLS automatically. Verify
renewal without changing the live certificate:

```bash
sudo certbot renew --dry-run
```

Allow public HTTP and HTTPS traffic on the VM:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status
```

Add an inbound TCP port 443 rule in the cloud provider's network security
rules. Port 80 should already be allowed from the HTTP setup above.

Confirm that Nginx is listening on HTTPS before testing from a browser:

```bash
sudo nginx -t
sudo systemctl reload nginx
sudo ss -ltnp | grep ':443'
curl -I https://crm.inveh.in
```

If `ss` shows no listener on `:443`, Certbot did not install the HTTPS
configuration. Run `sudo certbot --nginx -d crm.inveh.in` again and inspect
`sudo journalctl -u nginx -n 100 --no-pager`.

Open the app at:

```text
https://crm.inveh.in
```

Useful troubleshooting commands:

```bash
sudo systemctl status nginx
sudo journalctl -u nginx -n 100 --no-pager
sudo docker logs --tail 100 wacrm
curl http://127.0.0.1:3000
curl -H 'Host: crm.inveh.in' http://127.0.0.1
curl -I https://crm.inveh.in
```

## CPU Architecture

The image must match the destination computer's CPU architecture. To build
for a specific target, use Docker Buildx and replace `linux/amd64` with the
required platform when necessary:

```bash
docker buildx build \
  --platform linux/amd64 \
  --build-arg NEXT_PUBLIC_SUPABASE_URL="$NEXT_PUBLIC_SUPABASE_URL" \
  --build-arg NEXT_PUBLIC_SUPABASE_ANON_KEY="$NEXT_PUBLIC_SUPABASE_ANON_KEY" \
  -t wacrm:0.8.0 \
  --load .

docker save wacrm:0.8.0 | gzip > wacrm-0.8.0.tar.gz
```

The destination computer still needs Docker installed, but it does not need
Node.js, npm, or the source repository.
