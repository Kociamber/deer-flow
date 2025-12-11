# Security Recommendations for DeerFlow

## Priority Actions

### 🔴 CRITICAL: Fix urllib3 Vulnerability

**Current Status:** urllib3 2.3.0 (vulnerable)  
**Required Version:** urllib3 >= 2.6.0

#### How to Fix:

1. **Update pyproject.toml**
   Add or update the urllib3 dependency constraint:
   ```toml
   [project]
   dependencies = [
       # ... other dependencies ...
       "httpx>=0.28.1",
       "urllib3>=2.6.0",  # Add this line or update existing
       # ... rest of dependencies ...
   ]
   ```

2. **Update lock file**
   ```bash
   uv lock --upgrade-package urllib3
   ```

3. **Test the update**
   ```bash
   uv sync
   uv run pytest tests/
   ```

4. **Verify the version**
   ```bash
   uv pip list | grep urllib3
   # Should show: urllib3  2.6.0 (or higher)
   ```

#### Alternative: Update all dependencies
```bash
uv lock --upgrade
uv sync
```

---

## Production Deployment Configuration

### 1. Environment Variables

Create a production `.env` file:

```bash
# Security Settings - CRITICAL
ENABLE_PYTHON_REPL=false
ENABLE_MCP_SERVER_CONFIGURATION=false

# CORS - Update with your actual domain
ALLOWED_ORIGINS=https://yourdomain.com,https://www.yourdomain.com

# Application
APP_ENV=production
DEBUG=false

# API Keys - Use secrets management in production
TAVILY_API_KEY=your_actual_key_here
INFOQUEST_API_KEY=your_actual_key_here

# LangSmith Monitoring (Recommended)
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=your_langsmith_key
LANGSMITH_PROJECT=deerflow-production

# Database (if using checkpointing)
LANGGRAPH_CHECKPOINT_SAVER=true
LANGGRAPH_CHECKPOINT_DB_URL=postgresql://user:pass@host:5432/deerflow
```

### 2. Docker Deployment

**Use the provided Dockerfile:**
```bash
docker build -t deerflow:latest .
docker run -d \
  --name deerflow \
  -p 8000:8000 \
  --env-file .env.production \
  --restart unless-stopped \
  deerflow:latest
```

**With docker-compose:**
```yaml
version: '3.8'
services:
  deerflow:
    build: .
    ports:
      - "8000:8000"
    env_file:
      - .env.production
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
```

### 3. Nginx Reverse Proxy

**nginx.conf snippet:**
```nginx
upstream deerflow {
    server 127.0.0.1:8000;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req zone=api_limit burst=20 nodelay;

    location / {
        proxy_pass http://deerflow;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts for SSE
        proxy_read_timeout 300s;
        proxy_connect_timeout 75s;
    }
}
```

---

## Authentication & Authorization

### Option 1: API Key Authentication

Add middleware in `src/server/app.py`:

```python
from fastapi import Header, HTTPException
from typing import Optional

API_KEYS = set(os.getenv("API_KEYS", "").split(","))

async def verify_api_key(x_api_key: Optional[str] = Header(None)):
    if not x_api_key or x_api_key not in API_KEYS:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return x_api_key

# Add to endpoints:
@app.post("/api/chat/stream")
async def chat_stream(
    request: ChatRequest,
    api_key: str = Depends(verify_api_key)
):
    # ... existing code
```

### Option 2: JWT Authentication

```python
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import JWTError, jwt

security = HTTPBearer()

async def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    try:
        payload = jwt.decode(
            credentials.credentials,
            SECRET_KEY,
            algorithms=["HS256"]
        )
        return payload
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

---

## Monitoring & Logging

### 1. Enable LangSmith

```bash
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=your_key
LANGSMITH_PROJECT=deerflow-prod
```

### 2. Application Logging

**Structured logging setup:**
```python
import logging
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
        }
        return json.dumps(log_data)

handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logging.root.addHandler(handler)
```

### 3. Monitoring Tools

Consider integrating:
- **Prometheus** + Grafana for metrics
- **ELK Stack** for log aggregation
- **Sentry** for error tracking
- **Datadog** or **New Relic** for APM

---

## Network Security

### 1. Firewall Rules

```bash
# Allow only necessary ports
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp   # SSH (restrict to your IP)
ufw allow 443/tcp  # HTTPS
ufw enable
```

### 2. Network Segmentation

```
Internet
   ↓
[Load Balancer]
   ↓
[Web Tier] (nginx)
   ↓
[App Tier] (DeerFlow)
   ↓
[Data Tier] (PostgreSQL/MongoDB)
```

---

## Secrets Management

### Option 1: Environment Variables (Basic)
```bash
# Use separate .env files per environment
.env.development
.env.staging
.env.production
```

### Option 2: HashiCorp Vault (Recommended)
```python
import hvac

client = hvac.Client(url='http://vault:8200')
secrets = client.secrets.kv.v2.read_secret_version(path='deerflow')
os.environ['TAVILY_API_KEY'] = secrets['data']['data']['tavily_key']
```

### Option 3: AWS Secrets Manager
```python
import boto3

client = boto3.client('secretsmanager', region_name='us-east-1')
secret = client.get_secret_value(SecretId='deerflow/prod')
```

---

## Container Security

### 1. Scan Images
```bash
# Using Trivy
trivy image deerflow:latest

# Using Docker Scout
docker scout cves deerflow:latest
```

### 2. Run as Non-Root

Update Dockerfile:
```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-bookworm

# Create non-root user
RUN groupadd -r deerflow && useradd -r -g deerflow deerflow

WORKDIR /app
COPY . /app
RUN chown -R deerflow:deerflow /app

USER deerflow

CMD ["uv", "run", "python", "server.py"]
```

### 3. Read-Only Root Filesystem
```yaml
services:
  deerflow:
    read_only: true
    tmpfs:
      - /tmp
      - /app/logs
```

---

## Dependency Management

### 1. Regular Audits
```bash
# Weekly/monthly checks
pip-audit --desc

# Or use GitHub Dependabot
# Enable in .github/dependabot.yml
```

### 2. Pin Dependencies
```toml
# Already done in pyproject.toml with version constraints
dependencies = [
    "fastapi>=0.110.0",  # Minimum version
    "urllib3>=2.6.0",     # Security fix
]
```

### 3. Automated Updates

**GitHub Dependabot config:**
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

---

## Backup & Disaster Recovery

### 1. Database Backups
```bash
# PostgreSQL
pg_dump -h localhost -U user deerflow > backup_$(date +%Y%m%d).sql

# MongoDB
mongodump --uri="mongodb://localhost:27017/deerflow" --out=backup_$(date +%Y%m%d)
```

### 2. Configuration Backups
```bash
# Backup .env and conf.yaml
tar -czf config_backup_$(date +%Y%m%d).tar.gz .env conf.yaml
```

### 3. Automated Backups
```bash
# Cron job (daily at 2 AM)
0 2 * * * /path/to/backup_script.sh
```

---

## Incident Response Plan

### 1. Detection
- Monitor logs for suspicious activity
- Set up alerts for:
  - Multiple failed authentication attempts
  - Unusual API usage patterns
  - Errors in Python REPL (if enabled)
  - MCP server configuration changes

### 2. Response Steps
1. Identify the incident
2. Contain the threat (disable features, block IPs)
3. Investigate root cause
4. Remediate vulnerabilities
5. Document findings
6. Update security measures

### 3. Communication Plan
- Internal team notification
- Customer notification (if data breach)
- Regulatory compliance (GDPR, etc.)

---

## Compliance Considerations

### GDPR (if applicable)
- Implement data deletion on request
- Log consent for data processing
- Provide data export functionality
- Document data flows

### SOC 2 (if applicable)
- Access controls
- Audit logging
- Encryption at rest and in transit
- Regular security assessments

---

## Testing Security

### 1. Static Analysis
```bash
# Bandit for Python security issues
bandit -r src/

# Safety for dependency vulnerabilities
safety check
```

### 2. Dynamic Analysis
```bash
# OWASP ZAP for web vulnerabilities
zap-cli quick-scan http://localhost:8000

# Nikto for web server scanning
nikto -h http://localhost:8000
```

### 3. Penetration Testing
Consider hiring professional security researchers to:
- Test for SSRF vulnerabilities
- Attempt SQL injection
- Test authentication bypass
- Check for XSS vulnerabilities

---

## Security Training

Ensure team members understand:
- Secure coding practices
- OWASP Top 10
- Handling sensitive data
- Incident response procedures

---

## Contact Information

### Security Issues
- Create private security advisory on GitHub
- Email: [your-security-email]
- PGP Key: [if available]

### General Support
- GitHub Issues: https://github.com/Kociamber/deer-flow/issues
- Documentation: See README.md

---

**Last Updated:** December 11, 2025  
**Version:** 1.0  
**Status:** Active Recommendations
