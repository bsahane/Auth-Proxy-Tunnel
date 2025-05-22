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
    git clone https://github.com/bsahane/Auth-Proxy-Tunnel.git
    cd Auth-Proxy-Tunnel
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
