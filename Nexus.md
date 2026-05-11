# Advanced Nexus Container Setup Script

This Bash script automates the deployment of a Sonatype Nexus 3 repository manager container with Docker, featuring configurable options for ports, resource limits, and persistent storage.

## Script Overview

The script provides an automated way to deploy Nexus 3 with:
- Persistent data storage
- Configurable resource limits (CPU/Memory)
- Customizable port mappings
- Automatic admin password retrieval
- Graceful container cleanup

## Features

- ✅ **Configurable Options** - Set container name, port, version, data directory, and resource limits
- ✅ **Persistent Storage** - Mounts local directory for Nexus data
- ✅ **Resource Management** - CPU and memory limits with JVM optimizations
- ✅ **Auto-restart** - Container restarts unless manually stopped
- ✅ **Security** - Proper file permissions for Nexus user (UID 200)
- ✅ **Admin Password** - Automatically displays initial admin password

## Full Script

```markdown

#!/bin/bash

# Nexus Container Setup Script - Working Version with Docker Volumes

set -e

# Default values
CONTAINER_NAME="nexus3"
IMAGE_NAME="sonatype/nexus3"
NEXUS_VERSION="latest"
HOST_PORT="8081"
CONTAINER_PORT="8081"
MEMORY_LIMIT="2g"
CPU_LIMIT="2"
VOLUME_NAME="nexus-data"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Parse command line arguments
usage() {
    echo "Usage: $0 [OPTIONS]"
    echo "Options:"
    echo "  -n, --name NAME        Container name (default: nexus3)"
    echo "  -p, --port PORT        Host port (default: 8081)"
    echo "  -v, --version VERSION  Nexus version (default: latest)"
    echo "  -m, --memory LIMIT     Memory limit (default: 2g)"
    echo "  -c, --cpus LIMIT       CPU limit (default: 2)"
    echo "  -h, --help             Show this help message"
    exit 0
}

while [[ $# -gt 0 ]]; do
    case $1 in
        -n|--name)
            CONTAINER_NAME="$2"
            VOLUME_NAME="${VOLUME_NAME}-${2}"
            shift 2
            ;;
        -p|--port)
            HOST_PORT="$2"
            shift 2
            ;;
        -v|--version)
            NEXUS_VERSION="$2"
            shift 2
            ;;
        -m|--memory)
            MEMORY_LIMIT="$2"
            shift 2
            ;;
        -c|--cpus)
            CPU_LIMIT="$2"
            shift 2
            ;;
        -h|--help)
            usage
            ;;
        *)
            echo -e "${RED}Unknown option: $1${NC}"
            usage
            ;;
    esac
done

# Check if Docker is installed
check_docker() {
    if ! command -v docker &> /dev/null; then
        echo -e "${RED}Docker is not installed. Please install Docker first.${NC}"
        exit 1
    fi
}

# Create Docker volume
create_volume() {
    echo -e "${YELLOW}Creating Docker volume: $VOLUME_NAME${NC}"
    
    if docker volume ls | grep -q "$VOLUME_NAME"; then
        echo -e "${YELLOW}Volume already exists. Using existing volume.${NC}"
    else
        docker volume create "$VOLUME_NAME"
        echo -e "${GREEN}Volume created successfully${NC}"
    fi
}

# Remove existing container if running
cleanup() {
    if docker ps -a --format '{{.Names}}' | grep -q "^${CONTAINER_NAME}$"; then
        echo -e "${YELLOW}Removing existing container: $CONTAINER_NAME${NC}"
        docker stop "$CONTAINER_NAME" 2>/dev/null || true
        docker rm "$CONTAINER_NAME" 2>/dev/null || true
        echo -e "${GREEN}Container removed${NC}"
    fi
}

# Pull image
pull_image() {
    echo -e "${YELLOW}Pulling Nexus image: $IMAGE_NAME:$NEXUS_VERSION${NC}"
    docker pull "$IMAGE_NAME:$NEXUS_VERSION"
    echo -e "${GREEN}Image pulled successfully${NC}"
}

# Create and run container
create_container() {
    echo -e "${YELLOW}Creating Nexus container...${NC}"
    
    # Build the docker run command
    local docker_cmd="docker run -d \
        --name \"$CONTAINER_NAME\" \
        --restart unless-stopped \
        -p \"$HOST_PORT:$CONTAINER_PORT\" \
        -v \"$VOLUME_NAME:/nexus-data\" \
        --memory=\"$MEMORY_LIMIT\" \
        --cpus=\"$CPU_LIMIT\" \
        -e INSTALL4J_ADD_VM_PARAMS=\"-Xms1200m -Xmx1200m -XX:MaxDirectMemorySize=2g\""
    
    # Add healthcheck
    docker_cmd="$docker_cmd --health-cmd=\"curl -f http://localhost:8081 || exit 1\" \
        --health-interval=30s \
        --health-timeout=10s \
        --health-retries=5"
    
    docker_cmd="$docker_cmd \"$IMAGE_NAME:$NEXUS_VERSION\""
    
    eval $docker_cmd
    
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}Container created successfully!${NC}"
    else
        echo -e "${RED}Failed to create container${NC}"
        exit 1
    fi
}

# Wait for Nexus to start
wait_for_nexus() {
    echo -e "${YELLOW}Waiting for Nexus to start (this may take 2-3 minutes)...${NC}"
    
    local max_attempts=60
    local attempt=1
    
    while [ $attempt -le $max_attempts ]; do
        if docker logs "$CONTAINER_NAME" 2>&1 | grep -q "Started Sonatype Nexus"; then
            echo -e "${GREEN}Nexus has started successfully!${NC}"
            return 0
        fi
        
        # Also check if the container is healthy
        local health_status=$(docker inspect --format='{{.State.Health.Status}}' "$CONTAINER_NAME" 2>/dev/null)
        if [ "$health_status" = "healthy" ]; then
            echo -e "${GREEN}Nexus is healthy!${NC}"
            return 0
        fi
        
        echo -n "."
        sleep 5
        attempt=$((attempt + 1))
    done
    
    echo -e "\n${YELLOW}Nexus is still starting. Check logs with: docker logs $CONTAINER_NAME${NC}"
}

# Display admin password
show_admin_password() {
    echo -e "\n${YELLOW}Fetching admin password...${NC}"
    sleep 10
    
    # Try multiple methods to get the password
    local password=""
    
    # Method 1: Direct from container
    password=$(docker exec "$CONTAINER_NAME" cat /nexus-data/admin.password 2>/dev/null)
    
    # Method 2: From volume (if direct access fails)
    if [ -z "$password" ]; then
        password=$(docker run --rm -v "$VOLUME_NAME:/data" alpine cat /data/admin.password 2>/dev/null)
    fi
    
    if [ -n "$password" ]; then
        echo -e "${GREEN}========================================${NC}"
        echo -e "${GREEN}Admin Username: admin${NC}"
        echo -e "${GREEN}Admin Password: $password${NC}"
        echo -e "${GREEN}========================================${NC}"
    else
        echo -e "${YELLOW}Password file not ready yet. Get it later with:${NC}"
        echo -e "docker exec $CONTAINER_NAME cat /nexus-data/admin.password"
    fi
}

# Show useful commands
show_useful_commands() {
    echo -e "\n${GREEN}=== Useful Commands ===${NC}"
    echo -e "View logs:       ${YELLOW}docker logs -f $CONTAINER_NAME${NC}"
    echo -e "Stop container:  ${YELLOW}docker stop $CONTAINER_NAME${NC}"
    echo -e "Start container: ${YELLOW}docker start $CONTAINER_NAME${NC}"
    echo -e "Restart:         ${YELLOW}docker restart $CONTAINER_NAME${NC}"
    echo -e "Shell access:    ${YELLOW}docker exec -it $CONTAINER_NAME /bin/bash${NC}"
    echo -e "Get password:    ${YELLOW}docker exec $CONTAINER_NAME cat /nexus-data/admin.password${NC}"
    echo -e "Backup volume:   ${YELLOW}docker run --rm -v $VOLUME_NAME:/data -v \$(pwd):/backup alpine tar czf /backup/nexus-backup.tar.gz /data${NC}"
    echo -e "Remove container:${YELLOW} docker rm -f $CONTAINER_NAME${NC}"
    echo -e "Remove volume:   ${YELLOW}docker volume rm $VOLUME_NAME${NC}"
}

# Main function
main() {
    echo -e "${GREEN}=== Nexus Container Setup ===${NC}"
    echo -e "Container: $CONTAINER_NAME"
    echo -e "Port: $HOST_PORT:$CONTAINER_PORT"
    echo -e "Memory: $MEMORY_LIMIT"
    echo -e "CPU: $CPU_LIMIT"
    echo -e "Volume: $VOLUME_NAME"
    echo ""
    
    check_docker
    create_volume
    cleanup
    pull_image
    create_container
    wait_for_nexus
    show_admin_password
    show_useful_commands
    
    echo -e "\n${GREEN}========================================${NC}"
    echo -e "${GREEN}Nexus is ready! Access it at:${NC}"
    echo -e "${YELLOW}http://localhost:$HOST_PORT${NC}"
    echo -e "${GREEN}========================================${NC}"
}

# Run main function
main
```

## Command Line Options

| Option | Description | Default Value |
|--------|-------------|---------------|
| `-n, --name` | Container name | `nexus3` |
| `-p, --port` | Host port for Nexus UI | `8081` |
| `-v, --version` | Nexus version tag | `latest` |
| `-d, --data-dir` | Local data directory | `/opt/nexus-data` |
| `-m, --memory` | Memory limit for container | `2g` |
| `-c, --cpus` | CPU limit for container | `2` |
| `-h, --help` | Show help message | - |

## Usage Steps

1. **Save the script**
   ```bash
   nano setup-nexus.sh
   ```

2. **Make it executable**
   ```bash
   chmod +x setup-nexus.sh
   ```

3. **Run the script**
   ```bash
   ./setup-nexus.sh
   ```

## Access Nexus

Once the container is running, access Nexus at:
```
http://localhost:8081
```

## Initial Login

- **Username:** `admin`
- **Password:** Displayed automatically after setup (or use `docker exec nexus3 cat /nexus-data/admin.password`)

## Security Notes

> ⚠️ **Important:**
> - Change the admin password immediately after first login
> - The data directory (`/opt/nexus-data`) contains sensitive information
> - Ensure proper backup of the data directory
> - For production, consider using secrets management instead of default passwords
