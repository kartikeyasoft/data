## **setup-mysql.sh**

```bash
#!/bin/bash

# ============================================
# MySQL Container Setup Script for Voting App
# ============================================

set -e

echo "=========================================="
echo "MySQL Container Setup for Voting App"
echo "=========================================="

# Step 1: Install Docker
echo ""
echo "Step 1: Installing Docker..."
if ! command -v docker &> /dev/null; then
    sudo apt update
    sudo apt install docker.io -y
    sudo systemctl start docker
    sudo systemctl enable docker
    echo "✅ Docker installed successfully"
else
    echo "✅ Docker already installed"
fi

# Step 2: Remove existing container if exists
echo ""
echo "Step 2: Removing existing container if exists..."
if sudo docker ps -a --format '{{.Names}}' | grep -q "^mysql-db$"; then
    sudo docker stop mysql-db 2>/dev/null || true
    sudo docker rm mysql-db 2>/dev/null || true
    echo "✅ Removed existing mysql-db container"
fi

# Step 3: Run MySQL container
echo ""
echo "Step 3: Starting MySQL container..."
sudo docker run -d \
    --name mysql-db \
    --restart always \
    -p 0.0.0.0:3306:3306 \
    -e MYSQL_ROOT_PASSWORD=rootpassword \
    -e MYSQL_DATABASE=mydb \
    -e MYSQL_USER=appuser \
    -e MYSQL_PASSWORD=apppassword123 \
    -e MYSQL_ROOT_HOST='%' \
    mysql:8.0 \
    --bind-address=0.0.0.0 \
    --default-authentication-plugin=mysql_native_password

echo "✅ MySQL container started"

# Step 4: Wait for container to be ready
echo ""
echo "Step 4: Waiting for container to be ready..."
sleep 15

# Step 5: Check if running
echo ""
echo "Step 5: Checking container status..."
sudo docker ps | grep mysql-db

# Step 6: Check container logs
echo ""
echo "Step 6: Container logs (last 20 lines)..."
sudo docker logs mysql-db --tail 20

# Step 7: Get container IP
echo ""
echo "Step 7: Container IP address..."
sudo docker inspect mysql-db | grep IPAddress | head -1

# Step 8: Test local connection
echo ""
echo "Step 8: Testing local connection..."
if sudo docker exec mysql-db mysql -uappuser -papppassword123 -e "SHOW DATABASES;" 2>/dev/null; then
    echo "✅ Local connection successful"
else
    echo "❌ Local connection failed"
    exit 1
fi

# Step 9: Configure firewall (Ubuntu)
echo ""
echo "Step 9: Configuring firewall..."
if command -v ufw &> /dev/null; then
    sudo ufw allow 3306/tcp 2>/dev/null || true
    echo "✅ Firewall rule added (ufw)"
elif command -v firewall-cmd &> /dev/null; then
    sudo firewall-cmd --permanent --add-port=3306/tcp 2>/dev/null || true
    sudo firewall-cmd --reload 2>/dev/null || true
    echo "✅ Firewall rule added (firewalld)"
else
    echo "⚠️ No firewall detected - skipping"
fi

# Step 10: Grant remote access
echo ""
echo "Step 10: Setting up remote access..."
sudo docker exec mysql-db mysql -uroot -prootpassword -e "GRANT ALL PRIVILEGES ON *.* TO 'appuser'@'%' IDENTIFIED BY 'apppassword123' WITH GRANT OPTION;" 2>/dev/null || true
sudo docker exec mysql-db mysql -uroot -prootpassword -e "FLUSH PRIVILEGES;" 2>/dev/null || true
echo "✅ Remote access configured"

# Step 11: Display connection information
echo ""
echo "=========================================="
echo "✅ MySQL Setup Complete!"
echo "=========================================="
echo ""
echo "📊 Connection Information:"
echo "   Host: localhost"
echo "   Port: 3306"
echo ""
echo "🔑 Database Credentials:"
echo "   Database: mydb"
echo "   Username: appuser"
echo "   Password: apppassword123"
echo "   Root Password: rootpassword"
echo ""
echo "📝 JDBC URL:"
echo "   jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true"
echo ""
echo "🧪 Test Commands:"
echo ""
echo "   # Test from localhost:"
echo "   mysql -h 127.0.0.1 -u appuser -papppassword123 -e \"SHOW DATABASES;\""
echo ""
echo "   # Connect to MySQL inside container:"
echo "   sudo docker exec -it mysql-db mysql -uappuser -papppassword123 mydb"
echo ""
echo "   # View container logs:"
echo "   sudo docker logs mysql-db"
echo ""
echo "   # Stop container:"
echo "   sudo docker stop mysql-db"
echo ""
echo "   # Start container:"
echo "   sudo docker start mysql-db"
echo ""
echo "=========================================="
```

## **How to use:**

### 1. **Save and make executable:**
```bash
cd ~/Vote\ App
chmod +x setup-mysql.sh
```

### 2. **Run the script:**
```bash
./setup-mysql.sh
```

### 3. **Your application.properties will work as-is** with these credentials:
- Database: `mydb`
- Username: `appuser`
- Password: `apppassword123`
- Host: `mysql-db` (from Docker) or `localhost` (from your host)

### 4. **Run your Spring Boot app:**
```bash
mvn spring-boot:run
```

The script automatically creates the MySQL container with exactly the credentials your application expects!
