# Cloudflare Tunnel Configuration Guide

This guide details how to set up Cloudflare Tunnel and DNS to securely expose your Nginx Proxy Manager (NPM) instance, which then routes traffic to your self-hosted services like Authentik.

**Assumptions:**
* You have a Cloudflare account.
* Your domain (`website.com`) is added to your Cloudflare account and its DNS is managed by Cloudflare.
* Your Docker host (where `docker-compose up` will be run) has an IP like `192.168.50.41` (this IP is primarily for local access to NPM admin, not directly for the tunnel's public hostname config).
* Nginx Proxy Manager is configured to listen on port `80` internally within Docker (mapped to host port `4080` in `docker-compose.yml`). The NPM admin panel is on port `81` (mapped to host port `4081`).

## Step 1: Create a Cloudflare Tunnel

1.  **Navigate to Cloudflare Zero Trust Dashboard:**
    * Log in to your Cloudflare account.
    * From the main dashboard, select "Zero Trust" (usually on the left sidebar or in the top navigation).

2.  **Access Tunnels:**
    * In the Zero Trust dashboard, go to `Access` -> `Tunnels`.

3.  **Create a New Tunnel:**
    * Click `+ Create a tunnel`.
    * Choose `Cloudflared` as the connector type. Click `Next`.
    * **Name your tunnel:** Give it a descriptive name (e.g., `npm-gateway` or `homelab-services`). Click `Save tunnel`.

4.  **Install and Run a Connector:**
    * Cloudflare will now show you commands to install `cloudflared` and run it. **You primarily need the tunnel token from the `cloudflared tunnel run <TOKEN>` command.**
    * Example command provided by Cloudflare: `cloudflared tunnel run your-long-alphanumeric-token`
    * **Copy this token.** This is what you will put in your `.env` file for the `CF_TUNNEL_TOKEN` variable.
    * You don't need to run this command manually on your host if you are using the `cloudflared` service in the `docker-compose.yml`, as it will use this token.
    * Once you've noted the token, you can click `Next`. Your connector will show up as "inactive" for now, which is fine. It will become active once your `docker-compose` stack is running with the correct token.

## Step 2: Configure Public Hostname for Nginx Proxy Manager

Now, you need to tell the tunnel how to route public traffic to your Nginx Proxy Manager.

1.  **Select your Tunnel:**
    * In the `Access` -> `Tunnels` list, find the tunnel you just created (e.g., `npm-gateway`) and click `Configure`.

2.  **Add a Public Hostname:**
    * Go to the `Public Hostnames` tab.
    * Click `+ Add a public hostname`.

3.  **Configure the Route to Nginx Proxy Manager:**
    * **Subdomain:**
        * To make `website.com` itself point to NPM, leave this blank if your domain is `website.com`.
        * Alternatively, you can use a specific subdomain like `proxy` if you prefer (e.g., `proxy.website.com`), and then CNAME `*` and `@` (root) to `proxy.website.com` in DNS later. For simplicity, let's assume you want `website.com` (and `*.website.com` via DNS) to hit NPM.
    * **Domain:** Select `website.com` from the dropdown.
    * **Path:** Leave blank (unless you want to route a specific path only).
    * **Service Type:** Select `HTTP`.
    * **URL:** `nginxproxymanager:80`
        * **Explanation:** This tells Cloudflare Tunnel to forward requests to the `nginxproxymanager` service (as named in your `docker-compose.yml`) on port `80` *within the Docker internal network*. Cloudflare handles external HTTPS.
    * **Additional settings (Optional but Recommended):**
        * Expand `HTTP Settings` and enable `HTTP2 connection`.
        * Expand `TLS` and enable `No TLS Verify` if NPM is serving HTTP internally (which it is in this setup, as Cloudflare handles external SSL). If NPM were configured for HTTPS internally with a self-signed cert, you might need this. For `http://nginxproxymanager:80`, it's not strictly needed but doesn't hurt.

4.  **Save the Public Hostname:**
    * Click `Save hostname`.

## Step 3: Configure Cloudflare DNS Records

Cloudflare might have automatically created a CNAME record for your tunnel when you set up the public hostname. If not, or if you want a wildcard, you'll need to configure them manually.

1.  **Navigate to DNS Settings for `website.com`:**
    * Go back to your main Cloudflare Dashboard (not Zero Trust).
    * Select your domain `website.com`.
    * Click on the `DNS` section.

2.  **Add/Verify CNAME Record for the Tunnel:**
    * You should see a CNAME record that points to your tunnel. It usually looks like:
        * **Type:** `CNAME`
        * **Name:** `website.com` (or the subdomain you used in the Public Hostname setup)
        * **Target:** `<your-tunnel-id>.cfargotunnel.com` (e.g., `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.cfargotunnel.com`)
        * **Proxy status:** Proxied (Orange Cloud)
    * If this record was automatically created when you added the public hostname and it matches the domain/subdomain you set for NPM (e.g. `website.com`), it might be sufficient.

3.  **Add a Wildcard CNAME Record:**
    To make all subdomains (`*.website.com`) also go through the tunnel to Nginx Proxy Manager:
    * Click `+ Add record`.
    * **Type:** `CNAME`
    * **Name:** `*` (this creates the wildcard)
    * **Target:** `website.com` (if `website.com` is already CNAME'd to the tunnel) OR directly to your tunnel ID: `<your-tunnel-id>.cfargotunnel.com`. Pointing to `website.com` is often cleaner if `website.com` itself is already correctly pointing to the tunnel.
    * **Proxy status:** Proxied (Orange Cloud)
    * Click `Save`.

## Step 4: Deploy Docker Stack & Verify

1.  **Ensure your `.env` file has the correct `CF_TUNNEL_TOKEN`.**
2.  **Start your Docker stack:**
    ```bash
    docker-compose up -d
    ```
3.  **Check Tunnel Status:**
    * Go back to the Cloudflare Zero Trust dashboard (`Access` -> `Tunnels`).
    * Your tunnel connector should now show as "Healthy".

## Step 5: Configure Nginx Proxy Manager

* With the Cloudflare setup complete, any request to `https://*.website.com` or `https://website.com` will be routed by Cloudflare Tunnel to your Nginx Proxy Manager container on its internal port `80`.
* Access your Nginx Proxy Manager admin UI (e.g., locally via `http://192.168.50.41:4081` or by setting up a specific subdomain for it in NPM itself, like `npm.website.com`).
* In Nginx Proxy Manager, you will now:
    * Add Proxy Hosts for each service (e.g., `auth.website.com`, `service2.website.com`).
    * Point them to the respective Docker service names and ports (e.g., `auth.website.com` -> `http://authentik_server:9000`).
    * Manage SSL certificates (Let's Encrypt via DNS challenge is recommended for wildcard certs like `*.website.com`).

Your setup should now be live! Traffic flows from the internet -> Cloudflare -> Cloudflare Tunnel -> Nginx Proxy Manager -> Your internal Docker services (like Authentik).