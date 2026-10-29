# Docker + Portainer DevOps Lab on Rocky Linux

A hands-on record of installing Docker, deploying Portainer CE, running
Nginx, and publishing a static website using Docker Compose and
Portainer Stacks.

## Lab environment

-   OS: Rocky Linux 9.x
-   Host interface/IP observed during the lab: `eno1` / `192.168.1.145`
-   Project directory:
    `/home/shreyash.wadkar/Desktop/Docker/compose-demo`
-   Container management: Docker Engine and Portainer CE
-   Web server: `nginx:alpine`

> **Security:** The IP may change. Never commit passwords, setup tokens,
> private keys, or other secrets. Docker group membership effectively
> grants root-level control. Keep Portainer restricted to trusted
> networks.

## 1. Install Docker Engine

Docker was initially missing, while internet access was available. Add
Docker's CentOS-compatible repository and install the engine and Compose
plugin:

``` bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf repolist | grep -i docker

sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

Verify:

``` bash
docker --version
docker compose version
sudo docker run hello-world
```

The `hello-world` container prints a message and exits. An exited
container with code `0` is expected.

## 2. Configure Docker permissions

Running `docker run hello-world` as the normal user initially returned a
permission error for `/var/run/docker.sock`.

Add the user to the Docker group:

``` bash
sudo usermod -aG docker shreyash.wadkar
```

Inspect membership and socket permissions:

``` bash
getent group docker
id
ls -l /var/run/docker.sock
```

The user was listed in the group database, but the existing login
session did not include `docker` as an active group. Refresh group
membership by logging out of the desktop completely and logging back in,
or temporarily use:

``` bash
newgrp docker
```

Then test:

``` bash
id
docker ps
docker run --rm hello-world
```

**Security note:** Docker group membership is effectively root-level
access. Do not use `chmod 666 /var/run/docker.sock`.

## 3. Deploy Portainer CE

Create a persistent data volume:

``` bash
docker volume create portainer_data
```

Run Portainer:

``` bash
docker run -d \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:lts
```

Verify:

``` bash
docker ps
docker logs --tail 50 portainer
```

Open `https://localhost:9443`, create an administrator account with a
strong password of at least 12 characters, and complete initial setup
promptly. If the setup page times out, restart Portainer and reopen the
page:

``` bash
docker restart portainer
```

If a setup token is requested, obtain a fresh one from the Portainer
logs. Treat tokens as secrets and never commit them.

## 4. Run a basic Nginx container

A first Nginx container was used to test Docker and port publishing:

``` bash
docker run -d \
  --name my-nginx \
  --restart unless-stopped \
  -p 8080:80 \
  nginx:alpine
```

Open `http://localhost:8080` to view the default Nginx welcome page.

`8080:80` maps host port 8080 to port 80 inside the container.

## 5. Create the Compose project

The project was kept in the user's home directory to avoid permissions
issues under `/opt`:

``` bash
mkdir -p ~/Desktop/Docker/compose-demo
cd ~/Desktop/Docker/compose-demo
```

Create `index.html`:

``` bash
cat > index.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>My Docker Compose Lab</title>
</head>
<body>
  <h1>Hello from Docker Compose!</h1>
  <p>This website is running in an Nginx container.</p>
  <p>Deployed using Portainer Stacks on Rocky Linux.</p>
</body>
</html>
EOF
```

## 6. Deploy using Portainer Stacks

In Portainer, select the `local` environment, open **Stacks → Add
stack**, name it `compose-demo`, choose the Web editor, paste the YAML
below, and click **Deploy the stack**.

Save the same content in a file named `compose.yaml` in the project
directory for Git version control:

``` yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8081:80"
    volumes:
      - /home/shreyash.wadkar/Desktop/Docker/compose-demo/index.html:/usr/share/nginx/html/index.html:ro,Z
    restart: unless-stopped
```

### YAML explained

-   `services`: defines the application components.
-   `web`: the logical service name.
-   `image`: uses the lightweight Nginx Alpine image.
-   `ports: "8081:80"`: publishes host port 8081 and forwards traffic to
    container port 80.
-   `volumes`: bind-mounts the host's HTML file into Nginx's document
    root.
-   `:ro`: makes the mounted file read-only inside the container.
-   `:Z`: applies a private SELinux label for the bind mount on
    SELinux-enabled Rocky Linux.
-   `restart: unless-stopped`: restarts the container after failures or
    Docker restarts unless it was explicitly stopped.

**Portability:** The bind-mount path is specific to this machine and
username. Update it if you clone the repository elsewhere. A future
improvement is to mount the project directory or build a custom image
that includes the HTML.

## 7. Test the website

Open `http://localhost:8081`. The page should show **Hello from Docker
Compose!**

Test from the terminal:

``` bash
curl http://localhost:8081
docker ps
```

The Portainer Containers page should show Portainer, the standalone
`my-nginx` container on port 8080, and a Compose-managed web container
on port 8081. The `hello-world` container may also appear in the stopped
container list until removed.

## 8. Update the page without rebuilding

Edit `index.html` in the project directory. For example:

``` bash
cat > index.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>DevOps Lab</title>
</head>
<body>
  <h1>My Rocky Linux DevOps Lab</h1>
  <p>Application: Nginx</p>
  <p>Deployment: Docker Compose</p>
  <p>Management: Portainer</p>
  <p>My first containerized website is working!</p>
</body>
</html>
EOF
```

Refresh `http://localhost:8081`. Since the file is bind-mounted, changes
should appear without rebuilding the image or redeploying the stack. Use
`Ctrl+Shift+R` if the browser shows a cached page.

## 9. Troubleshooting commands

``` bash
docker ps
docker ps -a
docker images
docker volume ls
docker volume inspect portainer_data
docker logs --tail 80 portainer
docker logs --tail 80 my-nginx
sudo ss -lntp | grep -E ':(8080|8081|9443)\b'
sudo firewall-cmd --list-all
```

For the Compose container, find its actual name in `docker ps` or
Portainer before checking logs. If LAN access is required, allow the
relevant port only for a trusted network. Avoid exposing Portainer or
test services to the public internet.

## 10. Push the project to Git

Recommended repository structure:

``` text
compose-demo/
├── README.md
├── compose.yaml
└── index.html
```

From the project directory, ensure `compose.yaml` is saved, then:

``` bash
git init
git add README.md compose.yaml index.html
git commit -m "Add Docker Compose Nginx lab managed by Portainer"
git branch -M main
git remote add origin <YOUR_GIT_REPOSITORY_URL>
git push -u origin main
```

Replace `<YOUR_GIT_REPOSITORY_URL>` with the actual repository URL.
Never commit passwords, Portainer setup tokens, SSH private keys, or
other secrets.

## 11. Skills practiced

-   Installing Docker Engine and Docker Compose on Rocky Linux.
-   Understanding Docker daemon/socket permissions and group membership.
-   Deploying Portainer CE with persistent configuration storage.
-   Running Nginx and publishing container ports.
-   Defining services declaratively with Compose YAML.
-   Deploying and managing a Stack through Portainer.
-   Bind mounts, read-only mounts, and SELinux labels.
-   Updating static content without rebuilding an image.
-   Inspecting containers, images, volumes, logs, and listening ports.
-   Preparing a lab project for version control.

## Next improvements

1.  Make the bind mount more portable.
2.  Add a second service and a dedicated Compose network.
3.  Add health checks and resource limits.
4.  Build a custom image with a Dockerfile.
5.  Create a Git-based CI/CD pipeline.

------------------------------------------------------------------------

**Lab status:** Docker and Portainer were running, and the
Compose-managed Nginx website was successfully displayed in the browser.
