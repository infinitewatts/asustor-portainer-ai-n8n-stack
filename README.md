# n8n + AI Starter Stack for Asustor NAS (No SSH Required)

A **production-ready template** to run n8n workflow automation with Ollama (AI), Qdrant (vector DB), and PostgreSQL on your Asustor NAS using Portainer's web interface.

![Asustor NAS Compatible](https://img.shields.io/badge/Asustor-ADM_4.0%2B-blue?logo=asustor&style=flat-square)
![Portainer Required](https://img.shields.io/badge/Requires-Portainer_CE_2.18%2B-13bdfd?logo=portainer&style=flat-square)
![n8n](https://img.shields.io/badge/n8n-latest-FF6D5A?style=flat-square&logo=n8n)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17-336791?style=flat-square&logo=postgresql)

## What's New (January 2026)

- **PostgreSQL 17**: Upgraded from 16 to 17-alpine for improved performance
- **Llama 3.2**: Updated default model from Llama 3 to Llama 3.2 (smaller, faster)
- **Health Checks**: Added health checks for all services for better reliability
- **Resource Limits**: Added memory limits optimized for NAS devices
- **Timezone Support**: Added configurable timezone via `GENERIC_TIMEZONE` env var
- **Network Fix**: Fixed network configuration (was commented out)

> **Note about n8n 2.0**: n8n 2.0 was released in December 2025 with breaking changes. This stack uses `latest` which will include 2.0+. If you're upgrading from an older installation, please review the [n8n 2.0 migration guide](https://docs.n8n.io/2-0-breaking-changes/) before updating.

## Features

- **100% Web Interface Setup** - No terminal/SSH required
- **Asustor ADM 4.0+ Optimized** - Verified on AS53/54/67 series and Flashstor FS6706T
- **Secure Defaults** - Pre-configured with encryption & access controls
- **Local LLM Option** - Ollama included (see performance note below)
- **Health Monitoring** - Built-in health checks for all services
- **Resource Managed** - Memory limits prevent NAS overload

> **Performance Note on Ollama**: Running local LLMs via Ollama on NAS devices is **slow**. NAS CPUs (like Intel Celeron) lack the processing power for responsive AI inference. For production AI workflows, we recommend using **cloud-based LLM APIs** (OpenAI, Anthropic, etc.) instead. Ollama is included for experimentation but expect slow response times.

## Service Versions

| Service | Version | Purpose |
|---------|---------|---------|
| n8n | latest | Workflow automation |
| PostgreSQL | 17-alpine | Database backend |
| Ollama | latest | Local LLM inference (slow on NAS) |
| Qdrant | latest (v1.16+) | Vector database |
| Cloudflared | latest | Secure tunnel |

## Recommended AI Setup

For the best experience with AI workflows on your NAS:

| Option | Speed | Cost | Recommendation |
|--------|-------|------|----------------|
| **Cloud APIs** (OpenAI, Anthropic, Google) | Fast | Pay per use | **Recommended for production** |
| **Ollama on NAS** | Slow | Free | Experimentation only |

n8n has built-in nodes for OpenAI, Anthropic Claude, Google Gemini, and more. These provide much faster responses than running models locally on NAS hardware.

## Installation Guide

### Prerequisites
1. Asustor NAS with [Portainer CE](https://www.asustor.com/en-gb/online/College_topic?topic=350) installed
2. (Optional) [Cloudflare Tunnel](https://www.asustor.com/en-gb/online/College_topic?topic=349) for secure remote access
3. Minimum 8GB RAM recommended (16GB+ for larger models)

### Step 1 - Prepare Environment File
1. In Portainer, create new **Environment Variables**:
```env
# .env file template
POSTGRES_USER=admin
POSTGRES_PASSWORD=your_strong_db_password
POSTGRES_DB=n8n_production

N8N_ENCRYPTION_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
N8N_USER_MANAGEMENT_JWT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxx
TUNNEL_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxx

# Optional: Set your timezone (default: UTC)
GENERIC_TIMEZONE=America/New_York
```

### Step 2 - Deploy Stack
1. In Portainer, go to **Stacks** > **Add Stack**
2. Name: `n8n-ai-stack`
3. Build Method: **Web editor**
4. Paste this compose file:
```yaml
volumes:
  n8n_storage:
  postgres_storage:
  ollama_storage:
  qdrant_storage:

networks:
  work-net:
    driver: bridge
    # Uncomment below to use an existing external network
    # external: true

x-n8n-environment: &service-n8n-environment
  DB_TYPE: postgresdb
  DB_POSTGRESDB_HOST: postgres
  DB_POSTGRESDB_USER: ${POSTGRES_USER}
  DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
  N8N_DIAGNOSTICS_ENABLED: false
  N8N_PERSONALIZATION_ENABLED: false
  N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
  N8N_USER_MANAGEMENT_JWT_SECRET: ${N8N_USER_MANAGEMENT_JWT_SECRET}
  OLLAMA_HOST: ollama:11434

x-n8n: &service-n8n
  image: n8nio/n8n:latest
  networks: ['work-net']
  environment: &service-n8n-env
    <<: *service-n8n-environment

x-ollama: &service-ollama
  image: ollama/ollama:latest
  container_name: ollama
  networks: ['work-net']
  restart: unless-stopped
  ports:
    - 11434:11434
  volumes:
    - ollama_storage:/root/.ollama
  healthcheck:
    test: ['CMD-SHELL', 'ollama list || exit 1']
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 30s
  deploy:
    resources:
      limits:
        memory: 4G

x-init-ollama: &init-ollama
  image: ollama/ollama:latest
  networks: ['work-net']
  container_name: ollama-pull-llama
  volumes:
    - ollama_storage:/root/.ollama
  entrypoint: /bin/sh
  environment:
    - OLLAMA_HOST=ollama:11434
  command:
    - "-c"
    - "sleep 3; ollama pull llama3.2"

services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    networks: ['work-net']
    restart: unless-stopped
    command: tunnel --url http://n8n:5678 --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
    deploy:
      resources:
        limits:
          memory: 256M

  postgres:
    image: postgres:17-alpine
    hostname: postgres
    networks: ['work-net']
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_storage:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -h localhost -U ${POSTGRES_USER} -d ${POSTGRES_DB}']
      interval: 5s
      timeout: 5s
      retries: 10
    deploy:
      resources:
        limits:
          memory: 512M

  n8n:
    <<: *service-n8n
    hostname: n8n
    container_name: n8n
    restart: unless-stopped
    environment:
      <<: *service-n8n-env
      DB_POSTGRESDB_DATABASE: ${POSTGRES_DB}
      N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS: "true"
      GENERIC_TIMEZONE: ${GENERIC_TIMEZONE:-UTC}
      TZ: ${GENERIC_TIMEZONE:-UTC}
    ports:
      - 5678:5678
    volumes:
      - n8n_storage:/home/node/.n8n
      - ./shared:/data/shared
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ['CMD-SHELL', 'wget -qO- http://localhost:5678/healthz || exit 1']
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 1G

  qdrant:
    image: qdrant/qdrant:latest
    hostname: qdrant
    container_name: qdrant
    networks: ['work-net']
    restart: unless-stopped
    ports:
      - 6333:6333
    volumes:
      - qdrant_storage:/qdrant/storage
    healthcheck:
      test: ['CMD-SHELL', 'wget -qO- http://localhost:6333/readyz || exit 1']
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    deploy:
      resources:
        limits:
          memory: 1G

  ollama-cpu:
    <<: *service-ollama

  ollama-pull-llama-cpu:
    <<: *init-ollama
    depends_on:
      - ollama-cpu
```
5. Enable **Auto-update** (recommended)

### Step 3 - First Run Setup
1. After deployment completes (2-5 mins), access:
   - Local: `http://your-nas-ip:5678`
   - Remote: `https://n8n.your-domain.com` (if using Cloudflare)
2. **First-time login:**
   - Username: `your@email.com`
   - Password: 'whateverpasswordyouwant'

## Memory Requirements

| NAS RAM | Recommendation |
|---------|----------------|
| 4GB | Not recommended - may experience issues |
| 8GB | Minimum - use Llama 3.2 (3B) only |
| 16GB | Recommended - can run larger models |
| 32GB+ | Optimal - can run 13B+ models |

## Updating

Portainer will auto-update images when "Auto-update" is enabled. To manually update:

1. In Portainer > Stacks > `n8n-ai-stack`
2. Click **Update**
3. Check "Pull latest image"
4. Click **Update the stack**

### Upgrading to n8n 2.0

If upgrading from n8n 1.x to 2.0:
1. **Backup your workflows** first
2. Review [breaking changes](https://docs.n8n.io/2-0-breaking-changes/)
3. Test in a staging environment if possible
4. Key changes: Task runner now requires separate image for external mode

## Troubleshooting

### Services won't start
- Check Portainer logs for each container
- Verify environment variables are set correctly
- Ensure ports 5678, 6333, 11434 are not in use

### Ollama model download fails
- The init container needs network access
- Check if your NAS firewall allows outbound connections
- Try manually pulling: access Ollama container and run `ollama pull llama3.2`

### n8n shows "unhealthy"
- Wait for the start period (30 seconds)
- Check PostgreSQL is healthy first
- Verify the database credentials match

### Out of memory errors
- Reduce Ollama memory limit in compose file
- Use smaller models (llama3.2 instead of llama3.1)
- Close other applications on NAS

## Security Checklist

- [ ] Change default admin credentials
- [ ] Enable Cloudflare Access Policies
- [ ] Rotate encryption keys quarterly
- [ ] Monitor Portainer audit logs
- [ ] Keep images updated regularly

## Weekend Warrior Appreciation

After spending my **entire Saturday night** and **Sunday** debugging (including multiple Portainer wipes and NAS resets!), this stack finally works seamlessly with Cloudflared. If this setup saves you from similar headaches:

[![Cash App](https://img.shields.io/badge/Support_My_Late_Night_Coding-00C244?style=for-the-badge&logo=cashapp&label=Cash%20App&logoColor=white)](https://cash.app/$JohnEricHuggins89)
**[Direct Cash App Link](https://cash.app/$JohnEricHuggins89)**

Your support helps fund:
- More Asustor-specific optimizations
- Documentation improvements
- Future-proof maintenance (so you don't have to wipe your NAS like I did!)

*"The only thing I wiped more than my NAS this weekend was my brow!"*

---

**Maintained with care by Eric**

## Resources

- [n8n Documentation](https://docs.n8n.io/)
- [n8n 2.0 Breaking Changes](https://docs.n8n.io/2-0-breaking-changes/)
- [Ollama Models Library](https://ollama.com/library)
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [Portainer Documentation](https://docs.portainer.io/)
