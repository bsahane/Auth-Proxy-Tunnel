# Auth-Proxy-Tunnel
Deploy a Secure Application Access Stack with Cloudflare Tunnel, Nginx Proxy Manager, and Authentik using Docker Compose. Ideal for Self-Hosting with Authentication.

# Self-Hosted Secure Access Stack: Cloudflare Tunnel, Nginx Proxy Manager & Authentik

This project provides a Docker Compose setup for securely exposing self-hosted services to the internet using Cloudflare Tunnel, Nginx Proxy Manager, and Authentik for identity and access management.

This setup uses a `.env` file for easy configuration of secrets and paths, and stores persistent data in subdirectories within the project folder (e.g., `./nginxproxymanager/data`, `./goauthentik/postgresql/data`).

## 🌟 Key Features

* **Secure Exposure**: Leverages Cloudflare Tunnels to securely connect your services to the Cloudflare network without opening firewall ports.
* **Wildcard Subdomains**: Configure once to route all subdomains (e.g., `*.website.com`) to Nginx Proxy Manager.
* **Centralized Proxy Management**: Nginx Proxy Manager (NPM) provides an easy-to-use web UI for managing SSL certificates (including Let's Encrypt) and routing traffic to your internal services.
* **Authentication Ready**: Includes Authentik (server, worker, PostgreSQL, Redis) for robust application authentication and authorization.
* **Dockerized**: All services are containerized for easy deployment and management via Docker Compose.
* **Simplified Configuration**: Uses a `.env` file for sensitive data and local path configurations.
* **Local Data Persistence**: Service data is stored in subdirectories within the project folder by default (configured via `DATA_APPDATA_PATH=.` in `.env`).

## 🛠️ Services Included

* **Cloudflared**: Creates the secure tunnel to Cloudflare.
* **Nginx Proxy Manager**: Manages proxy hosts and SSL.
* **Authentik**:
    * `postgresql`: Database for Authentik.
    * `redis`: Cache for Authentik.
    * `server`: Authentik main application.
    * `worker`: Authentik background worker.

## 📋 Prerequisites

* **Docker and Docker Compose**: Installed on your server/machine.
* **Cloudflare Account**: With your domain (e.g., `website.com`) added and managed by Cloudflare.
* **Git**: For cloning the repository.

## 🚀 Setup Instructions

1.  **Clone the Repository:**
    ```bash
    git clone <your-repository-url>
    cd <repository-name>
    ```

2.  **Configure Environment Variables:**
    * Copy the example environment file:
        ```bash
        cp .env.example .env
        ```
    * Edit the `.env` file with your specific details:
        * `CF_TUNNEL_TOKEN`: Your Cloudflare Tunnel token (see Cloudflare Setup Guide below).
        * `AUTHENTIK_SECRET_KEY_VALUE`: A strong, unique secret for Authentik.
        * `POSTGRES_PASSWORD_VALUE`: A strong, unique password for the Authentik PostgreSQL database.
        * `DATA_APPDATA_PATH`: Set to `.` by default, meaning data folders (`nginxproxymanager`, `goauthentik`) will be created in the project's root directory. You can change this to an absolute path if preferred (e.g., `/opt/appdata`). Ensure the path is writable by the Docker daemon/user.

3.  **Cloudflare Configuration:**
    * Follow the detailed instructions in `CLOUDFLARE_SETUP.md` to configure your Cloudflare Tunnel and DNS records to point to Nginx Proxy Manager.

4.  **Start the Services:**
    * Once your `.env` file is configured and Cloudflare is set up to point to where NPM will be, run:
        ```bash
        docker-compose up -d
        ```
    * This will pull the necessary images and start all services in detached mode.
    * Data for Nginx Proxy Manager, Authentik's PostgreSQL, Redis, etc., will be stored in subdirectories like `./nginxproxymanager/data`, `./goauthentik/postgresql`, etc., relative to your `docker-compose.yml` file (if `DATA_APPDATA_PATH=.`).

5.  **Nginx Proxy Manager Initial Setup:**
    * Access Nginx Proxy Manager (NPM) admin UI. If your Docker host IP is `192.168.50.41`, you can access it locally via `http://192.168.50.41:4081` before you route a domain to it via Cloudflare.
    * Default Admin Credentials:
        * Email: `admin@example.com`
        * Password: `changeme`
    * **Change your credentials immediately after logging in!**

6.  **Configure Proxy Hosts in Nginx Proxy Manager:**
    * Now that Cloudflare Tunnel is directing traffic for `*.website.com` (and `website.com`) to NPM, you can set up your services.
    * **Example: Exposing Authentik:**
        * In NPM, add a new Proxy Host.
        * **Domain Names**: `auth.website.com` (or your preferred subdomain for Authentik).
        * **Scheme**: `http`
        * **Forward Hostname / IP**: `authentik_server` (this is the service name from `docker-compose.yml`).
        * **Forward Port**: `9000` (the internal port for Authentik server).
        * **SSL**: Request a new SSL certificate for `auth.website.com` (or use a wildcard certificate for `*.website.com` if you've set one up in NPM via DNS Challenge). Enable "Force SSL".
    * Repeat for other services you wish to expose, pointing to their respective Docker service names and internal ports.

## 🐳 `docker-compose.yml`

```yaml
name: goauthentik
services:
  cloudflared:
    cpu_shares: 90
    command:
      - tunnel
      - --no-autoupdate
      - run
      - --token
      - ${CF_TUNNEL_TOKEN}
    container_name: cloudflared-ngx
    deploy:
      resources:
        limits:
          memory: 15869M
    environment:
      - PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
      - SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
    hostname: cloudflared-ngx
    image: cloudflare/cloudflared:latest
    labels:
      icon: [https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp](https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp)
    restart: unless-stopped
    ports: []
    volumes: []
    devices: []
    cap_add: []
    network_mode: bridge
    privileged: false
  nginxproxymanager:
    cpu_shares: 90
    command: []
    container_name: nginxproxymanager
    deploy:
      resources:
        limits:
          memory: 15869M
        reservations:
          memory: "134217728"
    hostname: nginxproxymanager
    image: jc21/nginx-proxy-manager:2.12.2
    labels:
      icon: [https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp](https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp)
    ports:
      - target: 80
        published: "4080" # HTTP access to NPM (mainly for Cloudflare tunnel target)
        protocol: tcp
      - target: 443
        published: "4443" # HTTPS access to NPM (if handling SSL directly, not typical with CF)
        protocol: tcp
      - target: 81
        published: "4081" # Admin UI for NPM
        protocol: tcp
    restart: unless-stopped
    volumes:
      - type: bind
        source: ${DATA_APPDATA_PATH}/nginxproxymanager/data
        target: /data
      - type: bind
        source: <span class="math-inline">\{DATA\_APPDATA\_PATH\}/nginxproxymanager/etc/letsencrypt
target\: /etc/letsencrypt
devices\: \[\]
cap\_add\: \[\]
environment\: \[\]
network\_mode\: bridge
privileged\: false
postgresql\:
cpu\_shares\: 90
command\: \[\]
deploy\:
resources\:
limits\:
memory\: 15869M
environment\:
\- AUTHENTIK\_ERROR\_REPORTING\_\_ENABLED\=true
\- AUTHENTIK\_SECRET\_KEY\=</span>{AUTHENTIK_SECRET_KEY_VALUE}
      - PG_PASS=<span class="math-inline">\{POSTGRES\_PASSWORD\_VALUE\} \# Note\: Used by postgres entrypoint, ensure it matches POSTGRES\_PASSWORD
\- POSTGRES\_DB\=authentik
\- POSTGRES\_PASSWORD\=</span>{POSTGRES_PASSWORD_VALUE}
      - POSTGRES_USER=authentik
    healthcheck:
      test:
        - CMD-SHELL
        - "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"
      timeout: 5s
      interval: 30s
      retries: 5
      start_period: 20s
    image: docker.io/library/postgres:16-alpine
    labels:
      icon: [https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp](https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp)
    restart: unless-stopped
    volumes:
      - type: bind
        source: ${DATA_APPDATA_PATH}/goauthentik/postgresql
        target: /var/lib/postgresql/data
    ports: []
    devices: []
    cap_add: []
    networks:
      - default
    privileged: false
    container_name: authentik_postgres

  redis:
    cpu_shares: 90
    command:
      - --save
      - "60"
      - "1"
      - --loglevel
      - warning
    deploy:
      resources:
        limits:
          memory: 15869M
    healthcheck:
      test:
        - CMD-SHELL
        - redis-cli ping | grep PONG
      timeout: 3s
      interval: 30s
      retries: 5
      start_period: 20s
    image: docker.io/library/redis:alpine
    labels:
      icon: [https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp](https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp)
    restart: unless-stopped
    volumes:
      - type: bind
        source: <span class="math-inline">\{DATA\_APPDATA\_PATH\}/goauthentik/redisdata
target\: /data
ports\: \[\]
devices\: \[\]
cap\_add\: \[\]
environment\: \[\]
networks\:
\- default
privileged\: false
container\_name\: authentik\_redis
server\:
cpu\_shares\: 90
command\:
\- server
depends\_on\:
postgresql\:
condition\: service\_healthy
required\: true
redis\:
condition\: service\_healthy
required\: true
deploy\:
resources\:
limits\:
memory\: 15869M
environment\:
\- AUTHENTIK\_POSTGRESQL\_\_HOST\=postgresql
\- AUTHENTIK\_POSTGRESQL\_\_NAME\=authentik
\- AUTHENTIK\_POSTGRESQL\_\_PASSWORD\=</span>{POSTGRES_PASSWORD_VALUE}
      - AUTHENTIK_POSTGRESQL__USER=authentik
      - AUTHENTIK_REDIS__HOST=redis
      - AUTHENTIK_SECRET_KEY=<span class="math-inline">\{AUTHENTIK\_SECRET\_KEY\_VALUE\}
\- PG\_PASS\=</span>{POSTGRES_PASSWORD_VALUE} # For Authentik internal use if it needs direct PG access with this var
    image: ghcr.io/goauthentik/server:2025.4.1
    labels:
      icon: [https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp](https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp)
    ports:
      - mode: ingress
        target: 9000 # Internal Authentik HTTP port
        published: "7100" # Host port for Authentik HTTP (can be removed if only accessed via NPM)
        protocol: tcp
      - mode: ingress
        target: 9443 # Internal Authentik HTTPS port
        published: "7143" # Host port for Authentik HTTPS (can be removed if only accessed via NPM)
        protocol: tcp
    restart: unless-stopped
    volumes:
      - type: bind # Ensure path is correct in .env or here
        source: ${DATA_APPDATA_PATH}/goauthentik/Media # Or a more generic path like /media if defined elsewhere
        target: /media
      - type: bind
        source: <span class="math-inline">\{DATA\_APPDATA\_PATH\}/goauthentik/custom\-templates
target\: /templates
devices\: \[\]
cap\_add\: \[\]
networks\:
\- default
privileged\: false
container\_name\: authentik\_server
worker\:
cpu\_shares\: 90
command\:
\- worker
depends\_on\:
postgresql\:
condition\: service\_healthy
required\: true
redis\:
condition\: service\_healthy
required\: true
deploy\:
resources\:
limits\:
memory\: 15869M
environment\:
\- AUTHENTIK\_ERROR\_REPORTING\_\_ENABLED\=true
\- AUTHENTIK\_POSTGRESQL\_\_HOST\=postgresql
\- AUTHENTIK\_POSTGRESQL\_\_NAME\=authentik
\- AUTHENTIK\_POSTGRESQL\_\_PASSWORD\=</span>{POSTGRES_PASSWORD_VALUE}
      - AUTHENTIK_POSTGRESQL__USER=authentik
      - AUTHENTIK_REDIS__HOST=redis
      - AUTHENTIK_SECRET_KEY=<span class="math-inline">\{AUTHENTIK\_SECRET\_KEY\_VALUE\}
\- PG\_PASS\=</span>{POSTGRES_PASSWORD_VALUE} # For Authentik internal use
    image: ghcr.io/goauthentik/server:2025.4.1
    labels:
      icon: [https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp](https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp)
    restart: unless-stopped
    user: root # Review if 'root' is strictly necessary for worker operations.
    volumes:
      - type: bind
        source: /var/run/docker.sock # Required if Authentik needs to interact with Docker (e.g., for outpost deployment)
        target: /var/run/docker.sock
      - type: bind # Ensure path is correct in .env or here
        source: /DATA/Media # Original path, if using ${DATA_APPDATA_PATH} it would be ${DATA_APPDATA_PATH}/goauthentik/Media
        target: /media
      - type: bind
        source: ${DATA_APPDATA_PATH}/goauthentik/certs
        target: /certs
      - type: bind
        source: ${DATA_APPDATA_PATH}/goauthentik/custom-templates
        target: /templates
    ports: []
    devices: []
    cap_add: []
    networks:
      - default
    privileged: false # Be cautious with docker.sock; true might be needed if user 'root' doesn't have group access to docker.sock on host.
    container_name: authentik_worker
networks:
  default:
    name: goauthentik_default
x-casaos:
  author: self
  category: self
  hostname: ""
  icon: [https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp](https://vedproductions.in/wp-content/uploads/2025/05/file_000000004538622f8588011ad7aaa42f.webp)
  index: /
  is_uncontrolled: false
  port_map: "4081"
  scheme: http
  store_app_id: goauthentik
  title:
    custom: Auth Proxy Tunnel
    en_us: cloudflared