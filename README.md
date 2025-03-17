# Installing Quay Registry on RHEL 9

There are times when you need an internal container registry. e.g., Air-gapped OpenShift testing. This guide provides step-by-step instructions for setting up a Red Hat Quay registry on a Red Hat Enterprise Linux (RHEL) 9 system.

## Prerequisites

- RHEL 9 system
- Root or sudo access
- Valid Red Hat subscription, which includes free accounts.
- Internet connectivity

## Installation Steps

### 1. System Preparation

Ensure your RHEL 9 system is registered and subscribed:

```bash
sudo subscription-manager register
sudo subscription-manager attach
```

Update the system:

```bash
sudo dnf update -y
```

Install required packages:

```bash
sudo dnf install -y podman container-tools
```

### 2. Directory Structure

Create the directory structure for Quay:

```bash
sudo mkdir -p /var/lib/quay/{storage,config,postgres}
```

### 3. Container Registry Access

Set up credentials for pulling Red Hat container images:

```bash
sudo subscription-manager repos --enable=registry-rpms
sudo dnf install -y subscription-manager-rhsm-certificates
sudo podman login registry.redhat.io
```

### 4. Quay Configuration

Run the Quay configuration tool:

```bash
sudo podman run --rm -it --name quay_config -p 8080:8080 \
  -v /var/lib/quay/config:/conf/stack:Z \
  registry.redhat.io/quay/quay-rhel9:v3.8.0 config
```

Access the configuration UI at http://localhost:8080 in your browser to set up:
- Database configuration
- Redis settings (optional)
- Authentication providers
- Registry settings and preferences

Download the configuration bundle when complete.

Extract the configuration bundle:

```bash
sudo tar xvf quay-config.tar.gz -C /var/lib/quay/config
```

### 5. Database Setup

Start the PostgreSQL database:

```bash
sudo podman run -d --name postgresql \
  -e POSTGRES_USER=quayuser \
  -e POSTGRES_PASSWORD=quaypass \
  -e POSTGRES_DB=quay \
  -v /var/lib/quay/postgres:/var/lib/postgresql/data:Z \
  -p 5432:5432 \
  registry.redhat.io/rhel9/postgresql-13:latest
```

### 6. Quay Deployment

Deploy the Quay container:

```bash
sudo podman run -d --name quay \
  -p 80:8080 -p 443:8443 \
  -v /var/lib/quay/config:/conf/stack:Z \
  -v /var/lib/quay/storage:/datastorage:Z \
  registry.redhat.io/quay/quay-rhel9:v3.8.0
```

### 7. Firewall Configuration

Configure the firewall:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

### 8. Verification

Verify the deployment:

```bash
sudo podman ps
```

### 9. Auto-start with systemd

Create systemd service files for automatic startup:

```bash
sudo mkdir -p /etc/systemd/system/
sudo podman generate systemd --name quay > /etc/systemd/system/quay.service
sudo podman generate systemd --name postgresql > /etc/systemd/system/quay-postgresql.service
sudo systemctl enable --now quay-postgresql.service
sudo systemctl enable --now quay.service
```

## Production Considerations

For production deployments, consider:

- **High Availability**: Configure multiple Quay instances behind a load balancer
- **External Storage**: Use S3-compatible storage for image data
- **TLS Certificates**: Configure proper certificates for secure communication
- **Redis Caching**: Set up Redis for improved performance
- **Backup Strategy**: Implement regular backups of database and configuration
- **Monitoring**: Set up monitoring and alerting for the Quay instance
- **Resource Allocation**: Ensure sufficient CPU, memory, and storage

## Troubleshooting

### Common Issues

1. **Container fails to start**:
   - Check logs: `sudo podman logs quay`
   - Verify configuration: Check config files in `/var/lib/quay/config`

2. **Database connection issues**:
   - Verify PostgreSQL is running: `sudo podman ps | grep postgresql`
   - Check database credentials in config.yaml

3. **Permission errors**:
   - SELinux issues: Ensure volumes are mounted with `:Z` suffix
   - Check directory permissions: `ls -la /var/lib/quay/`

## Upgrading

To upgrade Quay to a newer version:

1. Back up your configuration and database
2. Pull the new image: `sudo podman pull registry.redhat.io/quay/quay-rhel9:[new-version]`
3. Stop the current container: `sudo podman stop quay`
4. Remove the container: `sudo podman rm quay`
5. Start a new container with the updated image using the same commands as in step 6

## References

- [Red Hat Quay Documentation](https://access.redhat.com/documentation/en-us/red_hat_quay)
- [RHEL 9 Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9)
