# Self-Hosted Vaultwarden Password Manager

## Project Overview

For this project, I am building my own self-hosted password manager using Vaultwarden, Docker, NGINX, and eventually WireGuard.

The goal is to learn more about Linux administration, Docker containers, networking, reverse proxies, HTTPS, VPNs, and general server security while also creating something I can actually use.

Instead of exposing Vaultwarden directly to the internet, I plan to use a VPN for remote access. This will eventually let me securely reach my password vault from my phone or laptop while away from home.

## Hardware

- Dell OptiPlex 7070
- Ubuntu Linux
- Ethernet connection
- Home network

## Technologies

- Ubuntu Linux
- Docker
- Vaultwarden
- NGINX
- WireGuard (planned)
- HTTPS/TLS (planned)
- Git/GitHub

---

## Part 1 - Preparing the Ubuntu Server

I started on my Dell OptiPlex 7070 running Ubuntu. Before installing anything, I updated the package list:

```bash
sudo apt update
```

Then I upgraded the installed packages:

```bash
sudo apt upgrade -y
```

After the updates finished, I rebooted. This made sure I was starting from an updated system before installing Docker and the other services.

## Part 2 - Installing Docker

Vaultwarden runs inside a Docker container, so the next step was installing Docker. I first checked whether it was already installed:

```bash
docker --version
```

Docker was not installed, so I installed it:

```bash
sudo apt install docker.io -y
```

Then I checked the service:

```bash
sudo systemctl status docker
```

Docker showed `Active: active (running)`. I also tested it:

```bash
sudo docker run hello-world
```

Docker downloaded and ran the test container, and the `Hello from Docker!` message confirmed it was working.

![Docker running](screenshots/01-docker-running.png)
![Docker hello-world](screenshots/02-docker-hello-world.png)

### Mistake / Troubleshooting

While checking Docker, I typed `systemctl` incorrectly a couple of times. This reminded me that Linux commands have to be entered exactly. Reading the error message helped me spot the typo and fix it.

## Part 3 - Creating Persistent Vaultwarden Storage

Next, I created a directory where Vaultwarden can permanently store its data:

```bash
sudo mkdir -p /opt/vaultwarden/data
```

I verified it existed:

```bash
sudo ls -la /opt/vaultwarden
```

I initially typed `mkdir` as `mddir`, which caused a "command not found" error. After correcting it, the directory was created.

This directory matters because Docker containers can be removed or recreated. By storing Vaultwarden's data outside the container, the vault data persists even if the container changes.

## Part 4 - Deploying Vaultwarden

I deployed Vaultwarden with Docker:

```bash
sudo docker run -d --name vaultwarden --restart unless-stopped -v /opt/vaultwarden/data:/data -p 127.0.0.1:8080:80 vaultwarden/server:latest
```

This command:

- Creates a container named `vaultwarden`
- Runs it in the background
- Restarts it automatically unless I manually stop it
- Connects `/opt/vaultwarden/data` to the container's `/data`
- Maps localhost port 8080 to Vaultwarden's port 80

I intentionally used `127.0.0.1:8080` instead of exposing Vaultwarden directly to the network. I checked the container with:

```bash
sudo docker ps
```

Vaultwarden showed as `healthy`. I then opened `http://127.0.0.1:8080` in Firefox and the login page loaded.

![Vaultwarden container](screenshots/03-vaultwarden-container.png)
![Vaultwarden on localhost](screenshots/04-vaultwarden-localhost.png)

### Docker Troubleshooting

My first attempts failed with:

```text
docker: invalid reference format
```

I was entering the command across multiple lines and had problems with the formatting and backslashes. Putting it on one line fixed it. This showed me how sensitive command-line syntax can be, especially with longer Docker commands.

## Part 5 - Installing NGINX

I wanted NGINX in front of Vaultwarden as a reverse proxy:

```bash
sudo apt install nginx -y
```

I initially entered `nginx-y` instead of `nginx -y`. Ubuntu treated `nginx-y` as the package name and said it could not locate it. After fixing the spacing, NGINX installed. I checked it with:

```bash
sudo systemctl status nginx
```

The service showed `Active: active (running)`.

## Part 6 - Configuring the NGINX Reverse Proxy

I created a new configuration file:

```bash
sudo nano /etc/nginx/sites-available/vaultwarden
```

It forwards requests from NGINX to the Vaultwarden container on localhost:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

I enabled it:

```bash
sudo ln -s /etc/nginx/sites-available/vaultwarden /etc/nginx/sites-enabled/vaultwarden
```

I also removed the default NGINX site:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Before reloading, I tested the configuration:

```bash
sudo nginx -t
```

## Part 7 - Troubleshooting NGINX

This was one of the most useful troubleshooting parts of the project so far.

### Error 1 - Incorrect Upstream Address

The first test failed with:

```text
host not found in upstream "127.0.0.1."
```

I had accidentally entered an extra period.

```nginx
# Incorrect
proxy_pass http://127.0.0.1.:8080;

# Correct
proxy_pass http://127.0.0.1:8080;
```

![NGINX upstream error](screenshots/05-nginx-error-upstream.png)

### Error 2 - Incorrect NGINX Variable

The next test returned:

```text
unknown "hot" variable
```

I had entered `$hot` instead of `$host`. After fixing it, I ran `sudo nginx -t` again, and this time NGINX returned:

```text
syntax is ok
test is successful
```

![NGINX variable error](screenshots/06-nginx-error-variable.png)
![NGINX test success](screenshots/07-nginx-test-success.png)

This taught me why testing a configuration before reloading a service is important. If I make a mistake, `nginx -t` lets me find it before applying the configuration.

## Part 8 - Activating the Reverse Proxy

After the successful test, I reloaded NGINX:

```bash
sudo systemctl reload nginx
sudo systemctl status nginx
```

NGINX was still `active (running)` and the logs showed a successful reload. I then opened `http://127.0.0.1` instead of port 8080, and the Vaultwarden login page appeared, confirming the reverse proxy works.

![NGINX running](screenshots/08-nginx-running.png)
![Vaultwarden through NGINX](screenshots/09-vaultwarden-nginx.png)

The connection currently looks like this:

```text
Browser
   |
   v
NGINX :80
   |
   v
127.0.0.1:8080
   |
   v
Docker
   |
   v
Vaultwarden
```

---

## Current Project Status

**Completed:**

- [x] Ubuntu system updated
- [x] Docker installed and running
- [x] Vaultwarden container deployed and healthy
- [x] Persistent Vaultwarden storage configured
- [x] Vaultwarden accessible locally
- [x] NGINX installed and configured as a reverse proxy
- [x] NGINX configuration tested
- [x] Troubleshooting documented

**Still to do:**

- [ ] Allow controlled access from other devices on my home network
- [ ] Configure firewall rules
- [ ] Configure HTTPS/TLS
- [ ] Secure the Vaultwarden configuration
- [ ] Configure WireGuard VPN
- [ ] Test remote access from my phone
- [ ] Connect the Bitwarden mobile app
- [ ] Configure backups
- [ ] Perform final security testing

## What I Have Learned So Far

Setting up a server is not just installing an application. Even though Vaultwarden runs inside Docker, I have already had to work with Linux commands, services, directories, ports, containers, persistent storage, NGINX configuration files, reverse proxies, and troubleshooting.

I also ran into multiple errors along the way. Instead of starting over when something failed, I read the error messages and used them to figure out what part of the configuration was wrong.

The project is not finished yet. My next goal is to make Vaultwarden accessible from other devices on my home network before moving on to HTTPS and remote VPN access.
