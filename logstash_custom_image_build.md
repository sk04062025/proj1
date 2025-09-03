# Build Custom Logstash Image with JDBC Plugin

## Step 1: On Internet-Connected Server

Create these files on your internet-connected server:

### Dockerfile
```dockerfile
# Dockerfile
FROM docker.elastic.co/logstash/logstash:8.11.0

# Switch to root user to install plugins and make system changes
USER root

# Install the JDBC input plugin
RUN /usr/share/logstash/bin/logstash-plugin install logstash-input-jdbc

# Create directory for JDBC drivers
RUN mkdir -p /usr/share/logstash/drivers

# Download PostgreSQL JDBC driver directly into the image
RUN curl -L -o /usr/share/logstash/drivers/postgresql-42.6.0.jar \
    https://jdbc.postgresql.org/download/postgresql-42.6.0.jar

# Set proper permissions for the drivers directory
RUN chown -R logstash:logstash /usr/share/logstash/drivers
RUN chmod 644 /usr/share/logstash/drivers/*.jar

# Verify plugin installation
RUN /usr/share/logstash/bin/logstash-plugin list | grep jdbc

# Switch back to logstash user for security
USER logstash

# Add labels for better image management
LABEL maintainer="your-team@company.com"
LABEL version="8.11.0-jdbc"
LABEL description="Logstash 8.11.0 with JDBC input plugin and PostgreSQL driver"
```

### Build Script
```bash
#!/bin/bash
# build_logstash_image.sh

set -e  # Exit on any error

echo "Building custom Logstash image with JDBC plugin..."

# Build the Docker image
docker build -t logstash-with-jdbc:8.11.0 .

# Verify the build was successful
echo "Verifying plugin installation..."
docker run --rm logstash-with-jdbc:8.11.0 /usr/share/logstash/bin/logstash-plugin list | grep jdbc

# Check JDBC driver
echo "Verifying JDBC driver..."
docker run --rm logstash-with-jdbc:8.11.0 ls -la /usr/share/logstash/drivers/

echo "Build successful! Exporting image..."

# Save the image as a tar file
docker save -o logstash-with-jdbc-8.11.0.tar logstash-with-jdbc:8.11.0

# Get file size
FILE_SIZE=$(ls -lh logstash-with-jdbc-8.11.0.tar | awk '{print $5}')
echo "Image exported successfully!"
echo "File: logstash-with-jdbc-8.11.0.tar"
echo "Size: $FILE_SIZE"
echo ""
echo "Transfer this file to your target server and load it with:"
echo "docker load -i logstash-with-jdbc-8.11.0.tar"
```

### Optional: Multi-plugin Dockerfile
If you want to include other common plugins as well:

```dockerfile
# Dockerfile.full-plugins
FROM docker.elastic.co/logstash/logstash:8.11.0

USER root

# Install multiple useful plugins
RUN /usr/share/logstash/bin/logstash-plugin install \
    logstash-input-jdbc \
    logstash-input-http \
    logstash-input-kafka \
    logstash-filter-json \
    logstash-filter-csv \
    logstash-filter-xml \
    logstash-output-kafka

# Download common JDBC drivers
RUN mkdir -p /usr/share/logstash/drivers

# PostgreSQL driver
RUN curl -L -o /usr/share/logstash/drivers/postgresql-42.6.0.jar \
    https://jdbc.postgresql.org/download/postgresql-42.6.0.jar

# MySQL driver (if needed)
RUN curl -L -o /usr/share/logstash/drivers/mysql-connector-j-8.0.33.jar \
    https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.0.33.jar

# SQL Server driver (if needed)
RUN curl -L -o /usr/share/logstash/drivers/mssql-jdbc-12.2.0.jre11.jar \
    https://github.com/Microsoft/mssql-jdbc/releases/download/v12.2.0/mssql-jdbc-12.2.0.jre11.jar

# Set permissions
RUN chown -R logstash:logstash /usr/share/logstash/drivers
RUN chmod 644 /usr/share/logstash/drivers/*.jar

USER logstash

LABEL version="8.11.0-full-plugins"
LABEL description="Logstash 8.11.0 with JDBC and multiple plugins"
```

## Step 2: Build Commands

Run these commands on your internet-connected server:

```bash
# Make the build script executable
chmod +x build_logstash_image.sh

# Build the image
./build_logstash_image.sh

# Optional: Test the image
docker run --rm -it logstash-with-jdbc:8.11.0 bash -c "
  echo 'Testing plugin installation:' && 
  /usr/share/logstash/bin/logstash-plugin list | grep jdbc &&
  echo 'Testing JDBC driver:' &&
  ls -la /usr/share/logstash/drivers/ &&
  echo 'All tests passed!'
"
```

## Step 3: Transfer to Target Server

```bash
# Compress for faster transfer (optional)
gzip logstash-with-jdbc-8.11.0.tar

# Transfer via SCP, SFTP, USB, etc.
scp logstash-with-jdbc-8.11.0.tar.gz user@target-server:/path/to/elk-stack/

# Or use any file transfer method you prefer
```

## Step 4: On Target Server (Air-gapped)

```bash
# If you compressed it
gunzip logstash-with-jdbc-8.11.0.tar.gz

# Load the Docker image
docker load -i logstash-with-jdbc-8.11.0.tar

# Verify the image is loaded
docker images | grep logstash-with-jdbc

# Should show: logstash-with-jdbc   8.11.0   [IMAGE_ID]   [TIME]   [SIZE]

# Test the loaded image (optional)
docker run --rm logstash-with-jdbc:8.11.0 /usr/share/logstash/bin/logstash-plugin list | grep jdbc
```

## Step 5: Updated docker-compose.yml

```yaml
version: '3.8'
services:
  logstash:
    image: logstash-with-jdbc:8.11.0  # Use your custom image
    container_name: logstash
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      # No need to mount drivers - they're built into the image
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx512m -Xms512m"
    networks:
      - elk-network
    depends_on:
      elasticsearch:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9600 || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
    restart: unless-stopped
  
  # Rest of your ELK services...
```

## Troubleshooting

### If build fails on internet server:
```bash
# Clean Docker cache and rebuild
docker system prune -f
docker build --no-cache -t logstash-with-jdbc:8.11.0 .
```

### If image won't load on target server:
```bash
# Check Docker version compatibility
docker version

# Try loading with verbose output
docker load --input logstash-with-jdbc-8.11.0.tar

# Check available space
df -h
```

### Verify everything works:
```bash
# Start just Logstash to test
docker run --rm -p 9600:9600 logstash-with-jdbc:8.11.0 &

# Test API endpoint
curl http://localhost:9600

# Check plugin list
docker exec [container_id] /usr/share/logstash/bin/logstash-plugin list | grep jdbc
```

This approach completely solves your internet connectivity issue and gives you a portable, self-contained Logstash image with all required plugins and drivers built-in.