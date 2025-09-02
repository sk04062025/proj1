# Offline Logstash JDBC Plugin Installation Guide

Since you cannot access rubygems.org directly from your server, here's how to set up the JDBC plugin offline.

## Step 1: Manual Downloads Required

You need to download these files manually from a machine with internet access:

### Required Files to Download:

1. **PostgreSQL JDBC Driver**
   - URL: https://jdbc.postgresql.org/download/postgresql-42.6.0.jar
   - Save as: `postgresql-42.6.0.jar`

2. **Logstash JDBC Input Plugin Gem**
   - URL: https://rubygems.org/downloads/logstash-input-jdbc-5.4.6.gem
   - Save as: `logstash-input-jdbc-5.4.6.gem`

3. **Plugin Dependencies** (download these gems as well):
   - https://rubygems.org/downloads/sequel-5.72.0.gem
   - https://rubygems.org/downloads/tzinfo-data-1.2023.3.gem
   - https://rubygems.org/downloads/rufus-scheduler-3.0.9.gem
   - https://rubygems.org/downloads/chronic_duration-0.10.6.gem

## Step 2: Directory Structure Setup

Create this directory structure on your server:

```
elk-stack/
├── docker-compose.yml
├── logstash/
│   ├── Dockerfile
│   ├── plugins/
│   │   ├── logstash-input-jdbc-5.4.6.gem
│   │   ├── sequel-5.72.0.gem
│   │   ├── tzinfo-data-1.2023.3.gem
│   │   ├── rufus-scheduler-3.0.9.gem
│   │   └── chronic_duration-0.10.6.gem
│   ├── drivers/
│   │   └── postgresql-42.6.0.jar
│   ├── config/
│   │   └── logstash.yml
│   └── pipeline/
│       └── logstash.conf
├── kibana/config/kibana.yml
└── filebeat/config/filebeat.yml
```

## Step 3: Alternative Approaches

### Option A: Custom Docker Image (Recommended)

Use the Dockerfile I provided above. This approach:
- Pre-installs all plugins during image build
- No internet required at runtime
- Most reliable for air-gapped environments

**Commands:**
```bash
# Build the custom image
docker-compose build logstash

# Start the services
docker-compose up -d
```

### Option B: Pre-built Image with Volume Mount

Create a pre-built image and push to your private registry:

```dockerfile
# Build this on a machine with internet access
FROM docker.elastic.co/logstash/logstash:8.11.0

USER root
RUN /usr/share/logstash/bin/logstash-plugin install logstash-input-jdbc
USER logstash

# Tag and push to your private registry
# docker build -t your-registry/logstash-with-jdbc:8.11.0 .
# docker push your-registry/logstash-with-jdbc:8.11.0
```

Then update docker-compose.yml:
```yaml
logstash:
  image: your-registry/logstash-with-jdbc:8.11.0
  # ... rest of configuration
```

### Option C: Manual Plugin Installation in Running Container

If you prefer to install manually in a running container:

```bash
# Start basic logstash first
docker-compose up -d logstash

# Copy plugin files to container
docker cp logstash-input-jdbc-5.4.6.gem logstash:/tmp/
docker cp sequel-5.72.0.gem logstash:/tmp/
docker cp postgresql-42.6.0.jar logstash:/usr/share/logstash/drivers/

# Install plugins manually
docker exec -u root logstash /usr/share/logstash/bin/logstash-plugin install file:///tmp/logstash-input-jdbc-5.4.6.gem
docker exec -u root logstash /usr/share/logstash/bin/logstash-plugin install file:///tmp/sequel-5.72.0.gem

# Restart logstash
docker-compose restart logstash
```

## Step 4: Download Script for Internet-Connected Machine

If you have access to another machine with internet, use this script:

```bash
#!/bin/bash
# download_logstash_plugins.sh

# Create directories
mkdir -p logstash-offline/{plugins,drivers}
cd logstash-offline

# Download JDBC driver
echo "Downloading PostgreSQL JDBC driver..."
curl -L -o drivers/postgresql-42.6.0.jar https://jdbc.postgresql.org/download/postgresql-42.6.0.jar

# Download plugin gems
echo "Downloading Logstash JDBC plugin and dependencies..."
cd plugins

# Main plugin
curl -L -o logstash-input-jdbc-5.4.6.gem https://rubygems.org/downloads/logstash-input-jdbc-5.4.6.gem

# Dependencies
curl -L -o sequel-5.72.0.gem https://rubygems.org/downloads/sequel-5.72.0.gem
curl -L -o tzinfo-data-1.2023.3.gem https://rubygems.org/downloads/tzinfo-data-1.2023.3.gem
curl -L -o rufus-scheduler-3.0.9.gem https://rubygems.org/downloads/rufus-scheduler-3.0.9.gem
curl -L -o chronic_duration-0.10.6.gem https://rubygems.org/downloads/chronic_duration-0.10.6.gem

cd ..
echo "All files downloaded successfully!"
echo "Transfer the 'logstash-offline' folder to your server."

# Create a tar for easy transfer
tar -czf logstash-offline.tar.gz logstash-offline/
echo "Created logstash-offline.tar.gz for transfer"
```

## Step 5: Installation Commands

Once you have all files on your server:

```bash
# Extract if you used the tar approach
tar -xzf logstash-offline.tar.gz

# Copy files to correct locations
cp logstash-offline/drivers/* logstash/drivers/
cp logstash-offline/plugins/* logstash/plugins/

# Build and start
docker-compose build logstash
docker-compose up -d
```

## Step 6: Verification

Check if the plugin is installed correctly:

```bash
# List installed plugins
docker exec logstash /usr/share/logstash/bin/logstash-plugin list

# Should show: logstash-input-jdbc

# Check JDBC driver
docker exec logstash ls -la /usr/share/logstash/drivers/

# Test JDBC connection (optional)
docker exec logstash /usr/share/logstash/bin/logstash -e '
input {
  jdbc {
    jdbc_driver_library => "/usr/share/logstash/drivers/postgresql-42.6.0.jar"
    jdbc_driver_class => "org.postgresql.Driver"
    jdbc_connection_string => "jdbc:postgresql://postgres:5432/testdb"
    jdbc_user => "logstash_user"
    jdbc_password => "logstash_pass"
    statement => "SELECT 1 as test"
  }
}
output { stdout { codec => rubydebug } }
' --config.test_and_exit
```

## Troubleshooting

**If build fails:**
```bash
# Clean and rebuild
docker-compose down
docker rmi $(docker images -f "dangling=true" -q) 2>/dev/null || true
docker-compose build --no-cache logstash
docker-compose up -d
```

**If plugin installation fails:**
- Ensure all gem files are in the correct directory
- Check file permissions: `chmod 644 logstash/plugins/*.gem`
- Verify gem file integrity by checking file sizes

This approach should work completely offline once you have the required files transferred to your server.