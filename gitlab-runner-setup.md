# GitLab Runner Setup Guide

This guide walks you through installing and configuring GitLab Runner on a Linux system.

## Prerequisites

- A Linux system (Ubuntu/Debian/CentOS/RHEL)
- Root or sudo access
- Docker installed (if you plan to use Docker executors)

## Installation Steps

### 1. Download GitLab Runner Binary

Download the latest GitLab Runner binary for your system:

```bash
sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
```

### 2. Grant Execute Permissions

Make the binary executable:

```bash
sudo chmod +x /usr/local/bin/gitlab-runner
```

### 3. Create GitLab Runner User

Create a dedicated user account for GitLab Runner:

```bash
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
```

### 4. Install and Start the Service

Install GitLab Runner as a system service and start it:

```bash
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

### 5. Register the Runner

Register your runner with your GitLab instance:

```bash
sudo gitlab-runner register --url https://vcs.technonext.com --token glrt-j1QPxBZ_Z8uFXPZdzFhnym86MQpwOjEzaQp0OjMKdTpnZxI.01.1b0eevw3m
```

During registration, you'll be prompted to provide:
- Executor type (e.g., docker, shell, kubernetes)
- Default Docker image (if using Docker executor)
- Tags for the runner (optional)

### 6. Configure Docker Access (Optional)

If you're using the Docker executor, add the gitlab-runner user to the docker group:

```bash
sudo usermod -aG docker gitlab-runner
sudo service docker restart
```

### 7. Configure Bash Logout (Optional)

Edit the bash logout file for the gitlab-runner user:

```bash
sudo nano /home/gitlab-runner/.bash_logout
```

Replace contents with:

```bash
# ~/.bash_logout: executed by bash(1) when login shell exits.

# when leaving the console clear the screen to increase privacy

#if [ "$SHLVL" = 1 ]; then
#    [ -x /usr/bin/clear_console ] && /usr/bin/clear_console -q
#fi
```

## Systemd Service

When you run the `gitlab-runner install` command, it automatically creates a systemd service file at `/etc/systemd/system/gitlab-runner.service`.

### Service File Location

```
/etc/systemd/system/gitlab-runner.service
```

### View Service Configuration

To view the auto-generated service file:

```bash
sudo cat /etc/systemd/system/gitlab-runner.service
```

### Typical Service File Content

```ini
[Unit]
Description=GitLab Runner
After=syslog.target network.target
ConditionFileIsExecutable=/usr/local/bin/gitlab-runner

[Service]
StartLimitInterval=5
StartLimitBurst=10
ExecStart=/usr/local/bin/gitlab-runner "run" "--working-directory" "/home/gitlab-runner" "--config" "/etc/gitlab-runner/config.toml" "--service" "gitlab-runner" "--user" "gitlab-runner"

Restart=always
RestartSec=120

[Install]
WantedBy=multi-user.target
```

### Manage Service with Systemctl

You can also manage GitLab Runner using standard systemd commands:

```bash
# Start the service
sudo systemctl start gitlab-runner

# Stop the service
sudo systemctl stop gitlab-runner

# Restart the service
sudo systemctl restart gitlab-runner

# Check service status
sudo systemctl status gitlab-runner

# Enable service to start on boot
sudo systemctl enable gitlab-runner

# Disable service from starting on boot
sudo systemctl disable gitlab-runner

# Reload systemd daemon after manual config changes
sudo systemctl daemon-reload
```

## Verification

Check that the runner is active:

```bash
sudo gitlab-runner status
```

Or using systemctl:

```bash
sudo systemctl status gitlab-runner
```

View registered runners:

```bash
sudo gitlab-runner list
```

## Useful Commands

- **Start runner**: `sudo gitlab-runner start`
- **Stop runner**: `sudo gitlab-runner stop`
- **Restart runner**: `sudo gitlab-runner restart`
- **Unregister runner**: `sudo gitlab-runner unregister --url https://vcs.technonext.com --token <token>`
- **View logs**: `sudo journalctl -u gitlab-runner -f`
