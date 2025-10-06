# Deployment Guide

This guide covers deploying Gate22 in production environments.

## Deployment Options

Gate22 can be deployed in several ways:

1. **Docker Compose** (Recommended for small-to-medium deployments)
2. **Kubernetes** (Recommended for large-scale deployments)
3. **Cloud Platforms** (AWS, GCP, Azure)
4. **Managed Service** (gate22.aci.dev)

## Prerequisites

### Required Services

- PostgreSQL 15+ with pgvector extension
- Container orchestration (Docker, Kubernetes)
- TLS/SSL certificates
- Domain name and DNS

### Recommended Resources

**Minimum:**
- 2 CPU cores
- 4 GB RAM
- 20 GB storage

**Recommended:**
- 4+ CPU cores
- 8+ GB RAM
- 50+ GB storage (SSD)

## Docker Compose Deployment

### Step 1: Prepare Environment

```bash
# Clone repository
git clone https://github.com/lifeisnphard/gate22.git
cd gate22/backend

# Copy environment file
cp .env.example .env.production
```

### Step 2: Configure Environment

Edit `.env.production`:

```bash
# Environment
ENVIRONMENT=production

# Database
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_USER=gate22
POSTGRES_PASSWORD=<strong_password>
POSTGRES_DB=gate22

# Security
SECRET_KEY=<random_secret_key_min_32_chars>
JWT_SECRET_KEY=<random_jwt_secret_min_32_chars>

# OAuth
GOOGLE_CLIENT_ID=<your_google_client_id>
GOOGLE_CLIENT_SECRET=<your_google_client_secret>
GOOGLE_REDIRECT_URI=https://your-domain.com/v1/control-plane/auth/google/callback

# External Services
OPENAI_API_KEY=<your_openai_api_key>
SENTRY_DSN=<your_sentry_dsn>

# CORS
CORS_ORIGINS=https://your-frontend-domain.com

# Application URLs
APP_BASE_URL=https://your-domain.com
```

### Step 3: Configure Docker Compose

Create `compose.production.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: pgvector/pgvector:pg15
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - gate22
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  migration:
    build:
      context: .
      dockerfile: Dockerfile.migration
    env_file:
      - .env.production
    depends_on:
      postgres:
        condition: service_healthy
    command: alembic upgrade head
    networks:
      - gate22

  control_plane:
    build:
      context: .
      dockerfile: Dockerfile.control_plane
    env_file:
      - .env.production
    depends_on:
      migration:
        condition: service_completed_successfully
    networks:
      - gate22
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/v1/control-plane/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  mcp:
    build:
      context: .
      dockerfile: Dockerfile.mcp
    env_file:
      - .env.production
    depends_on:
      migration:
        condition: service_completed_successfully
    networks:
      - gate22
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/v1/mcp/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  virtual_mcp:
    build:
      context: .
      dockerfile: Dockerfile.virtual_mcp
    env_file:
      - .env.production
    depends_on:
      migration:
        condition: service_completed_successfully
    networks:
      - gate22
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8002/v1/virtual/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - control_plane
      - mcp
      - virtual_mcp
    networks:
      - gate22
    restart: unless-stopped

networks:
  gate22:
    driver: bridge

volumes:
  postgres_data:
```

### Step 4: Configure Nginx

Create `nginx.conf`:

```nginx
events {
    worker_connections 1024;
}

http {
    upstream control_plane {
        server control_plane:8000;
    }

    upstream mcp {
        server mcp:8001;
    }

    upstream virtual_mcp {
        server virtual_mcp:8002;
    }

    # Redirect HTTP to HTTPS
    server {
        listen 80;
        server_name your-domain.com;
        return 301 https://$server_name$request_uri;
    }

    # HTTPS Server
    server {
        listen 443 ssl http2;
        server_name your-domain.com;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        
        # SSL configuration
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        # Security headers
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # Control Plane API
        location /v1/control-plane {
            proxy_pass http://control_plane;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # MCP Service
        location /v1/mcp {
            proxy_pass http://mcp;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        # Virtual MCP Service
        location /v1/virtual {
            proxy_pass http://virtual_mcp;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
```

### Step 5: Deploy

```bash
# Build and start services
docker compose -f compose.production.yml up -d

# Check status
docker compose -f compose.production.yml ps

# View logs
docker compose -f compose.production.yml logs -f

# Load MCP servers
./scripts/load_mcp_servers.sh
```

## Kubernetes Deployment

### Step 1: Create Namespace

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gate22
```

### Step 2: Create Secrets

```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: gate22-secrets
  namespace: gate22
type: Opaque
stringData:
  postgres-password: <strong_password>
  secret-key: <random_secret_key>
  jwt-secret-key: <random_jwt_secret>
  google-client-secret: <google_client_secret>
  openai-api-key: <openai_api_key>
```

### Step 3: PostgreSQL Deployment

```yaml
# postgres.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: gate22
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: gate22
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: pgvector/pgvector:pg15
        env:
        - name: POSTGRES_USER
          value: "gate22"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: gate22-secrets
              key: postgres-password
        - name: POSTGRES_DB
          value: "gate22"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: gate22
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

### Step 4: Application Deployments

```yaml
# control-plane.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: control-plane
  namespace: gate22
spec:
  replicas: 3
  selector:
    matchLabels:
      app: control-plane
  template:
    metadata:
      labels:
        app: control-plane
    spec:
      containers:
      - name: control-plane
        image: your-registry/gate22-control-plane:latest
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: POSTGRES_HOST
          value: "postgres"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: gate22-secrets
              key: postgres-password
        # ... other env vars
        ports:
        - containerPort: 8000
        livenessProbe:
          httpGet:
            path: /v1/control-plane/health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /v1/control-plane/health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: control-plane
  namespace: gate22
spec:
  selector:
    app: control-plane
  ports:
  - port: 8000
    targetPort: 8000
```

### Step 5: Ingress Configuration

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gate22-ingress
  namespace: gate22
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.your-domain.com
    secretName: gate22-tls
  rules:
  - host: api.your-domain.com
    http:
      paths:
      - path: /v1/control-plane
        pathType: Prefix
        backend:
          service:
            name: control-plane
            port:
              number: 8000
      - path: /v1/mcp
        pathType: Prefix
        backend:
          service:
            name: mcp
            port:
              number: 8001
      - path: /v1/virtual
        pathType: Prefix
        backend:
          service:
            name: virtual-mcp
            port:
              number: 8002
```

### Step 6: Deploy to Kubernetes

```bash
# Apply configurations
kubectl apply -f namespace.yaml
kubectl apply -f secrets.yaml
kubectl apply -f postgres.yaml
kubectl apply -f control-plane.yaml
kubectl apply -f mcp.yaml
kubectl apply -f virtual-mcp.yaml
kubectl apply -f ingress.yaml

# Check status
kubectl get pods -n gate22
kubectl get services -n gate22
kubectl get ingress -n gate22

# View logs
kubectl logs -f deployment/control-plane -n gate22
```

## Frontend Deployment

### Vercel Deployment

1. Connect repository to Vercel
2. Configure environment variables:
   ```
   NEXT_PUBLIC_API_URL=https://api.your-domain.com
   NEXT_PUBLIC_APP_URL=https://app.your-domain.com
   NEXT_PUBLIC_ENVIRONMENT=production
   ```
3. Deploy automatically on push

### Docker Deployment

```dockerfile
# Dockerfile.frontend
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app

COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules

EXPOSE 3000
CMD ["npm", "start"]
```

## Database Backup & Recovery

### Automated Backups

```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"
POSTGRES_CONTAINER="gate22_postgres_1"

# Create backup
docker exec $POSTGRES_CONTAINER pg_dump -U gate22 gate22 > \
  $BACKUP_DIR/gate22_$DATE.sql

# Keep only last 30 days
find $BACKUP_DIR -name "gate22_*.sql" -mtime +30 -delete

# Upload to S3 (optional)
aws s3 cp $BACKUP_DIR/gate22_$DATE.sql \
  s3://your-backup-bucket/postgres/
```

### Restore from Backup

```bash
# Restore database
docker exec -i gate22_postgres_1 psql -U gate22 gate22 < \
  /backups/gate22_20240101_120000.sql
```

## Monitoring & Observability

### Health Checks

All services expose health check endpoints:

```bash
# Control Plane
curl https://api.your-domain.com/v1/control-plane/health

# MCP Service
curl https://api.your-domain.com/v1/mcp/health

# Virtual MCP Service
curl https://api.your-domain.com/v1/virtual/health
```

### Prometheus Metrics (Planned)

```yaml
# prometheus.yaml
scrape_configs:
  - job_name: 'gate22-control-plane'
    static_configs:
      - targets: ['control-plane:8000']
    metrics_path: '/v1/control-plane/metrics'
```

### Logging

Gate22 uses structured JSON logging in production:

```json
{
  "timestamp": "2024-01-01T00:00:00Z",
  "level": "INFO",
  "message": "User authenticated",
  "user_id": "123",
  "request_id": "abc-def-123"
}
```

Configure log aggregation with:
- **ELK Stack** (Elasticsearch, Logstash, Kibana)
- **Grafana Loki**
- **Cloud provider logs** (CloudWatch, Cloud Logging)

## Security Considerations

### SSL/TLS

Always use TLS in production:
- Use Let's Encrypt for free certificates
- Configure cert-manager for automatic renewal
- Enforce HTTPS redirects

### Secrets Management

Never store secrets in code:
- Use Kubernetes Secrets
- AWS Secrets Manager
- HashiCorp Vault
- Environment variables

### Network Security

- Use private networks for internal communication
- Configure firewalls to restrict access
- Implement rate limiting
- Enable DDoS protection

### Database Security

- Use strong passwords
- Enable SSL for database connections
- Restrict network access
- Regular security updates

## Scaling

### Horizontal Scaling

Scale services independently:

```bash
# Kubernetes
kubectl scale deployment control-plane --replicas=5 -n gate22

# Docker Compose
docker compose -f compose.production.yml up -d --scale control_plane=5
```

### Database Scaling

For high load:
- Use read replicas for read-heavy workloads
- Implement connection pooling (PgBouncer)
- Consider managed database services (RDS, Cloud SQL)

### Caching

Implement caching for improved performance:
- Redis for session storage
- CDN for static assets
- Application-level caching

## Troubleshooting

### Common Issues

**Database Connection Failed**
```bash
# Check database is running
kubectl get pods -n gate22 | grep postgres

# Check connection
kubectl exec -it <pod-name> -n gate22 -- psql -U gate22 -d gate22
```

**High Memory Usage**
```bash
# Check resource usage
kubectl top pods -n gate22

# Adjust resource limits in deployment
```

**Service Not Responding**
```bash
# Check logs
kubectl logs -f deployment/control-plane -n gate22

# Check service status
kubectl describe service control-plane -n gate22
```

## Maintenance

### Updates

```bash
# Pull latest images
docker compose -f compose.production.yml pull

# Recreate services
docker compose -f compose.production.yml up -d

# Run migrations
docker compose -f compose.production.yml exec migration alembic upgrade head
```

### Database Migrations

```bash
# Check current version
kubectl exec deployment/control-plane -n gate22 -- alembic current

# Upgrade
kubectl exec deployment/control-plane -n gate22 -- alembic upgrade head

# Rollback
kubectl exec deployment/control-plane -n gate22 -- alembic downgrade -1
```

## Next Steps

- [Architecture Overview](architecture/overview.md)
- [API Reference](api-reference.md)
- [Security Best Practices](architecture/authentication.md)
