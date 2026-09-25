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

## Extract, Load, and Run

Run these commands on the destination computer. This workflow does not use a
reverse proxy and does not publish a Docker port with `-p`:

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

4. Run the image without a Docker port mapping:

   ```bash
   sudo docker run -d \
     --name wacrm \
     --restart unless-stopped \
     --env-file .env.local \
     -e PORT=3000 \
     wacrm:0.8.0
   ```

This starts the container, but it is not reachable from another computer or
from the VM public IP because no host port is published. Check its status and
logs with:

```bash
sudo docker ps
sudo docker logs --tail 100 wacrm
```

To access the app from a browser without a reverse proxy, Docker must publish
a host port. For the public IP on standard HTTP port 80, use `-p 80:3000` in
the `docker run` command and allow TCP port 80 in the cloud firewall and UFW.
Then open `http://129.159.232.184`.

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
