```markdown
# SonarQube Docker Setup

## Quick Start

Run the following command to start SonarQube LTS Community Edition:

```bash
docker run -d -p 9000:9000 sonarqube:lts-community
```

## Parameters

| Parameter | Description |
|-----------|-------------|
| `-d` | Run container in detached mode (background) |
| `-p 9000:9000` | Map host port 9000 to container port 9000 |
| `sonarqube:lts-community` | SonarQube LTS Community Edition image |

## Access

Once running, access SonarQube at: `http://localhost:9000`

## Default Credentials

- **Username:** `admin`
- **Password:** `admin`
```