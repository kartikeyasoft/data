# MariaDB Container Setup Script

This Bash script automates the deployment of a MariaDB container with Docker, including installation, configuration, firewall setup, and remote access.

## Prerequisites

- Linux system (CentOS/RHEL based with `yum` or Ubuntu/Debian with `ufw`)
- Root or sudo access

## Script Overview

The script performs the following steps:

1. Installs Docker (if not present)
2. Removes any existing `exam-db` container
3. Starts a new MariaDB 10.11 container
4. Waits for the container to be ready
5. Verifies container status and logs
6. Retrieves container IP address
7. Tests local database connection
8. Configures firewall (firewalld or ufw)
9. Grants remote access privileges
10. Displays connection information

## Full Script

```markdown

#!/bin/bash

# ============================================
# MariaDB Container Setup Script
# ============================================

set -e

echo "=========================================="
echo "MariaDB Container Setup"
echo "=========================================="

# Step 1: Install Docker
echo ""
echo "Step 1: Installing Docker..."
if ! command -v docker &> /dev/null; then
    sudo yum install docker -y
    sudo systemctl start docker
    sudo systemctl enable docker
    echo "✅ Docker installed successfully"
else
    echo "✅ Docker already installed"
fi

# Step 2: Remove existing container if exists
echo ""
echo "Step 2: Removing existing container if exists..."
if sudo docker ps -a --format '{{.Names}}' | grep -q "^exam-db$"; then
    sudo docker stop exam-db 2>/dev/null || true
    sudo docker rm exam-db 2>/dev/null || true
    echo "✅ Removed existing exam-db container"
fi

# Step 3: Run MariaDB container
echo ""
echo "Step 3: Starting MariaDB container..."
sudo docker run -d \
    --name exam-db \
    --restart always \
    -p 0.0.0.0:3306:3306 \
    -e MYSQL_ROOT_PASSWORD=Admin@123 \
    -e MYSQL_DATABASE=exam \
    -e MYSQL_USER=Admin \
    -e MYSQL_PASSWORD=Admin@123 \
    -e MYSQL_ROOT_HOST='%' \
    mariadb:10.11 \
    --bind-address=0.0.0.0

echo "✅ MariaDB container started"

# Step 4: Wait for container to be ready
echo ""
echo "Step 4: Waiting for container to be ready..."
sleep 10

# Step 5: Check if running
echo ""
echo "Step 5: Checking container status..."
sudo docker ps | grep exam-db

# Step 6: Check container logs
echo ""
echo "Step 6: Container logs (last 20 lines)..."
sudo docker logs exam-db --tail 20

# Step 7: Get container IP
echo ""
echo "Step 7: Container IP address..."
sudo docker inspect exam-db | grep IPAddress | head -1

# Step 8: Test local connection
echo ""
echo "Step 8: Testing local connection..."
if sudo docker exec exam-db mysql -uAdmin -pAdmin@123 -e "SHOW DATABASES;" 2>/dev/null; then
    echo "✅ Local connection successful"
else
    echo "❌ Local connection failed"
    exit 1
fi

# Step 9: Configure firewall
echo ""
echo "Step 9: Configuring firewall..."
if command -v firewall-cmd &> /dev/null; then
    sudo firewall-cmd --permanent --add-port=3306/tcp 2>/dev/null || true
    sudo firewall-cmd --reload 2>/dev/null || true
    echo "✅ Firewall rule added (firewalld)"
elif command -v ufw &> /dev/null; then
    sudo ufw allow 3306/tcp 2>/dev/null || true
    echo "✅ Firewall rule added (ufw)"
else
    echo "⚠️ No firewall detected - skipping"
fi

# Step 10: Grant remote access
echo ""
echo "Step 10: Setting up remote access..."
sudo docker exec exam-db mysql -uAdmin -pAdmin@123 -e "GRANT ALL PRIVILEGES ON *.* TO 'Admin'@'%' IDENTIFIED BY 'Admin@123' WITH GRANT OPTION;" 2>/dev/null || true
sudo docker exec exam-db mysql -uAdmin -pAdmin@123 -e "FLUSH PRIVILEGES;" 2>/dev/null || true
echo "✅ Remote access configured"

# Step 11: Display connection information
echo ""
echo "=========================================="
echo "✅ MariaDB Setup Complete!"
echo "=========================================="
echo ""
echo "📊 Connection Information:"
echo "   Host: $(curl -s ifconfig.me 2>/dev/null || hostname -I | awk '{print $1}')"
echo "   Port: 3306"
echo ""
echo "🔑 Database Credentials:"
echo "   Database: exam"
echo "   Username: Admin"
echo "   Password: Admin@123"
echo "   Root Password: Admin@123"
echo ""
echo "🧪 Test Commands:"
echo ""
echo "   # Test from localhost:"
echo "   mysql -h 127.0.0.1 -u Admin -pAdmin@123 -e \"SHOW DATABASES;\""
echo ""
echo "   # Connect to MySQL inside container:"
echo "   sudo docker exec -it exam-db mysql -uAdmin -pAdmin@123"
echo ""
echo "=========================================="
```

## Usage

1. Save the script as `setup-mariadb.sh`
2. Make it executable:
   ```bash
   chmod +x setup-mariadb.sh
   ```
3. Run the script:
   ```bash
   ./setup-mariadb.sh
   ```

## Container Configuration

| Parameter | Value |
|-----------|-------|
| Container Name | `exam-db` |
| Image | `mariadb:10.11` |
| Host Port | `3306` |
| Container Port | `3306` |
| Restart Policy | `always` |

## Database Credentials

| Variable | Value |
|----------|-------|
| Root Password | `Admin@123` |
| Database Name | `exam` |
| Username | `Admin` |
| User Password | `Admin@123` |

## Test Commands

After setup, verify the installation with:

```bash
# Test from localhost
mysql -h 127.0.0.1 -u Admin -pAdmin@123 -e "SHOW DATABASES;"

# Connect to MySQL inside container
sudo docker exec -it exam-db mysql -uAdmin -pAdmin@123
```

## Security Notes

> ⚠️ **Warning:** The script uses default credentials (`Admin@123`). For production environments, change these passwords immediately after setup.

## Troubleshooting

- **Connection refused:** Check if firewall is blocking port 3306
- **Container not starting:** Run `sudo docker logs exam-db` for detailed errors
- **Remote access issues:** Verify `MYSQL_ROOT_HOST='%'` is set correctly
