# Production Infrastructure — GKWebTech

Self-managed production VPS hosting 3 live websites, automation workflows, and a self-hosted ERP system. Sole technical owner responsible for all deployments, uptime, and incident response.

---

## Architecture Overview

```
Internet
    │
    ▼
Hostinger VPS (KVM2)
    │
    ▼
Traefik v3 (Reverse Proxy + SSL Termination)
    │
    ├── gkwebtech.cloud          → React Frontend Container
    ├── api.gkwebtech.cloud      → Node.js Backend Container
    ├── institute.gkwebtech.cloud → Next.js Frontend Container
    ├── n8n.gkwebtech.cloud      → n8n Automation Container
    ├── admin.gkwebtech.cloud    → Dokploy Dashboard
    └── odoo.tiwariclinic.org    → Odoo ERP Container
```

---

## Infrastructure Stack

| Component | Technology |
|-----------|-----------|
| VPS | Hostinger KVM2 |
| Container Runtime | Docker |
| Orchestration | Docker Swarm |
| Reverse Proxy | Traefik v3 |
| Deployment Manager | Dokploy |
| SSL Certificates | Let's Encrypt (auto-renew via Traefik) |
| DNS Management | Hostinger DNS |

---

## Services Running

### gkwebtech.cloud
- **Type:** Full-stack MERN agency website
- **Stack:** React + Vite, Node.js + Express, MongoDB
- **Features:** Multilingual (EN/NL/DE), GDPR cookie consent, AI chatbot, contact form, automated blog pipeline
- **Deployment:** Dokploy + Nixpacks, auto-deploy on GitHub push

### institute.gkwebtech.cloud
- **Type:** Training institute marketing website
- **Stack:** Next.js, Tailwind CSS, MongoDB
- **Features:** Course catalog, lead capture, GDPR compliance, email automation
- **Deployment:** Dokploy + Nixpacks

### n8n.gkwebtech.cloud
- **Type:** Self-hosted automation platform
- **Stack:** n8n Community Edition, SQLite database
- **Data:** Persistent via Docker volume at `/root/n8n-data/`
- **Workflows:** 8+ active automation workflows

### admin.gkwebtech.cloud
- **Type:** Deployment management dashboard
- **Stack:** Dokploy
- **Purpose:** Manages all service deployments, environment variables, domain routing

### odoo.tiwariclinic.org
- **Type:** Self-hosted ERP system
- **Stack:** Odoo Community Edition, PostgreSQL, Redis
- **Features:** Inventory management, POS billing, CRM, attendance tracking
- **Deployment:** Docker Compose (standalone stack)

---

## Deployment Strategy

All application services are managed through Dokploy with auto-deployment triggered on GitHub push to the main branch.

Odoo ERP runs as a standalone Docker Compose stack separate from the Dokploy-managed services to avoid Traefik routing conflicts between Docker Swarm and Compose providers.

### Environment Variables
All sensitive configuration (database URIs, API keys, SMTP credentials) is managed through Dokploy's environment variable panel. No secrets are stored in repository code.

### Branch Strategy
- `main` branch is protected with mandatory pull request reviews
- All changes go through feature branches and PRs
- Dokploy triggers deployment only on merge to main

---

## Incident Report — Dokploy Data Corruption (February 2026)

### Summary
A critical production incident in the second week of February 2026 caused all three live websites to go offline due to Dokploy configuration corruption.

### What Happened
During a Dokploy service reload, the internal routing configuration was corrupted. Traefik lost all router and service definitions, causing every domain to return 502 or 404 errors. The Dokploy dashboard became inaccessible.

### Impact
- **Duration:** ~72 hours of degraded service
- **Affected:** All 3 websites, n8n automation workflows
- **Data loss:** Zero

### Diagnosis
1. Checked Docker service status — all containers were still running (`docker service ls`)
2. Confirmed data volumes were intact (`docker volume ls`)
3. Identified that only the Traefik routing layer was destroyed, not the underlying services or data
4. Root cause: Dokploy internal database corruption during reload

### Recovery Steps
1. Rebuilt Traefik configuration manually with correct router labels
2. Redeployed frontend and backend services from GitHub repositories via Dokploy
3. For n8n — located the SQLite workflow database at `/root/n8n-data/database.sqlite` inside the container volume
4. Exported all workflow JSON files manually from the running container
5. Redeployed fresh n8n instance with restored database volume
6. Verified all workflows and automations were functional

### Outcome
All services restored within 72 hours. Zero data loss. Zero workflow loss.

### Lessons Learned
- Always know where persistent data lives before a failure happens
- n8n workflows should be regularly exported as JSON backups to a separate location
- Dokploy and manual Docker Swarm configurations should not be mixed — keep one source of truth
- Set up external uptime monitoring (UptimeRobot) to detect outages immediately

---

## Traefik Routing Issue — Key Debug

During the Traefik v3 upgrade, encountered a routing syntax breaking change:

```yaml
# Traefik v2 syntax (broken in v3)
rule: Host(`domain1.com`,`domain2.com`)

# Traefik v3 correct syntax
rule: Host(`domain1.com`) || Host(`domain2.com`)
```

This caused 404 errors for the ERP domain while other services remained online. Took significant debugging to identify because error messages were not explicit about the syntax change.

---

## Odoo ERP Deployment Notes

### Why Odoo Instead of ERPNext
Initially attempted to deploy ERPNext v16 for clinic inventory management. Encountered persistent version compatibility conflicts between the ERPNext release and the Dokploy template image tags. After multiple failed attempts, pivoted to Odoo Community Edition which deployed successfully on first attempt.

### Deployment Approach
Odoo runs as a standalone Docker Compose stack rather than through Dokploy. This was a deliberate decision to avoid Traefik routing conflicts — Dokploy uses Docker Swarm mode while Odoo's official Docker setup uses Compose, and having Traefik watch both providers simultaneously caused intermittent routing failures.

### Configuration
Key Odoo configuration set post-deployment:
- Inventory module with batch tracking and FEFO expiry management
- POS system for clinic billing
- CRM module for patient/customer management
- Attendance tracking for staff
- RBAC configured — billing staff access restricted to relevant modules only

### Staff Training
Full clinic staff adoption achieved within one week of deployment. Staff initially resistant (preferred existing system) but accepted after hands-on training sessions.

---

## Backup Strategy

**Current setup:**
- Dokploy performs periodic backups stored on VPS
- n8n workflows exported as JSON files manually

**Planned improvements:**
- Automated PostgreSQL dumps via cron + rclone to Google Drive
- Automated n8n workflow export on schedule
- Off-VPS backup storage to protect against full server failure

---

## Repository Structure

```
/nginx/
    nginx.conf              # Nginx configuration reference
/odoo/
    docker-compose.yml      # Odoo stack deployment file
/traefik/
    traefik.yml             # Traefik static configuration
/docs/
    recovery-incident.md    # Detailed incident post-mortem
    odoo-setup-notes.md     # Odoo configuration steps
    traefik-v3-gotchas.md   # Traefik v3 breaking changes learned
```

---

## Key Learnings

- Docker containers are replaceable. Data volumes are not. Always know where your data lives.
- Traefik v3 has breaking syntax changes from v2 — read the migration guide before upgrading.
- Mixing Docker Swarm (Dokploy) with Docker Compose on the same Traefik instance causes routing conflicts. Use one approach consistently.
- Self-hosting ERPNext is significantly more complex than Odoo for a simple inventory use case. Know when to pivot.
- Production incident response is a skill. Document everything during recovery — it becomes the post-mortem.

---

*Maintained by [Utkarsh Sharma](https://github.com/Utkarsh9571)*
