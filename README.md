## 14. Troubleshooting

- **Logs**: Check container logs with `podman logs kestra`
- **Database Connection**: Ensure PostgreSQL is running and accessible
- **Port Conflicts**: Verify port 8080 is available
- **Volume Permissions**: If you encounter permissions issues with volumes, check SELinux context with `podman volume inspect`
- **Version Mismatch**: Verify Podman client and VM versions match with `podman --version` and `podman machine info`### Windows Volume Considerations

When using Podman on Windows, volumes have similar considerations to macOS since Podman runs in a WSL2 VM:

1. Volume paths in Windows:
```powershell
# In Windows, volumes are stored inside the Podman VM, not directly on Windows filesystem
podman machine ssh
ls -la /var/lib/containers/storage/volumes/
```

2. To mount host directories, use:
```powershell
# Stop existing machine if running
podman machine stop

# Create new machine with host directory mounted
podman machine init -v C:\Users\username\data:/data

# Start the machine
podman machine start
```

3. Alternative: Use named volumes (recommended approach):
```powershell
# Create volume
podman volume create kestra-postgres-data

# Use in container
podman run -v kestra-postgres-data:/var/lib/postgresql/data postgres:15
```

4. For better performance on Windows:
   - Store data in WSL2 filesystem when possible
   - Avoid Windows paths with spaces
   - Use named volumes for best compatibility### macOS Volume Considerations

When running Podman on macOS, volumes work differently than on Linux because Podman runs inside a virtual machine:

1. Volume paths in macOS:
```bash
# In macOS, volumes are stored inside the Podman VM, not directly on macOS filesystem
podman machine ssh
ls -la /var/lib/containers/storage/volumes/
```

2. To mount host directories, you need to specify them when creating the VM:
```bash
# Stop existing machine if running
podman machine stop

# Create new machine with host directory mounted
podman machine init -v $HOME:/home/user

# Start the machine
podman machine start
```

3. Alternative: Use named volumes (recommended approach):
```bash
# Create volume
podman volume create kestra-postgres-data

# Use in container
podman run -v kestra-postgres-data:/var/lib/postgresql/data postgres:15
```

4. Troubleshooting volume access on macOS:
   - If you get "no such file or directory" errors, recreate the VM with proper mounts
   - Check the Podman version with `podman --version` and `podman machine info`
   - Ensure client and VM versions match (if not, recreate VM)## 2. Additional macOS and Windows Considerations

### macOS Considerations

When running Podman on macOS, keep in mind:

1. Podman runs in a VM on macOS:
```bash
# Check Podman machine status
podman machine list

# SSH into Podman VM if needed
podman machine ssh
```

2. Volume paths are relative to the VM, not your Mac:
```bash
# See where volumes are stored in the VM
podman machine ssh
ls -la /var/lib/containers/storage/volumes/
```

3. Port forwarding works automatically through the VM

4. Resource allocation:
```bash
# Adjust VM resources (CPU, memory)
podman machine stop
podman machine set --cpus 2 --memory 4096
podman machine start
```

### Windows Considerations

1. Podman runs in a WSL2 VM on Windows:
```powershell
# Check Podman machine status
podman machine list

# SSH into Podman VM if needed
podman machine ssh
```

2. Similar to macOS, volume paths are relative to the VM, not your Windows filesystem

3. For better performance, consider storing volumes in the WSL filesystem instead of Windows-mounted directories- [Podman Documentation](https://docs.podman.io/en/latest/)### Managing Volumes

View all volumes:
```bash
podman volume ls
```

Inspect volume details:
```bash
podman volume inspect kestra-postgres-data
```

Remove a volume (only if you want to start fresh - this will delete all data!):
```bash
podman volume rm kestra-postgres-data
```

### Volume Location
By default, Podman volumes are stored in:
```
/var/lib/containers/storage/volumes/
```

You can customize the location when creating volumes with:
```bash
podman volume create --opt device=/path/to/custom/location/postgres-data kestra-postgres-data
```## 4. Create podman-compose.yml File# Running Kestra with Podman for Event-Driven Terraform & Ansible

## Why Kestra for Terraform and Ansible?

Kestra is a powerful workflow orchestration platform that provides significant benefits when used with infrastructure-as-code tools like Terraform and Ansible:

- **Unified Orchestration**: Kestra allows you to combine Terraform, Ansible, and other DevOps tools into a single declarative workflow, eliminating tool silos and creating a cohesive automation system.

- **Event-Driven Infrastructure**: Trigger infrastructure changes automatically in response to events (webhooks, file changes, schedules) rather than relying solely on manual execution.

- **Improved Visibility**: Track Terraform plans, Ansible playbook executions, logs, and state changes through Kestra's intuitive UI for better observability.

- **Error Handling**: Implement sophisticated retry mechanisms, notifications, and conditional workflows when infrastructure operations fail or need approvals.

- **State Management**: Securely pass variables and outputs between tasks (e.g., Terraform outputs to Ansible variables) with built-in state management.

- **GitOps Integration**: Synchronize your infrastructure-as-code with Git for version control and auditability while maintaining execution history.

This guide walks you through setting up Kestra using Podman containers with persistent storage for orchestrating event-driven infrastructure workflows with Terraform and Ansible.

## Prerequisites

- Podman - Container management tool (required)
- podman-compose - Multi-container orchestration tool (recommended)
- Terraform and/or Ansible installed (for local execution)

For installation instructions for all platforms (Linux, macOS, Windows), please refer to:
- [Podman Installation Guide](https://podman.io/docs/installation)
- [Podman Compose Installation](https://github.com/containers/podman-compose)

**Note:** Podman Desktop is recommended for an easier container management experience but is not required. It can be downloaded from [podman-desktop.io](https://podman-desktop.io/).

## 1. Create Persistent Storage Volumes

Create Podman volumes for persistent storage to ensure your data survives container restarts and rebuilds:

```bash
# Create named volumes for PostgreSQL data
podman volume create kestra-postgres-data

# Create named volume for Kestra application storage
podman volume create kestra-storage

# Verify volumes were created
podman volume ls
```

These volumes will be mounted to the containers to store:
- PostgreSQL database files (workflows, triggers, executions history)
- Kestra application files (logs, temporary files, outputs)

You can inspect volume details with:
```bash
podman volume inspect kestra-postgres-data
podman volume inspect kestra-storage
```

## 3. Create a Working Directory

```bash
# Create directory for compose file
mkdir -p ~/kestra
cd ~/kestra
```

## 3. Create Persistent Storage Volumes

Create Podman volumes for persistent storage to ensure your data survives container restarts and rebuilds:

```bash
# Create named volumes for PostgreSQL data
podman volume create kestra-postgres-data

# Create named volume for Kestra application storage
podman volume create kestra-storage

# Verify volumes were created
podman volume ls
```

These volumes will be mounted to the containers to store:
- PostgreSQL database files (workflows, triggers, executions history)
- Kestra application files (logs, temporary files, outputs)

You can inspect volume details with:
```bash
podman volume inspect kestra-postgres-data
podman volume inspect kestra-storage
```

The volumes are stored in Podman's storage directory, typically `/var/lib/containers/storage/volumes/` on most systems.

## 5. Create podman-compose.yml File

```bash
cat > ~/kestra/podman-compose.yml << 'EOF'
version: '3'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: kestra
      POSTGRES_PASSWORD: k3str4
      POSTGRES_DB: kestra
    volumes:
      - kestra-postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kestra"]
      interval: 30s
      timeout: 10s
      retries: 5

  kestra:
    image: kestra/kestra:latest
    pull: always
    depends_on:
      - postgres
    ports:
      - "8080:8080"
    environment:
      KESTRA_CONFIGURATION: |
        datasources:
          postgres:
            url: jdbc:postgresql://postgres:5432/kestra
            driverClassName: org.postgresql.Driver
            username: kestra
            password: k3str4
        kestra:
          repository:
            type: postgres
          queue:
            type: postgres
          storage:
            type: local
            local:
              basePath: "/app/storage"
        server:
          gracefulShutdown: PT50S
    volumes:
      - kestra-storage:/app/storage
    command: server standalone
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/api/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  kestra-postgres-data:
    external: true
  kestra-storage:
    external: true
EOF
```

## 6. Start Kestra

Using podman-compose:
```bash
cd ~/kestra
podman-compose -f podman-compose.yml up -d
```

Alternatively, using pods directly:
```bash
# Create a pod
podman pod create --name kestra-pod -p 8080:8080

# Create volumes
podman volume create kestra-postgres-data
podman volume create kestra-storage

# Run postgres
podman run -d --pod kestra-pod \
  --name kestra-postgres \
  -e POSTGRES_USER=kestra \
  -e POSTGRES_PASSWORD=k3str4 \
  -e POSTGRES_DB=kestra \
  -v kestra-postgres-data:/var/lib/postgresql/data \
  postgres:15

# Run kestra
podman run -d --pod kestra-pod \
  --name kestra \
  -e KESTRA_CONFIGURATION="datasources:
  postgres:
    url: jdbc:postgresql://localhost:5432/kestra
    driverClassName: org.postgresql.Driver
    username: kestra
    password: k3str4
kestra:
  repository:
    type: postgres
  queue:
    type: postgres
  storage:
    type: local
    local:
      basePath: /app/storage" \
  -v kestra-storage:/app/storage \
  kestra/kestra:latest server standalone
```

## 7. Verify Kestra is Running

Check container status:
```bash
podman-compose -f podman-compose.yml ps
# or
podman pod ps
```

Check logs:
```bash
podman-compose -f podman-compose.yml logs -f kestra
# or
podman logs kestra
```

## 8. Access the Kestra UI

Open your browser and navigate to:
```
http://localhost:8080
```

## 9. Creating Event-Driven Workflows

### Create a Basic Terraform Workflow

Create a file in a directory, for example `~/kestra/flows/terraform-flow.yml`:

```yaml
id: terraform_example
namespace: infrastructure
description: "Event-driven Terraform workflow"

triggers:
  - id: schedule
    type: io.kestra.core.models.triggers.types.Schedule
    cron: "0 0 * * *"  # Daily at midnight
    
tasks:
  - id: terraform_init
    type: io.kestra.plugin.scripts.shell.Commands
    commands:
      - terraform init
    workingDirectory: /path/to/terraform/directory

  - id: terraform_plan
    type: io.kestra.plugin.scripts.shell.Commands
    commands:
      - terraform plan -out=tfplan
    workingDirectory: /path/to/terraform/directory
    
  - id: terraform_apply
    type: io.kestra.plugin.scripts.shell.Commands
    commands:
      - terraform apply -auto-approve tfplan
    workingDirectory: /path/to/terraform/directory

  - id: notify_success
    type: io.kestra.core.tasks.log.Log
    message: "Terraform infrastructure successfully updated!"
```

### Create a Basic Ansible Workflow

Create a file in a directory, for example `~/kestra/flows/ansible-flow.yml`:

```yaml
id: ansible_example
namespace: infrastructure
description: "Event-driven Ansible workflow"

triggers:
  - id: webhook
    type: io.kestra.core.models.triggers.types.Webhook
    
tasks:
  - id: run_ansible_playbook
    type: io.kestra.plugin.scripts.shell.Commands
    commands:
      - ansible-playbook -i inventory.yml playbook.yml
    workingDirectory: /path/to/ansible/directory

  - id: notify_success
    type: io.kestra.core.tasks.log.Log
    message: "Ansible playbook successfully executed!"
```

## 10. Upload Workflows to Kestra

You can upload these workflows via:

1. The Kestra UI (http://localhost:8080)
2. Using the Kestra CLI
3. Using the Kestra API

## 11. Event-Driven Integration Ideas

### File-Based Trigger
Configure Kestra to watch for file changes:

```yaml
triggers:
  - id: file_watch
    type: io.kestra.plugin.fs.watch.Watch
    directory: "/path/to/watch"
    pattern: "*.yml"
```

### Conditional Execution Based on Events
Execute Terraform or Ansible based on specific conditions:

```yaml
tasks:
  - id: check_condition
    type: io.kestra.core.tasks.flows.Switch
    value: "{{ trigger.event.type }}"
    cases:
      INFRASTRUCTURE_UPDATE:
        - id: run_terraform
          type: io.kestra.plugin.scripts.shell.Commands
          commands:
            - terraform apply
      CONFIGURATION_UPDATE:
        - id: run_ansible
          type: io.kestra.plugin.scripts.shell.Commands
          commands:
            - ansible-playbook playbook.yml
```

## 12. Managing Kestra and Persistent Storage

### Stopping Kestra
```bash
cd ~/kestra
podman-compose -f podman-compose.yml down
# or
podman pod stop kestra-pod
podman pod rm kestra-pod
```

### Data Persistence
The volumes we created (`kestra-postgres-data` and `kestra-storage`) will preserve your data even when containers are stopped or removed. Your data includes:

- Database content (flows, executions, triggers, etc.)
- Storage files (task outputs, logs, temporary files)

### Backing Up Data
To back up your data:

```bash
# For PostgreSQL data
podman run --rm -v kestra-postgres-data:/data -v $(pwd):/backup alpine tar -czvf /backup/postgres-backup.tar.gz /data

# For Kestra storage
podman run --rm -v kestra-storage:/data -v $(pwd):/backup alpine tar -czvf /backup/kestra-storage-backup.tar.gz /data
```

### Restoring Data
To restore from backups:

```bash
# For PostgreSQL data
podman run --rm -v kestra-postgres-data:/data -v $(pwd):/backup alpine sh -c "rm -rf /data/* && tar -xzvf /backup/postgres-backup.tar.gz -C /"

# For Kestra storage
podman run --rm -v kestra-storage:/data -v $(pwd):/backup alpine sh -c "rm -rf /data/* && tar -xzvf /backup/kestra-storage-backup.tar.gz -C /"
```

## 13. Upgrading Kestra

To upgrade your Kestra installation to the latest version, follow these steps:

### Using podman-compose

1. Update your podman-compose.yml file to specify the new version:

```yaml
services:
  kestra:
    # Change from
    # image: kestra/kestra:latest
    # to a specific version
    image: kestra/kestra:v0.19.0  # Replace with desired version
    pull_policy: always
    # ... rest of configuration
```

2. Pull the new image and restart the containers:

```bash
cd ~/kestra
podman-compose pull
podman-compose down
podman-compose up -d
```

### Using Podman directly

1. Stop and remove the existing containers:

```bash
podman pod stop kestra-pod
podman pod rm kestra-pod
```

2. Create the pod and containers with the new version:

```bash
# Create a pod
podman pod create --name kestra-pod -p 8080:8080

# Run postgres
podman run -d --pod kestra-pod \
  --name kestra-postgres \
  -e POSTGRES_USER=kestra \
  -e POSTGRES_PASSWORD=k3str4 \
  -e POSTGRES_DB=kestra \
  -v kestra-postgres-data:/var/lib/postgresql/data \
  postgres:15

# Run kestra with new version
podman run -d --pod kestra-pod \
  --name kestra \
  -e KESTRA_CONFIGURATION="datasources:
  postgres:
    url: jdbc:postgresql://localhost:5432/kestra
    driverClassName: org.postgresql.Driver
    username: kestra
    password: k3str4
kestra:
  repository:
    type: postgres
  queue:
    type: postgres
  storage:
    type: local
    local:
      basePath: /app/storage" \
  -v kestra-storage:/app/storage \
  kestra/kestra:v0.19.0 server standalone  # Replace with desired version
```

### Version Options

- `latest`: Always pulls the most recent stable release
- `v0.19.0`: Specific version (replace with the actual version number)
- `develop`: Latest development build (not recommended for production)

Kestra releases new versions regularly, so check the [official documentation](https://kestra.io/docs/administrator-guide/upgrades) for the latest version information.

## 14. Additional Resources

- [Kestra Documentation](https://kestra.io/docs)
- [Terraform Documentation](https://www.terraform.io/docs)
- [Ansible Documentation](https://docs.ansible.com/)

## 16. Security Considerations

- Change default passwords in production
- Consider network segmentation
- Implement proper secrets management
- Use HTTPS for production deployments
