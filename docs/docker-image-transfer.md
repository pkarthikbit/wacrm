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

## Load and Run

On the destination computer:

```bash
docker load < wacrm-0.8.0.tar.gz

docker run -d \
  --name wacrm \
  --restart unless-stopped \
  --env-file .env.local \
  -e PORT=3000 \
  -p 3000:3000 \
  wacrm:0.8.0
```

Open `http://localhost:3000` on the destination computer. To publish a
different host port, change the left side of the mapping, for example
`-p 8080:3000`.

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
