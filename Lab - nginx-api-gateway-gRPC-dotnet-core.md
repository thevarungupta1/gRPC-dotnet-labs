# NGINX API Gateway for gRPC and .NET Core using NGINX Docker Container

## 1. Setup Your .NET gRPC Service
First, create a **.NET gRPC service** if you don't have one:

```bash
dotnet new grpc -o GrpcService
cd GrpcService
dotnet run
```

By default, gRPC services in .NET listen on **port 5001 (HTTPS)**.

---

## 2. Create an NGINX Configuration for gRPC
Create a file named `nginx.conf` with the following content:

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    upstream grpc_backend {
        server grpc_service:5001;  # Change this to your .NET gRPC service address
    }

    server {
        listen 80 http2;

        location / {
            grpc_pass grpc://grpc_backend;
            error_page 502 = /error502grpc;
        }

        location = /error502grpc {
            internal;
            default_type application/grpc;
            add_header grpc-status 14;
            add_header content-length 0;
            return 204;
        }
    }
}
```

This config:
- Defines an upstream called `grpc_backend` pointing to the gRPC service.
- Uses `http2` to handle gRPC requests.
- Forwards gRPC requests to the backend via `grpc_pass grpc://grpc_backend;`.

---

## 3. Create a Dockerfile for NGINX
Create a `Dockerfile` in the same directory:

```dockerfile
FROM nginx:latest

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 4. Create a Docker Compose File
Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  grpc_service:
    image: my-grpc-service:latest  # Build your .NET gRPC service and replace this
    ports:
      - "5001:5001"

  nginx:
    build: .
    ports:
      - "8081:80"
    depends_on:
      - grpc_service

```

---

## 5. Build and Run with Docker

### Step 1: Build Your gRPC Service Docker Image
Inside your **gRPC .NET project**, create a `Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
ENV ASPNETCORE_HTTP_PORTS=5001
ENV ASPNETCORE_HTTP_PORT=5000
COPY bin/Release/net9.0/publish/ .
ENTRYPOINT ["dotnet", "GrpcService.dll"]
```

Then build and tag your image:

```bash
dotnet publish -c Release
docker build -t my-grpc-service .
```

### Step 2: Build and Start NGINX
Run:

```bash
docker-compose up --build
```

Your **NGINX API Gateway** for gRPC will be available at `http://localhost:8081/`.

---

