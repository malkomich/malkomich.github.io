---
date: 2026-02-18 13:31:26
layout: post
title: "Self-Hosting n8n on Raspberry Pi: HTTPS, DDNS & Automated Backups"
subtitle: A production-ready home automation server with Docker, PostgreSQL, SSL
  and cloud backups
description: A full guide on running n8n on Raspberry Pi with Docker, HTTPS,
  DDNS, PostgreSQL, reverse proxy, and automated backups. Production-ready,
  secure, and resilient.
image: /assets/img/uploads/8ed679b2-e4aa-4f6e-b612-648cdcf2e977.png
optimized_image: /assets/img/uploads/8ed679b2-e4aa-4f6e-b612-648cdcf2e977.png
author: malkomich
permalink: /self-hosting-n8n-on-raspberry-pi:-https,-ddns-&-automated-backups/
category: automation
tags:
  - n8n
  - docker
  - ddns
  - raspberrypi
  - automation
paginate: false
---
## 1. Why I Choose to Self-Host n8n on a Raspberry Pi

When I first started working with automation tools, n8n's cloud offering seemed like the obvious choice. But after a few months, I found myself increasingly concerned about data residency, especially when handling customer information and internal business logic. The costs were also scaling faster than I'd anticipated, and I kept running into those limitations that come with any SaaS platform: can't customize the infrastructure, can't control update timing, can't peek under the hood when something goes sideways.

As a backend engineer with a homelab setup already humming along in my office, the idea of self-hosting n8n on a Raspberry Pi started making more sense. I wanted complete control over my automation stack without the recurring charges or vendor dependencies. More importantly, I wanted to understand every layer of the system—from the database to the reverse proxy to the backup strategy. This wasn't just about getting n8n running locally on port 5678 and calling it done. I needed a setup that could handle production workloads, stay accessible 24/7 from anywhere, and recover gracefully from failures.

What I ended up building is what I'm sharing with you here: a fully Dockerized n8n stack with PostgreSQL, secure HTTPS endpoints, dynamic DNS to handle my home ISP's changing IP addresses, and automated cloud backups. It's been running reliably for months now, handling everything from customer onboarding workflows to internal notification systems. The best part? No monthly bills, and I sleep better knowing exactly where my data lives.

## 2. Choosing the Right Hardware for Real-World Workloads

I've experimented with several Raspberry Pi models over the years, and for n8n specifically, I've settled on the Pi 4 with at least 4GB of RAM. The older Pi 3 models struggle once you start running multiple concurrent workflows, especially when PostgreSQL is competing for resources. The memory overhead matters more than you might think—n8n itself isn't heavy, but once you're processing webhooks, running HTTP requests, and transforming data simultaneously, those gigabytes disappear quickly.

Here's what I learned the hard way about storage: microSD cards are fine for testing, but they'll betray you in production. I experienced database corruption twice before switching to a USB 3.0 SSD. The I/O performance difference is dramatic, and you eliminate the SD card wear issue that plagues long-running database workloads. If you're serious about uptime, spend the extra thirty dollars on a small SSD. Your future self will thank you during the next power outage when your database doesn't need recovery.

Temperature management isn't glamorous, but it matters. My Pi runs in a closet where the ambient temperature can climb during summer. A simple aluminum heat sink case keeps things stable under sustained load. I've also added a small UPS battery backup—not an enterprise-grade one, just a consumer model that gives me fifteen minutes of runtime. That's enough to gracefully shut down during brief power blips, which are surprisingly common in my neighborhood.

## 3. Architecting for Modularity with Docker Compose

![Container architecture diagram showing the five main components (n8n, PostgreSQL, pgAdmin, Nginx Proxy Manager, rclone) with arrows indicating data flow, dependencies, and volume mounts. Should visualize how requests flow from the internet through Nginx to n8n, and how n8n connects to PostgreSQL.](https://media.geeksforgeeks.org/wp-content/uploads/20240715174859/Microservices-with-Docker-Containers.webp)



I could have installed n8n directly on the Pi's operating system, and honestly, that would have been simpler initially. But I've been burned before by monolithic setups where upgrading one component breaks everything else. Docker Compose gives me the architectural separation I need: each service runs in its own container, with explicit dependencies and isolated environments.

The stack I've built consists of five main components. At the center is n8n itself, the automation orchestrator that runs my workflows. Behind it sits PostgreSQL as the production database—I tried SQLite early on, but the file-locking issues and lack of concurrent write support made it unsuitable for anything beyond toy projects. pgAdmin runs alongside for those moments when I need to inspect database state or run manual queries. Nginx Proxy Manager handles all the reverse proxy work, SSL termination, and Let's Encrypt certificate management. Finally, rclone on the host system pushes nightly backups to Google Drive.

This separation has proven invaluable. Last month I upgraded n8n to test a new feature, and when it caused issues with one of my workflows, I simply rolled back the n8n container without touching the database or proxy. The architectural boundaries enforce discipline and make disaster recovery straightforward—each piece can be replaced or restored independently.

## 4. Managing Secrets Properly from Day One

I've reviewed enough homelab setups to know that secret management is where most people cut corners. I've seen `.env` files committed to GitHub, passwords hardcoded in Compose files, and database credentials stored in plaintext on the filesystem with world-readable permissions. Don't do this. Even in a homelab, proper secret hygiene matters—not just for security, but for operational sanity when you need to rotate credentials or share your setup with a colleague.

I keep all sensitive configuration in a single `.env` file in the project directory. This file contains database credentials, n8n's basic auth settings, the encryption key for stored credentials, and the webhook domain configuration. The encryption key is particularly important—n8n uses it to encrypt OAuth tokens and other secrets it stores in the database. If you lose this key, you lose access to all those stored credentials. I generate it using a cryptographically secure random source and immediately back it up separately from the system.

```env
POSTGRES_USER=n8nprod
POSTGRES_PASSWORD=supersecretpassword
POSTGRES_DB=n8n
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=automation
N8N_BASIC_AUTH_PASSWORD=change_me_later
N8N_ENCRYPTION_KEY=<32 char random string>
N8N_HOST=yourdomain.tld
N8N_PROTOCOL=https
N8N_PORT=443
```

The first thing I do after creating this file is lock down its permissions. Only the user running Docker should be able to read it, which I enforce with `chmod 600 .env`. This prevents other users on the system (or compromised services) from accessing the secrets. In a more paranoid setup, you could use Docker secrets, sops for encrypted config files, or even HashiCorp Vault, but for a homelab, strict file permissions and regular password rotation get you 95% of the way there.

## 5. Composing the Stack: Services, Dependencies, and Volumes

The Docker Compose file is where everything comes together. I've iterated on this configuration over several months, and what you see below reflects the lessons I've learned about dependency ordering, volume mounting, and environment variable injection. The key insight is that Compose manages not just containers, but the relationships between them—PostgreSQL must start before n8n, volumes must persist across restarts, and environment variables must be sourced securely.

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - ./pgdata:/var/lib/postgresql/data
  
  pgadmin:
    image: dpage/pgadmin4
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@yourdomain.tld
      PGADMIN_DEFAULT_PASSWORD: another_secret_here
    ports:
      - "8081:80"
  
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: ${POSTGRES_DB}
      DB_POSTGRESDB_USER: ${POSTGRES_USER}
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
      N8N_BASIC_AUTH_ACTIVE: ${N8N_BASIC_AUTH_ACTIVE}
      N8N_BASIC_AUTH_USER: ${N8N_BASIC_AUTH_USER}
      N8N_BASIC_AUTH_PASSWORD: ${N8N_BASIC_AUTH_PASSWORD}
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
      N8N_HOST: ${N8N_HOST}
      N8N_PROTOCOL: ${N8N_PROTOCOL}
      N8N_PORT: ${N8N_PORT}
    ports:
      - "5678:5678"
    depends_on:
      - postgres
    volumes:
      - ./n8n_data:/home/node/.n8n
  
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - ./nginx/data:/data
      - ./nginx/letsencrypt:/etc/letsencrypt
```

The volume mounts deserve special attention. For PostgreSQL, I'm mounting `./pgdata` to persist the database across container restarts. This directory on the host contains all the database files, and it's what I'll be backing up. For n8n, the `./n8n_data` mount stores workflow definitions, credentials, and execution history. Nginx Proxy Manager needs two volumes—one for its configuration database and another for Let's Encrypt certificates.

I use `restart: unless-stopped` for all services rather than `restart: always`. The distinction matters: if I manually stop a container for maintenance, I don't want it automatically restarting. But if the Pi reboots or Docker daemon restarts, everything should come back up without manual intervention.

## 6. Exposing n8n Securely with Nginx Proxy Manager

![Network flow diagram showing: external traffic on port 80/443 → Nginx Proxy Manager (TLS termination) → internal n8n service on port 5678 over plain HTTP. Should illustrate the reverse proxy pattern and where SSL/TLS encryption happens.](https://www.samueldowling.com/wp-content/uploads/2020/01/reverse-proxy.png)



Running n8n directly on port 5678 with HTTP is fine for localhost testing, but completely unacceptable for production use. You need HTTPS for security, you need proper hostname routing, and you need automatic certificate renewal. I used to configure all of this manually with raw nginx configs and certbot, but Nginx Proxy Manager (NPM) has simplified my life considerably.

NPM provides a clean web interface—usually accessible on port 81—where you can configure proxy hosts, SSL certificates, and access controls without touching nginx config files. When I first set this up, I pointed `n8n.yourdomain.tld` to the internal n8n container at `n8n:5678`. NPM sits in front, listening on ports 80 and 443, and forwards requests to the appropriate backend based on the hostname.

The SSL configuration is remarkably straightforward. In the NPM interface, I enabled "Force SSL" and requested a Let's Encrypt certificate. NPM handles the ACME challenge automatically, as long as ports 80 and 443 are properly forwarded from your router. The certificates renew automatically every 60 days, and I've never had to intervene manually. This is the kind of automation that makes self-hosting sustainable—set it up once, and it just works.

What I particularly appreciate about this architecture is that all TLS termination happens at the proxy layer. n8n itself runs plain HTTP internally, which simplifies its configuration and reduces the attack surface. Only NPM needs access to private keys, and only NPM faces the public internet directly. This separation of concerns makes security auditing much easier.

## 7. Registering a Domain and Configuring Cloudflare

You can't have a proper HTTPS setup without a real domain name. I registered mine through Porkbun because their pricing is competitive and, crucially, they provide a straightforward DNS API for dynamic updates. The domain itself is the foundation for everything else—webhooks, SSL certificates, and external service integrations all depend on having a stable, trusted hostname.

After registering the domain, I immediately changed the nameservers to Cloudflare. This might seem like an unnecessary step, but Cloudflare's global DNS network provides significantly faster propagation and better DDoS protection than most registrars' default nameservers. More importantly, Cloudflare's API is more robust than Porkbun's for programmatic updates, though I ended up using Porkbun's API anyway for DDNS because it's simpler for single-record updates.

In Cloudflare's DNS management interface, I created an A record for `n8n.yourdomain.tld` pointing to my home's public IP address. I set the TTL to 300 seconds (five minutes) rather than Cloudflare's default "Auto." This shorter TTL means that when my dynamic IP changes, the old DNS entry expires quickly, minimizing downtime. It's a trade-off—shorter TTLs mean more DNS queries, but for a single-server homelab, the resolution load is negligible.

One configuration choice I deliberated over was Cloudflare's proxy option (the orange cloud icon). When enabled, Cloudflare proxies all traffic through their edge network, hiding your home IP and providing DDoS protection. I ultimately disabled it because it complicates Let's Encrypt validation and adds latency to webhook responses. My threat model doesn't require that level of protection, and I value the directness of having external services connect directly to my Pi.

## 8. Implementing Dynamic DNS to Track IP Changes

![Timeline/sequence diagram showing: ISP changes home IP → DDNS script detects change (via ipify.org) → Porkbun API updates DNS record → Cloudflare propagates → external services resolve to new IP. Should include the 15-minute check interval and TTL expiration windows.](https://www.paloaltonetworks.com/content/dam/pan/en_US/images/cyberpedia/what-is-dynamic-dns/DNSDynamic2025_1-DDNS.png?imwidth=480)



Most residential ISPs don't provide static IP addresses, which presents an obvious problem: if your public IP changes, your DNS record becomes stale, and nobody can reach your n8n instance. I've seen my IP change as often as twice a month, usually during maintenance windows or after brief outages. Dynamic DNS (DDNS) solves this by automatically updating your DNS record whenever your IP changes.

Porkbun offers a DDNS-specific API endpoint that's simpler than their full DNS API. I wrote a bash script that checks my current public IP using ipify.org, compares it to what DNS currently returns, and updates the record only if they differ. This approach minimizes unnecessary API calls and makes the logs easier to read when troubleshooting.

```bash
#!/bin/bash
API_KEY="YOUR_PORKBUN_API_KEY"
SECRET_KEY="YOUR_PORKBUN_API_SECRET"
DOMAIN="yourdomain.tld"
SUBDOMAIN="n8n"

CUR_IP=$(curl -s https://api.ipify.org)
DNS_IP=$(dig +short ${SUBDOMAIN}.${DOMAIN} @1.1.1.1)

if [ "$CUR_IP" != "$DNS_IP" ]; then
  curl -s -X POST "https://porkbun.com/api/json/v3/dns/edit/${DOMAIN}/${SUBDOMAIN}" \
    -H "Content-Type: application/json" \
    -d '{"apikey":"'$API_KEY'","secretapikey":"'$SECRET_KEY'","content":"'$CUR_IP'","type":"A","ttl":600}'
  
  echo "$(date): Updated DNS from $DNS_IP to $CUR_IP" >> /var/log/ddns-update.log
fi
```

The script is deliberately conservative. By querying Cloudflare's DNS resolver (1.1.1.1) rather than my local resolver, I avoid cached results that might mask the real DNS state. The conditional update means I only touch Porkbun's API when necessary, respecting rate limits and keeping logs clean. I've been running this script every 15 minutes for months, and it's caught IP changes within that window every time.

Error handling in production DDNS scripts matters more than you might expect. I initially had no retry logic, and once when Porkbun's API was briefly unavailable, my script failed silently. I now wrap the curl command in basic error checking and log failures distinctly. If three consecutive updates fail, the script sends me a notification through one of my n8n workflows—a nice example of self-hosted systems monitoring themselves.

## 9. Configuring Your Router for Reliable Access

![Network topology diagram showing: Internet → Router (with port forwarding rules 80/443) → Raspberry Pi (192.168.1.100 with DHCP reservation) → Nginx/n8n services. Should illustrate NAT, port forwarding rules, and internal IP allocation.](https://preview.redd.it/updated-proposed-diagram-for-home-network-v2-0-v0-qvtkw5kz5gqa1.png?auto=webp&s=d0ae1f2386ab3d9e7a52e287cdb5fd08f96f87d8)



Your router is the gatekeeper between the internet and your homelab, and it needs two specific configurations to make this setup work reliably. First, you need a DHCP reservation for your Raspberry Pi, ensuring it always receives the same local IP address. Second, you need port forwarding rules that direct incoming traffic on ports 80 and 443 to your Pi.

DHCP reservations are usually configured based on the Pi's MAC address. I logged into my router's admin interface (if you're not sure how, I found this guide on configuring routers as access points helpful for understanding the general navigation patterns), found the DHCP section, and created a static lease. Now, no matter how many times the Pi reboots or renews its lease, it always gets 192.168.1.100. This consistency is essential because port forwarding rules reference specific internal IP addresses.

Port forwarding is where external traffic on your public IP gets directed to internal devices. I created two forwarding rules: TCP port 80 forwarding to 192.168.1.100:80, and TCP port 443 forwarding to 192.168.1.100:443. These rules ensure that when Let's Encrypt tries to validate your domain by connecting to `yourdomain.tld:80`, and when browsers connect to `https://n8n.yourdomain.tld:443`, the traffic reaches Nginx Proxy Manager on your Pi.

Testing port forwarding can be frustrating because most routers implement NAT reflection poorly or not at all. From inside your home network, you might not be able to access `n8n.yourdomain.tld` even though everything is configured correctly. I use my phone's cellular connection for testing—switching off WiFi and connecting to the domain verifies that external access works. If you see connection timeouts, check firewall rules on both your router and the Pi itself.

## 10. Securing Webhooks and Public-Facing Endpoints

![Security layers diagram showing webhook request journey: external service → HTTPS encryption → n8n webhook endpoint → signature verification node → workflow execution. Should visualize the layered defense approach (HTTPS, basic auth, signature verification, UFW firewall).](https://www.researchgate.net/publication/274733863/figure/fig1/AS:294669188648961@1447266017379/Layers-of-defense-in-depth-architecture.png)



Webhooks are n8n's superpower—they let external services trigger your workflows in real-time. But they also represent a security challenge because you're deliberately exposing an endpoint to the internet. I've spent considerable time thinking through the security implications, and I've landed on a layered approach that balances accessibility with protection.

HTTPS is non-negotiable. Every webhook URL must use the `https://` scheme, enforced by Nginx Proxy Manager's "Force SSL" setting. This encrypts webhook payloads in transit, preventing eavesdropping and man-in-the-middle attacks. When services like Stripe or GitHub send you sensitive data via webhook, that data is traveling across the public internet—encryption protects it.

n8n's basic authentication adds another layer. I've enabled it globally through environment variables, requiring a username and password for all access to the n8n interface. This doesn't directly protect webhooks—most webhook senders don't support basic auth—but it prevents unauthorized access to your workflow editor, where someone could view or modify your automations.

For webhook-specific security, I rely on signature verification. Most modern webhook providers include a signature header that you can validate within your n8n workflow. For example, Stripe signs each webhook with a secret key, and your workflow can verify that signature before processing the payload. This proves the webhook came from Stripe and hasn't been tampered with. I implement this verification as the first node in every webhook workflow—if the signature fails, the workflow terminates immediately.

I've also configured UFW (Uncomplicated Firewall) on the Pi to restrict which ports are accessible. Only ports 80, 443, and SSH from my local network are allowed inbound. This defense-in-depth approach means that even if there's a vulnerability in n8n or Nginx Proxy Manager, an attacker can't easily pivot to other services or the underlying system.

## 11. Automating PostgreSQL Backups with rclone

Data loss is inevitable if you run any system long enough. I've experienced SD card corruption, accidental deletions, and one memorable incident where a buggy workflow update corrupted several database tables. The only defense is comprehensive, automated backups—stored offsite where local disasters can't touch them.

I use rclone to push PostgreSQL backups to Google Drive every night. rclone is remarkable in its simplicity and power—it speaks native protocols for dozens of cloud storage providers, handles resumption and retries, and uses minimal resources. Configuration involves running `rclone config` once, authenticating with Google via OAuth, and saving the credentials locally. After that initial setup, rclone can sync files unattended.

My backup script runs nightly via cron. It dumps the entire n8n database to a timestamped SQL file, syncs the backup directory to Google Drive, and cleans up backups older than 14 days. The script is straightforward, but the details matter—using `PGPASSWORD` to avoid interactive prompts, ensuring the backup directory exists before writing, logging to a separate file for troubleshooting.

```bash
#!/bin/bash
BACKUP_DIR="/home/pi/backups/n8n"
DATE=$(date +'%Y-%m-%d')
mkdir -p $BACKUP_DIR

# Dump PostgreSQL to a timestamped file
PGPASSWORD="supersecretpassword" pg_dump -U n8nprod -h localhost n8n > $BACKUP_DIR/n8n_backup_$DATE.sql

# Sync backups with Google Drive
rclone sync $BACKUP_DIR gdrive:n8n-backups --backup-dir gdrive:n8n-backups/archive/$(date +%Y)

# Delete old local backups (older than 14 days)
find $BACKUP_DIR -type f -mtime +14 -delete

echo "$(date): Backup completed successfully" >> /var/log/n8n_backup.log
```

The `--backup-dir` flag deserves explanation. When rclone encounters files that would be deleted or overwritten during sync, it moves them to the backup directory instead. This protects against accidental deletions—if I corrupt a workflow and it gets backed up before I notice, I can recover the previous state from the yearly archive.

I learned through painful experience to actually test my backups. Once every few months, I restore a backup to a separate Docker container and verify that it works. I've caught issues where the backup completed successfully but the SQL dump was truncated or missing tables. Testing restoration is the only way to know your backups are viable.

## 12. Scheduling Automation with Cron and Systemd

Reliability in a homelab setup depends heavily on automation that doesn't require manual intervention. I use two mechanisms for scheduling recurring tasks: traditional cron for the nightly backup script, and systemd timers for the DDNS updater. Both approaches have their place, and understanding when to use each has improved my system's robustness.

The backup script runs via cron at 2 AM daily. I chose this time because it's when my workflows are least active, minimizing the risk of backing up in the middle of a long-running execution. The cron entry pipes output to a log file, so I have a record of every backup attempt. This logging has been invaluable when investigating backup failures—I can see exactly what pg_dump or rclone reported.

```cron
0 2 * * * /home/pi/scripts/n8n_backup.sh >> /var/log/n8n_backup.log 2>&1
```

For the DDNS updater, I use systemd timers instead of cron. Systemd timers offer better logging integration, automatic retries, and clearer dependency management. The timer runs every 15 minutes, which gives me a reasonable balance between responsiveness and not hammering Porkbun's API. I created both a service unit and a timer unit.

```ini
# /etc/systemd/system/porkbun-ddns.service
[Unit]
Description=Porkbun DDNS updater

[Service]
Type=oneshot
ExecStart=/home/pi/scripts/porkbun-ddns.sh
User=pi
```

```ini
# /etc/systemd/system/porkbun-ddns.timer
[Unit]
Description=Run DDNS updater every 15min

[Timer]
OnCalendar=*:0/15
Persistent=true
Unit=porkbun-ddns.service

[Install]
WantedBy=timers.target
```

The `Persistent=true` setting means that if the timer would have fired while the system was off, it fires immediately on boot. This ensures IP changes during downtime get caught quickly. I enable and start the timer with `sudo systemctl enable --now porkbun-ddns.timer`, and from that point forward, it runs automatically.

Systemd's logging integration means I can check timer status and view recent executions with `systemctl status porkbun-ddns.timer` and `journalctl -u porkbun-ddns.service`. This centralized logging makes troubleshooting significantly easier than parsing scattered cron logs.

## 13. Hardening Security for Internet-Facing Services

Exposing any service to the internet increases your attack surface, and homelabs are often particularly vulnerable because they lack the security infrastructure of enterprise environments. I don't pretend my setup is impenetrable, but I've implemented several layers of defense that significantly raise the bar for potential attackers.

UFW (Uncomplicated Firewall) is my first line of defense. Despite the name, it's quite powerful, and its simple syntax makes it easy to maintain firewall rules without reference documentation. I've configured it to allow HTTP and HTTPS from anywhere—necessary for webhooks—and SSH only from my local network. This means even if someone discovers an SSH vulnerability, they can't exploit it from outside my home.

```bash
sudo ufw allow 80,443/tcp
sudo ufw allow from 192.168.1.0/24 proto tcp to any port 22
sudo ufw enable
```

I keep all Docker images updated aggressively. n8n releases updates regularly, often with security fixes that don't make headlines. I've set up a monthly maintenance window where I pull the latest images and restart containers. This is also when I review access logs, check for unusual activity, and verify that backups are completing successfully.

Operating system updates are just as important. Raspberry Pi OS releases security updates frequently, and I apply them weekly with `sudo apt update && sudo apt upgrade`. I've enabled automatic security updates for critical patches, accepting the small risk of something breaking in exchange for not running known-vulnerable software.

Password hygiene matters, even in a homelab. I rotate the n8n basic auth password quarterly, use strong random passwords everywhere, and never reuse passwords across services. The `.env` file with all these secrets is backed up separately and stored encrypted. If I need to share access with someone, I use temporary credentials and revoke them immediately after.

I disabled root login over SSH and use SSH keys exclusively. Password authentication is disabled entirely, which eliminates an entire class of brute-force attacks. My SSH private key is passphrase-protected and lives only on my primary workstation—never in the cloud, never on a phone.

## 14. What I've Learned After Months in Production

Running this setup in production has taught me lessons you can't learn from documentation. The first few weeks were rocky—Let's Encrypt certificate renewals failed mysteriously, webhooks timed out intermittently, and I experienced one database corruption event that I still don't fully understand. But after the initial debugging period, the system has been remarkably stable.

Memory is the primary constraint on a Raspberry Pi 4. With PostgreSQL, n8n, Nginx Proxy Manager, and pgAdmin all running, I hover around 3.2GB of RAM usage under normal load. When workflows execute heavily concurrent operations, I've seen spikes to 3.8GB. The 4GB Pi handles this, but not comfortably—a Pi with 8GB would provide more headroom. I've considered adding swap on the SSD, but good memory discipline in my workflows has kept me from needing it.

Storage I/O was a bottleneck until I switched to SSD. With a microSD card, database queries would occasionally take seconds, and I experienced periodic corruption requiring restores from backup. The SSD eliminated both issues entirely. If you take nothing else from this article, take this: use an SSD for any serious self-hosted setup.

DDNS timing is trickier than I expected. My ISP changes my IP without warning, usually in the middle of the night. With a 15-minute DDNS update interval and a 5-minute DNS TTL, there's a window where my domain resolves to the old IP. During this window, webhooks fail. I've learned to configure webhook sources with automatic retries when possible, so they'll try again after DNS propagates.

Let's Encrypt certificate renewal occasionally fails, almost always because of port 80 being momentarily unreachable. I've never determined the root cause—sometimes it's my router rebooting for firmware updates, sometimes it's ISP maintenance. Nginx Proxy Manager retries automatically, and with 60-day certificate lifetimes renewed at 30 days, there's plenty of buffer. Still, I monitor certificate expiration dates and get nervous when they drop below 20 days.

Testing backups revealed an embarrassing gap in my initial setup. I was backing up the database successfully, but I wasn't backing up the n8n data directory containing workflow definitions and custom nodes. A workflow I'd spent hours developing disappeared after an errant Docker volume pruning command. I learned that backups aren't complete unless you've documented what needs to be backed up and tested restoration of all those pieces.

## 15. Why This Approach Works and What It Enables

After running this setup for months, I've come to appreciate how the architectural decisions reinforce each other. Docker Compose provides modularity and rollback capabilities. Nginx Proxy Manager handles the complexity of HTTPS and certificate management. Dynamic DNS keeps me accessible despite a residential ISP. Automated backups provide confidence and recovery options. Each piece is replaceable, testable, and maintainable independently.

The benefits extend beyond the technical. I have complete visibility into my automation workflows—no cloud provider can see my logic, my data, or my API keys. When I need to troubleshoot, I have direct access to logs, databases, and process information. When I want to experiment with alpha-quality n8n features, I can do so without affecting a production SaaS account.

The cost savings are significant but not the primary motivation. I've invested perhaps $150 in hardware and maybe 20 hours in initial setup and learning. Compare that to n8n's cloud pricing at scale, where costs can easily reach $50-100 monthly for moderate usage. The payback period is measured in months, not years.

More importantly, this setup has become a platform for further homelab projects. I've added monitoring with Prometheus and Grafana, also running on the same Pi. I've integrated n8n with Home Assistant for home automation workflows. I'm experimenting with local AI models for workflow enrichment. Having a stable, production-grade n8n instance created possibilities I hadn't anticipated.

The challenges are real—this isn't a turnkey solution like signing up for a SaaS platform. You need to understand Linux, networking, Docker, and security fundamentals. When something breaks, there's no support ticket to file. But for engineers who value learning and control, who want to understand their tools deeply rather than treating them as black boxes, self-hosting n8n on a Raspberry Pi offers rewards that far exceed the effort required.

If you're reading this and wondering whether to attempt it yourself, consider your comfort with troubleshooting and your tolerance for occasional downtime. If those don't scare you, I encourage you to try it. Start small, test thoroughly, and build confidence incrementally. The skills you'll develop and the infrastructure you'll create will serve you far beyond this single project. Most importantly, you'll own your automation stack completely—no vendor lock-in, no surprise pricing changes, no terms of service alterations. In an era of increasing cloud dependence, that independence is valuable.