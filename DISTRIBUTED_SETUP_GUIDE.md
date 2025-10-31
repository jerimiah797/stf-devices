# Running STF/DeviceFarmer with Docker Compose in a Distributed Architecture

## Overview

This guide documents a production-ready setup for running [DeviceFarmer (formerly STF)](https://github.com/DeviceFarmer/stf) using Docker Compose in a distributed architecture with one central server and multiple remote machines hosting the physical devices.

### What Makes This Guide Different?

The [official STF deployment documentation](https://github.com/DeviceFarmer/stf/blob/master/doc/DEPLOYMENT.md) focuses on a **systemd + Docker** approach, where you manually manage individual Docker containers as systemd services on each host. While powerful and flexible, this approach requires:
- Writing and maintaining individual systemd unit files for each service
- Managing Docker run commands with many flags
- Understanding systemd service dependencies
- Manual configuration distribution across hosts

The official repository also includes a [docker-compose.yaml](https://github.com/DeviceFarmer/stf/blob/master/docker-compose.yaml) example, but it's designed as an **all-in-one development setup** running all services and devices on a single machine.

**This guide bridges the gap** by providing a production-ready Docker Compose setup for a **distributed architecture** where:
- A central server runs the core STF services (web UI, database, API)
- Multiple remote machines physically host the Android devices
- All components are orchestrated via Docker Compose (no systemd units required)
- SSL/TLS is fully configured for secure access
- Custom device metadata and images are managed via Git submodule
- The setup is designed for scalability and production use

### What This Guide Covers (That Official Docs Don't)

1. **True Docker Compose Architecture**: Unlike the official systemd approach or the all-in-one docker-compose example, this uses Docker Compose for both central services AND distributed remotes
2. **Multi-Remote Configuration**: Complete configuration for connecting multiple remote machines with physical devices back to a central server
3. **SSL/TLS Implementation**: Full nginx configuration, certificate setup, and internal TLS handling (the official docs only mention `NODE_TLS_REJECT_UNAUTHORIZED` in passing)
4. **Device Metadata Management**: How to use the stf-devices Git submodule to customize device names, images, and specifications in the UI
5. **Production Hardening**: Security considerations, monitoring, backup strategies, and troubleshooting real-world issues
6. **Custom Docker Images**: Why certain containers need custom Dockerfiles (TLS validation, permissions) and how to build them
7. **Network Architecture**: Detailed explanation of ZeroMQ triproxy connections, port requirements, and firewall configuration

### Why This Architecture?

This distributed Docker Compose architecture provides:
- **Simplicity**: Manage services with `docker-compose up/down` instead of systemd units
- **Portability**: Configuration is code - easily replicate your setup
- **Scalability**: Add more remote machines as your device collection grows
- **Flexibility**: Devices can be located anywhere on your network
- **Reliability**: If one remote goes down, the rest of the farm continues operating
- **Security**: Centralized authentication and encrypted communications
- **Maintainability**: Version control your entire infrastructure configuration

## Architecture Components

### Central Server
Runs on a single machine and hosts:
- **nginx**: SSL termination and reverse proxy
- **rethinkdb**: Database for device state and user management
- **app**: Main web application UI
- **auth**: Authentication service
- **api**: REST API for programmatic access
- **websocket**: Real-time communication with browsers
- **processor**: Device state processing
- **triproxy/dev-triproxy**: ZeroMQ message routing
- **reaper**: Cleans up disconnected devices
- **storage-temp**: Temporary file storage (APKs, screenshots, etc.)
- **storage-plugin-apk**: APK installation handling
- **storage-plugin-image**: Screenshot handling

### Remote Machines
Each remote machine runs:
- **local-adb**: ADB server with direct USB access to devices
- **local-provider**: STF provider that registers devices and handles commands

## Prerequisites

- Central server with Docker and Docker Compose installed
- One or more remote machines with Docker, Docker Compose, and USB ports
- Network connectivity between all machines
- (Optional) SSL certificates for HTTPS access

## Setup Instructions

### 1. Central Server Setup

#### Directory Structure

```
stf-docker-compose/
├── docker-compose.yml          # Main server configuration
├── .env                        # Central server environment variables
├── nginx/
│   ├── Dockerfile
│   ├── nginx.conf              # Reverse proxy configuration
│   ├── entrypoint.sh
│   └── certs/                  # SSL certificates
│       ├── server.crt
│       └── server.key
├── provider/
│   └── Dockerfile              # Custom provider image (disables TLS validation)
├── storage-temp/
│   └── Dockerfile              # Custom storage image (permissions fix)
├── stf-devices/                # Git submodule for device metadata
│   ├── devices-latest.json
│   ├── icon/
│   └── photo/
└── remote1/, remote2/...       # Remote machine configurations (not deployed on central)
```

#### Create .env File

Create `.env` in the root directory:

```bash
PUBLIC_IP=stf.yourdomain.com    # Public hostname/IP for STF
SECRET=your-secret-key-here     # Shared secret for auth (change this!)
RETHINKDB_PORT_28015_TCP=tcp://rethinkdb:28015
STATION_NAME=SRVR               # Name for central server
```

#### Custom Dockerfiles

**provider/Dockerfile** - Disables TLS certificate validation:
```dockerfile
FROM devicefarmer/stf:3.7.7

USER root
ENV NODE_TLS_REJECT_UNAUTHORIZED=0

USER stf
ENV NODE_TLS_REJECT_UNAUTHORIZED=0
```

This is necessary because the storage plugins communicate internally via HTTPS, but use self-signed certificates.

**storage-temp/Dockerfile** - Fixes permissions:
```dockerfile
FROM devicefarmer/stf:3.7.7

USER root
RUN mkdir data && chown stf:stf data
USER stf
VOLUME ["/data"]
```

#### Device Metadata Submodule

Set up the `stf-devices` submodule for custom device information:

```bash
git submodule add https://github.com/yourusername/stf-devices.git
cd stf-devices
# Edit devices-latest.json with your device metadata
```

The submodule is mounted into containers at:
```
~/stf-docker-compose/stf-devices:/app/node_modules/@devicefarmer/stf-device-db/dist
```

This allows you to customize device names, images, and specifications that appear in the UI.

#### Main docker-compose.yml

Key sections of the central server configuration:

```yaml
volumes:
  rethinkdb:      # Persistent database storage
  storage-temp:   # Persistent file storage

services:
  nginx:
    build: nginx/
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/certs:/etc/nginx/certs
    ports:
      - 80:80
      - 443:443
    # ... (see full file for all dependencies)

  rethinkdb:
    image: rethinkdb:2.3
    ports:
      - 8080:8080   # Web admin interface
    volumes:
      - rethinkdb:/data

  app:
    image: devicefarmer/stf:3.7.7
    volumes:
      - ~/stf-docker-compose/stf-devices:/app/node_modules/@devicefarmer/stf-device-db/dist
    command: stf app --auth-url https://${PUBLIC_IP}/auth/mock/ --websocket-url wss://${PUBLIC_IP}/ --port 3000

  api:
    image: devicefarmer/stf:3.7.7
    volumes:
      - ~/stf-docker-compose/stf-devices:/app/node_modules/@devicefarmer/stf-device-db/dist
    command: stf api --port 3000 --connect-sub tcp://triproxy:7150 --connect-push tcp://triproxy:7170 --connect-sub-dev tcp://dev-triproxy:7250 --connect-push-dev tcp://dev-triproxy:7270

  dev-triproxy:
    image: devicefarmer/stf:3.7.7
    command: stf triproxy dev --bind-pub "tcp://*:7250" --bind-dealer "tcp://*:7260" --bind-pull "tcp://*:7270"
    ports:
      - 7250:7250   # Remotes connect here
      - 7270:7270
```

**Important**: Note that the ADB and provider services are commented out in the central docker-compose.yml because devices connect via remote machines.

#### Start Central Server

```bash
cd stf-docker-compose
docker-compose up -d --build
```

### 2. Remote Machine Setup

Each remote machine needs its own configuration to connect devices back to the central server.

#### Directory Structure on Remote

```
stf-docker-compose/
├── remote1/  (or remote2, remote3, etc.)
│   ├── docker-compose.yml
│   ├── .env
│   └── provider/
│       └── Dockerfile  (same as central server's provider/Dockerfile)
└── stf-devices/  (git submodule, same as central)
```

#### Create .env for Remote

Create `remote1/.env`:

```bash
PUBLIC_IP=stf.yourdomain.com                    # Central server public hostname
SECRET=your-secret-key-here                     # Same as central server
RETHINKDB_PORT_28015_TCP=tcp://rethinkdb:28015  # Not used by remote, but required
STATION_NAME=REMOTE1                            # Unique name for this remote
STATION_IP=remote1.yourdomain.com               # This remote's public hostname/IP
```

**Critical**: Each remote must have a unique `STATION_NAME` (e.g., REMOTE1, REMOTE2, BUILDING_A, etc.)

#### Remote docker-compose.yml

```yaml
version: '3'

services:
  local-adb:
    image: devicefarmer/adb:latest
    restart: unless-stopped
    privileged: true
    volumes:
      - /dev/bus/usb:/dev/bus/usb

  local-provider:
    build: provider/
    restart: unless-stopped
    volumes:
      - ~/stf-docker-compose/stf-devices:/app/node_modules/@devicefarmer/stf-device-db/dist
    command: >
      stf provider
      --name ${STATION_NAME}
      --connect-sub tcp://${PUBLIC_IP}:7250
      --connect-push tcp://${PUBLIC_IP}:7270
      --storage-url https://${PUBLIC_IP}/
      --public-ip ${PUBLIC_IP}
      --heartbeat-interval 10000
      --connect-url-pattern "${STATION_IP}:<%= publicPort %>"
      --screen-ws-url-pattern "wss://${PUBLIC_IP}/d/${STATION_NAME}/<%= serial %>/<%= publicPort %>/"
      --adb-host local-adb
      --min-port 7400
      --max-port 7700
      --allow-remote
      --cleanup false
    ports:
      - 7400-7700:7400-7700
    depends_on:
      - local-adb
```

#### Key Configuration Flags

- `--name ${STATION_NAME}`: Unique identifier for this provider
- `--connect-sub tcp://${PUBLIC_IP}:7250`: Connect to central server's dev-triproxy
- `--connect-push tcp://${PUBLIC_IP}:7270`: Send updates to central server
- `--public-ip ${PUBLIC_IP}`: Central server IP (for client routing)
- `--connect-url-pattern "${STATION_IP}:<%= publicPort %>"`: How clients connect to THIS remote
- `--screen-ws-url-pattern`: WebSocket pattern for screen streaming
- `--adb-host local-adb`: Use the local ADB container
- `--min-port/--max-port`: Port range for device connections
- `--allow-remote`: Allow network ADB connections
- `--cleanup false`: Don't reset devices on disconnect

#### USB Memory Buffer Configuration

On each remote machine, increase the USB memory buffer to prevent ADB instability:

```bash
sudo nano /etc/systemd/system/usbfs-memory.service
```

Add:
```ini
[Unit]
Description=Increase USB filesystem memory
DefaultDependencies=no
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo 256 > /sys/module/usbcore/parameters/usbfs_memory_mb'

[Install]
WantedBy=sysinit.target
```

Enable it:
```bash
sudo systemctl enable usbfs-memory.service
sudo systemctl start usbfs-memory.service
```

#### Start Remote Provider

```bash
cd stf-docker-compose/remote1
docker-compose up -d --build
```

#### Verify Remote Connection

Check logs to ensure connection to central server:
```bash
docker-compose logs -f local-provider
```

You should see messages about connecting to the triproxy and devices being registered.

### 3. Network and Firewall Configuration

#### Ports to Open

**Central Server**:
- `80`: HTTP (redirect to HTTPS)
- `443`: HTTPS (web UI)
- `7250`: ZeroMQ SUB (remotes connect here)
- `7270`: ZeroMQ PUSH (remotes send updates here)
- `8080`: RethinkDB admin (restrict to internal network)

**Remote Machines**:
- `7400-7700`: Device connection ports (must be accessible from central server)

#### DNS/Host Configuration

Ensure that:
- `${PUBLIC_IP}` resolves to the central server
- `${STATION_IP}` for each remote resolves to that specific remote machine
- SSL certificates match these hostnames

### 4. SSL/TLS Setup

#### Generate Self-Signed Certificates (for testing)

```bash
cd nginx/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout server.key -out server.crt \
  -subj "/CN=stf.yourdomain.com"
```

#### Use Let's Encrypt (for production)

```bash
# Install certbot
sudo apt-get install certbot

# Get certificates
sudo certbot certonly --standalone -d stf.yourdomain.com

# Copy to nginx/certs/
sudo cp /etc/letsencrypt/live/stf.yourdomain.com/fullchain.pem nginx/certs/server.crt
sudo cp /etc/letsencrypt/live/stf.yourdomain.com/privkey.pem nginx/certs/server.key
```

### 5. Device Management

#### Adding Device Metadata

When you connect a new device, add its metadata to the `stf-devices` submodule:

1. Connect via ADB and gather device info
2. Fetch specifications from GSMArena or manufacturer specs
3. Add entry to `devices-latest.json` with device model as key
4. Add device images (icon/x24, icon/x120, photo/x800)
5. Commit and push submodule
6. Update parent repo to reference new submodule commit
7. Pull changes on server and restart containers

See the `/add-device` slash command for the detailed workflow.

#### Removing a Device from the System

Open RethinkDB admin interface at `https://stf.yourdomain.com:8080`, go to Data Explorer, and run:

```javascript
r.db('stf').table('devices').get('DEVICE_SERIAL').delete()
```

### 6. User Management

STF uses mock authentication by default. To list all users:

```javascript
r.db('stf').table('users')
```

To delete a user:

```javascript
r.db('stf').table('users').get('user@email.com').delete()
```

For production, consider implementing LDAP or OAuth authentication (requires modifying the `auth` service configuration).

## Troubleshooting

### Remote Provider Not Connecting

**Symptoms**: Devices don't appear in STF UI

**Solutions**:
1. Check remote logs: `docker-compose logs local-provider`
2. Verify firewall allows connections to central server ports 7250/7270
3. Verify PUBLIC_IP in remote's .env matches central server hostname
4. Ensure SECRET matches between central and remote
5. Check network connectivity: `telnet central-server 7250`

### Device Image/Name Not Showing

**Symptoms**: Device appears as generic/unknown

**Solutions**:
1. Check server logs for "Unable to find device data" warning with model name
2. Verify JSON key in devices-latest.json matches device's reported model exactly
   - Watch for leading/trailing spaces
   - Spaces become underscores in key
3. Restart containers after updating stf-devices submodule

### SSL Certificate Errors

**Symptoms**: Internal service communication fails

**Solutions**:
1. Verify provider Dockerfile includes `NODE_TLS_REJECT_UNAUTHORIZED=0`
2. Rebuild provider containers: `docker-compose up -d --build`
3. Check nginx certificate paths in nginx.conf

### ADB Instability on Remote

**Symptoms**: Devices randomly disconnect/reconnect

**Solutions**:
1. Verify USB memory buffer is increased (see USB Memory Buffer Configuration)
2. Check for USB hub power issues
3. Reduce number of devices per remote
4. Use powered USB hubs

### Database Issues

**Symptoms**: UI shows errors, devices not persisting

**Solutions**:
1. Check RethinkDB logs: `docker-compose logs rethinkdb`
2. Verify rethinkdb volume has sufficient disk space
3. Run database migrations: `docker-compose run migrate`
4. Backup and restore RethinkDB if corrupted:
   ```bash
   rethinkdb dump -c rethinkdb:28015
   rethinkdb restore dump_file.tar.gz -c rethinkdb:28015
   ```

## Monitoring and Administration

### Web Interfaces

- **STF UI**: `https://stf.yourdomain.com`
- **RethinkDB Admin**: `https://stf.yourdomain.com:8080`
- **Cockpit** (optional): `https://stf.yourdomain.com:9090`

### Checking System Status

```bash
# Central server
docker-compose ps
docker-compose logs -f [service-name]

# Remote machine
cd remote1
docker-compose ps
docker-compose logs -f local-provider

# Check connected devices
docker-compose exec local-adb adb devices
```

### Updating STF Version

1. Update image version in docker-compose.yml files
2. Rebuild custom images:
   ```bash
   docker-compose down
   docker-compose build
   docker-compose up -d
   ```
3. Run migrations if needed:
   ```bash
   docker-compose run migrate
   ```

## Performance Tips

1. **Database Backups**: Schedule regular RethinkDB backups
2. **Storage Cleanup**: Monitor storage-temp volume size, clean periodically
3. **Log Rotation**: Configure Docker log rotation to prevent disk fill
4. **Resource Limits**: Set memory/CPU limits in docker-compose.yml for production
5. **Scaling**: Add more remote machines rather than overloading one with too many devices

## Security Considerations

1. **Change Default Secret**: Use a strong, random SECRET value
2. **Restrict RethinkDB**: Don't expose port 8080 publicly
3. **Use Real Certificates**: Replace self-signed certs with Let's Encrypt or commercial certs
4. **Implement Real Auth**: Replace mock auth with LDAP/OAuth for production
5. **Network Segmentation**: Use VLANs or VPNs to isolate device network
6. **Regular Updates**: Keep Docker images and host OS patched

## Advanced Configuration

### Custom Authentication

Replace the `auth` service command with a different auth provider:

```yaml
auth:
  image: devicefarmer/stf:3.7.7
  command: stf auth-ldap --app-url https://${PUBLIC_IP}/ --port 3000 --ldap-url ldap://your-ldap-server
```

### High Availability

For production environments, consider:
- Running multiple central servers behind a load balancer
- Using RethinkDB clustering for database redundancy
- Shared storage (NFS, S3) for storage-temp

### Monitoring Integration

Add Prometheus/Grafana for monitoring:
- Device availability metrics
- Provider health checks
- Database performance
- Container resource usage

## Comparison: Docker Compose vs. Systemd Deployment

### When to Use Docker Compose (This Guide)

**Recommended for:**
- Teams new to STF wanting a faster path to production
- Organizations preferring infrastructure-as-code
- Setups where you control the entire stack
- Environments with 1-10 remote locations
- Testing and staging environments
- Quick disaster recovery requirements

**Advantages:**
- Faster initial setup and configuration
- Easier to version control and replicate
- Simpler service orchestration and dependencies
- Built-in container networking
- Less OS-specific knowledge required
- Easier to tear down and rebuild

**Disadvantages:**
- Less granular control over individual services
- Docker Compose has some overhead
- May require learning Docker Compose if unfamiliar

### When to Use Systemd (Official Docs)

**Recommended for:**
- Very large deployments (dozens of remote locations)
- Organizations with strong systemd expertise
- Highly customized service configurations
- Integration with existing systemd-based infrastructure
- Maximum performance requirements
- Complex service dependencies and ordering

**Advantages:**
- Fine-grained control over each service
- Native OS integration
- Lower overhead (direct Docker run)
- Better for very large scale
- More flexible service restart policies

**Disadvantages:**
- Steeper learning curve
- More files to maintain
- Manual dependency management
- Less portable across environments
- Longer initial setup time

### Migrating Between Approaches

**From Docker Compose to Systemd:**
The docker-compose.yml files in this guide can serve as a reference for creating systemd units. Each service definition translates roughly to one systemd unit file.

**From Systemd to Docker Compose:**
Extract the `docker run` commands from your systemd units and convert them to docker-compose service definitions. This guide provides the necessary structure.

**From Official All-in-One to Distributed:**
If you started with the official docker-compose.yaml (all-in-one):
1. Deploy this guide's central server setup
2. Move your RethinkDB data volume to the new server
3. Comment out the local adb/provider in the central server
4. Deploy remote machine configurations
5. Update device configurations to point to new central server

## Contributing Back to the Community

If you make improvements to this setup:
1. Document your changes
2. Consider creating a pull request to the main STF repository
3. Share your custom configurations (anonymized) with the community
4. Write blog posts about your experience
5. Submit this guide as a community resource to the DeviceFarmer organization

## Conclusion

This distributed Docker Compose setup for STF/DeviceFarmer provides a scalable, maintainable solution for running a device farm across multiple physical locations. While the initial setup requires some effort, the result is a robust system that can grow with your testing needs.

The key innovations in this setup are:
- **Simplified deployment** using Docker Compose instead of systemd units
- **Separation of concerns** (central services vs. device hosting)
- **Custom Dockerfiles** to work around TLS and permission issues
- **Device metadata management** via Git submodule
- **Clear network architecture** for distributed deployment
- **Production-ready configuration** with SSL/TLS, authentication, and monitoring

This guide fills a gap in the official documentation by providing a complete, working example of a distributed STF deployment using Docker Compose. It took several weeks of trial and error to work out the networking, TLS configuration, and device metadata integration - hopefully this guide saves you that time.

With this foundation, you can easily add more devices, scale to additional remote locations, and maintain a professional-grade mobile device testing infrastructure without the complexity of managing systemd services.

## Real-World Usage

This configuration has been running in production with:
- **18+ Android devices** ranging from Android 7 to Android 15
- **Multiple remote locations** (2+ physical sites)
- **Custom device metadata** with proper names and images for all devices
- **SSL/TLS encryption** for secure access
- **Integration with Marathon testing framework** for automated test distribution

Performance characteristics:
- Device registration takes < 5 seconds per device on restart
- Web UI is responsive even with all devices connected
- Video streaming works smoothly across network boundaries
- System handles device disconnects/reconnects gracefully

## Additional Resources

- [DeviceFarmer GitHub](https://github.com/DeviceFarmer/stf)
- [Official STF Deployment Documentation](https://github.com/DeviceFarmer/stf/blob/master/doc/DEPLOYMENT.md)
- [Official Docker Compose Example](https://github.com/DeviceFarmer/stf/blob/master/docker-compose.yaml) (all-in-one)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [RethinkDB Documentation](https://rethinkdb.com/docs/)
- [Community Docker Compose Examples](https://github.com/Jamesits/docker-compose-devicefarmer)

## Acknowledgments

Special thanks to the DeviceFarmer team for creating and maintaining this excellent open-source project. This guide builds upon their work and aims to make distributed deployments more accessible to the community.

---

*This guide was created based on a production deployment running STF 3.7.7 with multiple remote locations hosting 18+ Android devices. Last updated: October 2025.*

*If you found this guide helpful, please consider starring the repository and sharing it with others who might benefit from a production-ready STF setup.*
