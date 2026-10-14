# Fix: `the --mount option requires BuildKit`

## The error

`docker build` stops like this:

```
DEPRECATED: The legacy builder is deprecated and will be removed in a future release.
            Install the buildx component to build images with BuildKit:
            https://docs.docker.com/go/buildx/
...
the --mount option requires BuildKit. Refer to https://docs.docker.com/go/buildkit/ to learn how to build images with BuildKit enabled
```

## The cause

The **buildx** plugin is not installed on your VM, so Docker uses its old builder.

## The solution

Run these commands on the VM:

```bash
ARCH=$(uname -m); case "$ARCH" in x86_64) BX=amd64; DC=x86_64 ;; aarch64|arm64) BX=arm64; DC=aarch64 ;; esac
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -fsSL "https://github.com/docker/buildx/releases/download/v0.19.3/buildx-v0.19.3.linux-$BX" -o /usr/local/lib/docker/cli-plugins/docker-buildx
sudo curl -fsSL "https://github.com/docker/compose/releases/download/v2.32.4/docker-compose-linux-$DC" -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-buildx /usr/local/lib/docker/cli-plugins/docker-compose
```

## Check

Both must print a version:

```bash
docker buildx version
docker compose version
```

Then run your build again:

```bash
docker build -t <image> .
```
