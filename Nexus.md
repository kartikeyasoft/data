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

# Advanced Nexus Container Setup Script

set -e

# Default values
CONTAINER_NAME="nexus3"
IMAGE_NAME="sonatype/nexus3"
NEXUS_VERSION="latest"
HOST_PORT="8081"
CONTAINER_PORT="8081"
NEXUS_DATA_DIR="/opt/nexus-data"
MEMORY_LIMIT="2g"
CPU_LIMIT="2"

# Parse command line arguments
usage() {
    echo "Usage: $0 [OPTIONS]"
    echo "Options:"
    echo "  -n, --name NAME        Container name (default: nexus3)"
    echo "  -p, --port PORT        Host port (default: 8081)"
    echo "  -v, --version VERSION  Nexus version (default: latest)"
    echo "  -d, --data-dir DIR     Data directory (default: /opt/nexus-data)"
    echo "  -m, --memory LIMIT     Memory limit (default: 2g)"
    echo "  -c, --cpus LIMIT       CPU limit (default: 2)"
    echo "  -h, --help             Show this help message"
    exit 0
}

while [[ $# -gt 0 ]]; do
    case $1 in
        -n|--name)
            CONTAINER_NAME="$2"
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
        -d|--data-dir)
            NEXUS_DATA_DIR="$2"
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
            echo "Unknown option: $1"
            usage
            ;;
    esac
done


# Create data directory with proper permissions
setup_data_directory() {
    echo "Setting up data directory: $NEXUS_DATA_DIR"
    if [ ! -d "$NEXUS_DATA_DIR" ]; then
        mkdir -p "$NEXUS_DATA_DIR"
    fi
    chown -R 200:200 "$NEXUS_DATA_DIR"
    chmod -R 755 "$NEXUS_DATA_DIR"
}

# Remove existing container if running
cleanup() {
    if docker ps -a | grep -q "$CONTAINER_NAME"; then
        echo "Removing existing container: $CONTAINER_NAME"
        docker stop "$CONTAINER_NAME" 2>/dev/null || true
        docker rm "$CONTAINER_NAME" 2>/dev/null || true
    fi
}

# Pull image
pull_image() {
    echo "Pulling Nexus image: $IMAGE_NAME:$NEXUS_VERSION"
    docker pull "$IMAGE_NAME:$NEXUS_VERSION"
}

# Create container
create_container() {
    echo "Creating Nexus container..."

    docker run -d \
        --name "$CONTAINER_NAME" \
        --restart unless-stopped \
        -p "$HOST_PORT:$CONTAINER_PORT" \
        -v "$NEXUS_DATA_DIR:/nexus-data" \
        --memory="$MEMORY_LIMIT" \
        --cpus="$CPU_LIMIT" \
        -e INSTALL4J_ADD_VM_PARAMS="-Xms1200m -Xmx1200m -XX:MaxDirectMemorySize=2g" \
        "$IMAGE_NAME:$NEXUS_VERSION"

    echo "Container created successfully!"
}

# Display admin password
show_admin_password() {
    echo "Waiting for admin password to be generated..."
    sleep 30

    local password_file="$NEXUS_DATA_DIR/admin.password"
    if [ -f "$password_file" ]; then
        echo "Admin Password: $(cat $password_file)"
    else
        echo "Get admin password with: docker exec $CONTAINER_NAME cat /nexus-data/admin.password"
    fi
}

# Main
main() {
    echo "=== Nexus Container Setup ==="
    setup_data_directory
    cleanup
    pull_image
    create_container

    echo
    echo "Container is starting..."
    echo "Access Nexus at: http://localhost:$HOST_PORT"
    echo
    show_admin_password
    echo
    echo "Container Status:"
    docker ps --filter "name=$CONTAINER_NAME"
}

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

## Usage Examples

### Basic setup (default values)
```bash
./setup-nexus.sh
```

### Custom port and container name
```bash
./setup-nexus.sh --name my-nexus --port 9090
```

### With resource limits
```bash
./setup-nexus.sh --memory 4g --cpus 4
```

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

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Permission denied | Ensure `chown 200:200` ran successfully on data directory |
| Port already in use | Use `-p` option to specify a different port |
| Container not starting | Check logs: `docker logs nexus3` |
| High memory usage | Adjust `--memory` and JVM parameters |
| Slow startup | Nexus can take 2-3 minutes to fully initialize |

## Maintenance Commands

```bash
# Stop container
docker stop nexus3

# Start container
docker start nexus3

# View logs
docker logs -f nexus3

# Access container shell
docker exec -it nexus3 bash

# Backup data
tar -czf nexus-backup.tar.gz /opt/nexus-data
```