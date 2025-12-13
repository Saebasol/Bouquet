# Bouquet 💐

Bouquet is a easy deployment solution for Heliotrope and Hibiscus.

## 🏗️ Architecture

This project consists of the following services:

- **Heliotrope**: Main API application
- **Sunflower**: Hitomi.la mirror service
- **Hibiscus**: Frontend built on top of Heliotrope
- **Traefik**: Reverse proxy and load balancer
- **PostgreSQL**: Relational database for galleryinfo storage
- **MongoDB**: NoSQL database for info storage

## 🚀 Quick Start

### Prerequisites

- Docker and Docker Compose installed
- Cloudflare account and API token

### Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/Saebasol/Bouquet
   cd Bouquet
   ```

2. **Configure environment variables**
   
Replace the following values in the `docker-compose.yaml` file with your actual values:
   
```yaml
# Cloudflare configuration
CLOUDFLARE_EMAIL: <your-cloudflare-email>
CLOUDFLARE_DNS_API_TOKEN: <your-cloudflare-api-token>

# Domain configuration
Host(`example.com`) → Host(`your-domain.com`)
Host(`www.example.com`) → Host(`www.your-domain.com`)

# Database credentials (recommended to change for security)
POSTGRES_USER: your-postgres-user
POSTGRES_PASSWORD: your-secure-password
MONGO_INITDB_ROOT_USERNAME: your-mongo-user
MONGO_INITDB_ROOT_PASSWORD: your-secure-password

# Heliotrope configuration
SANIC_GALLEYINFO_DB_URL: postgres://your-postgres-user:your-secure-password@postgres:5432/heliotrope
SANIC_INFO_DB_URL: mongodb://your-mongo-user:your-secure-password@mongodb:2701

```

3. **Start services**
   ```bash
   docker-compose up -d
   ```

## 🔧 Advanced Configuration

### Dynamic Routing Setup

To use Traefik's dynamic configuration:

1. Create a `dynamic.yaml` file
2. Uncomment the corresponding line in `docker-compose.yaml`:
```yaml
volumes:
    - ./dynamic.yaml:/etc/traefik/dynamic/dynamic.yml:ro

command:
    - --providers.file.directory=/etc/traefik/dynamic
    - --providers.file.watch=true
```

### Support MongoDB Atlas Search

To enable MongoDB Atlas Search, you need to ensure that your MongoDB instance is configured to support it. This typically involves using a MongoDB Atlas cluster with the search feature enabled.

Then add the following environment variable to the `heliotrope` service in `docker-compose.yaml`:

```yaml
environment:
    # https://github.com/Saebasol/Heliotrope/blob/main/heliotrope/infrastructure/sanic/config.py
    SANIC_USE_ATLAS_SEARCH: "true"
```