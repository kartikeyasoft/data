## Complete Working Setup Script

Save this as `setup-vault.sh`:

```bash
#!/bin/bash

echo "Setting up HashiCorp Vault..."

# Stop and remove existing container
docker stop vault 2>/dev/null
docker rm vault 2>/dev/null

# Run Vault container with proper capabilities
echo "Starting Vault container..."
docker run -d \
  --name vault \
  --cap-add=IPC_LOCK \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID="root" \
  -e VAULT_DEV_LISTEN_ADDRESS="0.0.0.0:8200" \
  hashicorp/vault:latest \
  server -dev

# Wait for Vault to be ready
echo "Waiting for Vault to initialize..."
sleep 10

# Check if Vault is healthy
if curl -s http://localhost:8200/v1/sys/health | grep -q '"sealed":false'; then
    echo "✅ Vault is running and unsealed"
else
    echo "❌ Vault failed to start properly"
    docker logs vault
    exit 1
fi

# Enable KV secrets engine
echo "Enabling KV secrets engine..."
docker exec -e VAULT_TOKEN="root" vault vault secrets enable -path=secret kv-v2 2>/dev/null || echo "KV engine already enabled"

# Store Nexus credentials
echo "Storing Nexus credentials..."
docker exec -e VAULT_TOKEN="root" vault vault kv put secret/nexus \
  username="admin" \
  password="admin123"

# Verify
echo "Verifying credentials..."
docker exec -e VAULT_TOKEN="root" vault vault kv get secret/nexus

```

Run it:
```bash
chmod +x setup-vault.sh
./setup-vault.sh
```


The issue is that Vault inside the container is trying to use HTTPS by default, but your dev server is running on HTTP. You need to set `VAULT_ADDR` to point to the HTTP endpoint.

Here's how to fix it:

## Solution 1: Set VAULT_ADDR when running the command

```bash
docker exec -it -e VAULT_TOKEN="root" -e VAULT_ADDR="http://127.0.0.1:8200" vault /bin/sh
```

Then inside the container:
```bash
vault kv put secret/nexus username="admin" password="admin123"
vault kv get secret/nexus
```
