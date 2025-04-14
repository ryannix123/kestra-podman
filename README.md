# Running Kestra with Podman for Event-Driven Terraform & Ansible

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

**Note:** Podman Desktop is recommended for an easier container management experience but is not required. It can be downloaded from [podman-desktop.io](https://podman-desktop.io/)

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

### Copying Data to Volumes

Once your containers are running, you can copy data directly to them:

```bash
# Copy a single file to the PostgreSQL container's volume
podman cp ~/path/to/yourfile kestra-postgres:/var/lib/postgresql/data/

# Copy a directory to the Kestra container's storage (recursive copy)
podman cp ~/path/to/your/directory kestra:/app/storage/
```

Note: Container names may vary based on your setup. Use `podman ps` to verify the correct container names.

### Managing Volumes

View all volumes:
```bash
podman volume ls
```

Remove a volume (only if you want to start fresh - this will delete all data!):
```bash
podman volume rm kestra-postgres-data
```

Note: On macOS and Windows, Podman runs in a virtual machine, so volumes are created inside that VM, not directly on the host filesystem.

## 2. Create a Working Directory

```bash
# Create directory for compose file
mkdir -p ~/kestra
cd ~/kestra
```

## 3. Create podman-compose.yml File

## 4. Start Kestra

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

## 5. Verify Kestra is Running

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

## 6. Access the Kestra UI

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

## 7. Git Integration with Kestra

Kestra offers powerful Git integration capabilities that help you manage infrastructure code effectively:

### Cloning Repositories

You can clone Git repositories directly within Kestra workflows:

```yaml
id: terraform_from_git
namespace: infrastructure
tasks:
  - id: workspace
    type: io.kestra.plugin.core.flow.WorkingDirectory
    tasks:
      - id: clone_repo
        type: io.kestra.plugin.git.Clone
        url: https://github.com/your-org/terraform-configs
        branch: main
        
      - id: terraform_init
        type: io.kestra.plugin.scripts.shell.Commands
        commands:
          - terraform init
```

### Authentication Options

Kestra supports various authentication methods for Git repositories:

- **Public repositories**: No authentication required
- **Username/password**: For private repositories
  ```yaml
  - id: clone_repo
    type: io.kestra.plugin.git.Clone
    url: https://github.com/your-org/private-repo
    branch: main
    username: git_username
    password: "{{ secret('GIT_TOKEN') }}"
  ```
- **SSH with private key**: For SSH-based authentication
  ```yaml
  - id: clone_repo
    type: io.kestra.plugin.git.Clone
    url: git@github.com:your-org/private-repo.git
    privateKey: "{{ secret('SSH_PRIVATE_KEY') }}"
    passphrase: "{{ secret('SSH_PASSPHRASE') }}"
  ```

### Synchronizing Workflows

Kestra can also synchronize flows and namespace files between Git and Kestra, enabling GitOps workflows:

```yaml
id: sync_from_git
namespace: system
tasks:
  - id: sync
    type: io.kestra.plugin.git.SyncFlows
    url: https://github.com/your-org/kestra-flows
    branch: main
    targetNamespace: infrastructure
    delete: true  # Deletes flows not in Git
```

This capability allows you to treat Git as the single source of truth for your infrastructure automation workflows.

Open your browser and navigate to:
```
http://localhost:8080
```

## 8. Creating Event-Driven Workflows

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

## 9. Upload Workflows to Kestra

You can upload these workflows via:

1. The Kestra UI (http://localhost:8080)
2. Using the Kestra CLI
3. Using the Kestra API

## 10. Event-Driven Integration Ideas

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

## 11. Managing Kestra and Persistent Storage

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

### Managing Volumes

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
```

## 12. Upgrading Kestra

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

## 13. Troubleshooting

- **Logs**: Check container logs with `podman logs kestra`
- **Database Connection**: Ensure PostgreSQL is running and accessible
- **Port Conflicts**: Verify port 8080 is available
- **Volume Permissions**: If you encounter permissions issues with volumes, check SELinux context with `podman volume inspect`
- **Version Mismatch**: Verify Podman client and VM versions match with `podman --version` and `podman machine info`

## 14. Additional Resources

- [Kestra Documentation](https://kestra.io/docs)
- [Terraform Documentation](https://www.terraform.io/docs)
- [Ansible Documentation](https://docs.ansible.com/)
- [Podman Documentation](https://docs.podman.io/en/latest/)

## 15. Security Considerations

- Change default passwords in production
- Consider network segmentation
- Implement proper secrets management
- Use HTTPS for production deployments
