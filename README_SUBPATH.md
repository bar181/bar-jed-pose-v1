# Subpath Hosting Guide

This application supports hosting at any subpath, configured entirely at runtime. This is perfect for Kubernetes deployments where the same container image needs to run at different paths across environments.

## Runtime Configuration

The application uses a **single environment variable** configured at runtime:

- `BASE_PATH` - The URL path where the app will be served (default: `/`)

## Key Features

✅ **Single container image** - Build once, deploy anywhere  
✅ **Runtime configuration** - No rebuild needed for different paths  
✅ **Kubernetes-friendly** - Perfect for multi-environment deployments  
✅ **Dynamic asset loading** - All resources load correctly regardless of path  

## Quick Start

### Build Once
```bash
docker build -t gait-detection .
```

### Deploy Anywhere

**Root path (default):**
```bash
docker run -p 8080:80 gait-detection
```
Access at: `http://localhost:8080/`

**Subpath deployment:**
```bash
docker run -p 8080:80 -e BASE_PATH=/gait-detection gait-detection
```
Access at: `http://localhost:8080/gait-detection`

**Nested subpath:**
```bash
docker run -p 8080:80 -e BASE_PATH=/apps/pose-detector gait-detection
```
Access at: `http://localhost:8080/apps/pose-detector`

## Docker Compose Example

See `docker-compose.example.yml` for multiple deployment scenarios using the same image.

## Kubernetes Deployment

Deploy to Kubernetes with any path - see `k8s-example.yaml` for complete examples:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gait-detection
spec:
  template:
    spec:
      containers:
      - name: gait-detection
        image: your-registry/gait-detection:latest
        env:
        - name: BASE_PATH
          value: "/gait-detection"  # Configure your path here
```

### ConfigMap for Multiple Environments

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gait-detection-config
data:
  # Different paths for different environments
  path.dev: "/dev/pose"
  path.staging: "/staging/pose"
  path.prod: "/pose-detection"
---
# Use in deployment
env:
- name: BASE_PATH
  valueFrom:
    configMapKeyRef:
      name: gait-detection-config
      key: path.prod
```

## Integration with Reverse Proxy

When using a reverse proxy (like nginx, Apache, or Traefik), ensure the proxy path matches the configured subpath:

### Nginx Example
```nginx
location /gait-detection {
    proxy_pass http://gait-detection-container:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

### Kubernetes Ingress Example
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gait-detection-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /gait-detection
        pathType: Prefix
        backend:
          service:
            name: gait-detection-service
            port:
              number: 80
```

## Notes

1. The subpath must start with `/` and should not end with `/`
2. Both `PUBLIC_URL` (build-time) and `BASE_PATH` (runtime) must match
3. All assets and API calls will be relative to the configured subpath
4. The health check endpoint remains at `/health` relative to the container root