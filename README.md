# Ansible Management for Raspberry Pi Docker Services

**Current Date:** May 11, 2025

This Ansible project automates the setup, deployment, and management of Dockerized services (Pi-hole, custom web applications, etc.) on a Raspberry Pi. It leverages Git submodules to include the source code of individual web applications, which are then built as Docker images directly on the Pi.

## Table of Contents

1.  [Project Goals](#project-goals)
2.  [Overview of Approach](#overview-of-approach)
3.  [Project Structure](#project-structure)
4.  [Prerequisites](#prerequisites)
    * [On the Raspberry Pi](#on-the-raspberry-pi)
    * [On the Ansible Control Node](#on-the-ansible-control-node)
5.  [Initial Setup](#initial-setup)
    * [Step 1: Prepare the Raspberry Pi](#step-1-prepare-the-raspberry-pi)
    * [Step 2: Prepare the Control Node](#step-2-prepare-the-control-node)
    * [Step 3: Configure This Ansible Project](#step-3-configure-this-ansible-project)
6.  [Running the Ansible Playbook](#running-the-ansible-playbook)
7.  [Services Managed](#services-managed)
8.  [Application Source Code (Submodules)](#application-source-code-submodules)
9.  [Backup and Restore Strategy](#backup-and-restore-strategy)
10. [Updating Services](#updating-services)
    * [Updating Pi-hole or Other Pre-built Images](#updating-pi-hole-or-other-pre-built-images)
    * [Updating Your Web Applications](#updating-your-web-applications)
11. [Troubleshooting with Ansible](#troubleshooting-with-ansible)
12. [Troubleshooting Docker Services on the Pi](#troubleshooting-docker-services-on-the-pi)

## Project Goals

* **Automated Setup:** Quickly provision a new Raspberry Pi from a base OS to a fully functional application server.
* **Declarative Configuration:** Define the desired state of the system using Ansible playbooks and roles.
* **Version Controlled Infrastructure:** All configurations, deployment logic, and application sources (via submodules) are stored in Git.
* **Resilience:** Facilitate easy recovery and re-setup in case of SD card failure.
* **Modularity:** Organize tasks into reusable Ansible roles.

## Overview of Approach

This Ansible setup automates the provisioning of a Raspberry Pi for homelab purposes. It performs the following:
* Basic system configuration (updates, required packages like Git).
* Installation of Docker and the Docker Compose plugin.
* Synchronization of this repository's relevant parts (including web app submodules) to the Pi.
* Deployment of `docker-compose.yml` (from the `app_deployment` role) to the target directory on the Pi.
* Creation of necessary directories on the Pi for Docker volumes (e.g., for Pi-hole config, web app data).
* Deployment and management of Dockerized services defined in the deployed `docker-compose.yml` file.

## Project Structure

Here is the intended structure for this Ansible project:

```
ansible_rpi_homelab/        # Your main Ansible project directory
├── README.md
├── ansible.cfg             # Optional: Ansible configuration (e.g., default user, roles path)
├── inventory.ini           # Defines your Raspberry Pi host(s) and connection details
├── playbook.yml            # Main Ansible playbook to run
│
├── roles/                  # Reusable components for organizing tasks
│   ├── common/             # Role for basic system setup
│   │   └── tasks/main.yml
│   ├── docker/             # Role for Docker & Docker Compose installation
│   │   └── tasks/main.yml
│   └── app_deployment/     # Role for deploying your services & configurations
│       ├── files/
│       │   └── docker-compose.yml  # Your main Docker Compose file
│       ├── tasks/main.yml          # Tasks to copy docker-compose.yml, sync submodules, run compose
│       └── vars/main.yml           # Variables specific to app_deployment role
│
├── webapp1_src/            # Git submodule for your first web app (at project root)
│   ├── Dockerfile
│   ├── requirements.txt
│   └── ...
├── webapp2_src/            # Git submodule for your second web app (at project root)
│   └── ...
│
└── scripts/                # Directory for helper scripts (also at project root)
├── backup_data.sh
└── restore_data.sh
```

## Prerequisites

### On the Raspberry Pi

* A Raspberry Pi (Model 4 or newer recommended).
* A reliable microSD card (32GB or larger, high endurance recommended).
* Raspberry Pi OS Lite (64-bit) installed.
* SSH access enabled, and a user account configured (this will be your `ansible_user`).
* The Pi connected to your network (wired Ethernet is recommended for servers, but WiFi works).
* Python 3 installed (usually comes with Raspberry Pi OS Lite, Ansible requires it on the target).

### On the Ansible Control Node

(This is the machine you'll run Ansible commands from, e.g., your Macbook or another Linux system.)

* **Ansible installed:** Version 2.10 or newer recommended.
* **Git installed:** To clone this repository and manage submodules.
* **SSH client:** To connect to the Raspberry Pi. SSH key-based authentication is highly recommended.

## Initial Setup

### Step 1: Prepare the Raspberry Pi

1.  **Flash Raspberry Pi OS Lite (64-bit):**
    * Use Raspberry Pi Imager. In advanced options (⚙️ icon or Ctrl+Shift+X):
        * Set a hostname (e.g., `rpi-homelab`).
        * Enable SSH and create a user (e.g., `pi_admin`) with a strong password. This is your `ansible_user`.
        * Configure WiFi if needed. Set locale/timezone.
2.  **First Boot & Network:**
    * Boot Pi, find its IP or use hostname.
3.  **(Highly Recommended) Set up SSH Key Authentication for Ansible:**
    * From control node: `ssh-keygen -t ed25519 -C "your_email@example.com"`
    * Copy public key: `ssh-copy-id pi_admin@<raspberry_pi_ip_or_hostname>`
    * Test passwordless SSH: `ssh pi_admin@<raspberry_pi_ip_or_hostname>`
    * Optionally, disable password authentication on Pi via `/etc/ssh/sshd_config`.

### Step 2: Prepare the Control Node

1.  **Clone This Ansible Project Repository:**
    ```bash
    git clone --recurse-submodules <URL_of_THIS_Ansible_project_repo> ansible_rpi_homelab
    cd ansible_rpi_homelab
    ```
    *(Replace `<URL_...>` with your actual Git repo URL).*
    If cloned without `--recurse-submodules`, run `git submodule update --init --recursive` inside the directory.

2.  **Add your web app repositories as submodules (if not done during clone):**
    Place submodules at the root of this Ansible project (e.g., `webapp1_src`, `webapp2_src`).
    ```bash
    git submodule add <URL_to_your_webapp1_repo> webapp1_src
    # Repeat for other apps...
    git commit -m "Added web application submodules"
    # Push changes to your Ansible project remote
    ```

### Step 3: Configure This Ansible Project

1.  **Configure `inventory.ini`:**
    Edit `inventory.ini` with your Pi's IP/hostname and `ansible_user`.
    ```ini
    [raspberry_pi_hosts]
    rpi-homelab ansible_host=192.168.1.XX ansible_user=pi_admin

    [raspberry_pi_hosts:vars]
    ansible_python_interpreter=/usr/bin/python3
    # ansible_become_pass=your_sudo_password_here # (Or use --ask-become-pass)
    ```

2.  **Review `playbook.yml` Variables:**
    Adjust global `vars` in `playbook.yml` if needed:
    ```yaml
    vars:
      project_target_path: "/opt/rpi_homelab_services" # Base directory on the Pi for all deployed files
      # ...
    ```

3.  **Customize `roles/app_deployment/files/docker-compose.yml`:**
    * Place your main `docker-compose.yml` file into the `ansible_rpi_homelab/roles/app_deployment/files/` directory.
    * The `app_deployment` role will be responsible for copying this file to `{{ project_target_path }}/docker-compose.yml` on the Raspberry Pi.
    * **Crucially:** In this `docker-compose.yml`, `build: context:` paths for your web apps should be relative to where `docker-compose.yml` will *reside on the Pi* alongside the synced submodule directories. For example, if `webapp1_src` is synced to `{{ project_target_path }}/webapp1_src/`, then the context should be `../webapp1_src` (if `docker-compose.yml` is at `{{ project_target_path }}/docker-compose.yml`) or simply `webapp1_src` if you adjust the `project_src` for the docker_compose module to be `{{ project_target_path }}` and structure your project sync accordingly.
    * **Recommendation for simplicity:** The `app_deployment` role should sync the web app submodules (e.g., `webapp1_src`) into `{{ project_target_path }}/webapp1_src/` on the Pi. Then, in `roles/app_deployment/files/docker-compose.yml`, the build contexts can be straightforward, like:
        ```yaml
        services:
          webapp1:
            build:
              context: ./webapp1_src # This path is relative to {{ project_target_path }} on the Pi
        # ...
        ```
    * Volume paths in `docker-compose.yml` should also be relative (e.g., `./pihole_config:/etc/pihole`). These will be created relative to `{{ project_target_path }}` on the Pi.

4.  **Define Web App Submodules in `roles/app_deployment/vars/main.yml`:**
    This file is useful for listing your submodules, which can be referenced by tasks in the `app_deployment` role (e.g., for explicitly syncing them or ensuring their directories exist).
    Edit `ansible_rpi_homelab/roles/app_deployment/vars/main.yml`:
    ```yaml
    # roles/app_deployment/vars/main.yml
    ---
    # project_target_path is defined globally in playbook.yml
    # List of web app submodule directory names (must match directories at Ansible project root)
    web_app_submodules:
      - webapp1_src
      - webapp2_src
      # Add more as they are created at your Ansible project root
    ```
    The `app_deployment` role tasks (in `tasks/main.yml`) would use this list. For instance, a task to synchronize submodules might look like:
    ```yaml
    # Example task in roles/app_deployment/tasks/main.yml
    - name: Synchronize web app submodule directories to target
      ansible.posix.synchronize:
        src: "{{ item }}" # item is from the loop, e.g., "webapp1_src"
        dest: "{{ project_target_path }}/{{ item }}"
        delete: true
        rsync_opts:
          - "--exclude=.git" # Important to exclude .git from submodules if only code is needed
      loop: "{{ web_app_submodules }}"
      # Note: This assumes submodules are at the root of your Ansible project.
      # The `ansible.builtin.git` module can also be used to manage submodules directly on the target.
    ```

5.  **Customize `scripts/backup_data.sh` and `scripts/restore_data.sh`:**
    * These scripts are located at the root of your Ansible project (e.g., `ansible_rpi_homelab/scripts/`). The `app_deployment` role should include tasks to copy these scripts to `{{ project_target_path }}/scripts/` on the Pi and make them executable.
    * Edit `scripts/backup_data.sh` regarding `BACKUP_TARGET_DIR` (off-Pi location) and `APP_DATA_DIRS` (relative to `{{ project_target_path }}`).
    * Review `scripts/restore_data.sh`.

## Running the Ansible Playbook

1.  **Navigate to the Ansible Project Directory on your Control Node.**
2.  **Test Ansible Connection (Optional):** `ansible raspberry_pi_hosts -m ping`
3.  **Syntax Check Playbook:** `ansible-playbook playbook.yml --syntax-check`
4.  **Dry Run (Check Mode):** `ansible-playbook playbook.yml --check --diff --ask-become-pass`
5.  **Execute the Playbook:** `ansible-playbook playbook.yml --ask-become-pass`
    This will execute tasks, including copying `roles/app_deployment/files/docker-compose.yml` to `{{ project_target_path }}/docker-compose.yml`, syncing submodules and scripts, and then running `docker compose up -d --build` from `{{ project_target_path }}` on the Pi.

## Services Managed

(List your services like Pi-hole, WebApp1, WebApp2, as per your `docker-compose.yml`.)

## Application Source Code (Submodules)

Your custom web applications are included as Git submodules at the root of this Ansible project (e.g., `webapp1_src/`).
* The `Dockerfile` for each web app must reside within its respective submodule's repository root.
* The Ansible `app_deployment` role will sync these submodule directories to `{{ project_target_path }}` on the Pi. The deployed `docker-compose.yml` (itself copied from `roles/app_deployment/files/`) will then use these synced directories as build contexts.

## Backup and Restore Strategy

* **Configuration & Code:** All Ansible configurations, the template `docker-compose.yml` (in `roles/app_deployment/files/`), helper scripts, and web app source code (via Git submodules) are version controlled in *this* Git repository. **Commit and push changes regularly!**
* **Persistent Application Data:**
    * Data is stored in host directories under `{{ project_target_path }}` on the Pi (e.g., `{{ project_target_path }}/pihole_config/`).
    * Use `{{ project_target_path }}/scripts/backup_data.sh` on the Pi. Customize this script (in the Ansible project's `scripts/` dir) for backup locations and data directories.
    * **Store backups off-Pi.**
    * Automate `backup_data.sh` using `cron` on the Pi (Ansible can configure this).
    * To restore:
        1.  Run Ansible playbook to set up the Pi.
        2.  Stop services on Pi: `cd {{ project_target_path }} && sudo docker compose down`.
        3.  Copy backup archives to Pi (e.g., `/tmp/restore/`).
        4.  Run restore script on Pi: `{{ project_target_path }}/scripts/restore_data.sh /tmp/restore/your_backup.tar.gz`.
        5.  Restart services: `cd {{ project_target_path }} && sudo docker compose up -d`, or re-run Ansible playbook.

## Updating Services

### Updating Pi-hole or Other Pre-built Images

1.  Update the image tag in `ansible_rpi_homelab/roles/app_deployment/files/docker-compose.yml`.
2.  Re-run `ansible-playbook playbook.yml --ask-become-pass`. (Ensure `app_deployment` role's Docker Compose task has `pull: true`).

### Updating Your Web Applications

1.  Push changes to the individual web app's Git repository.
2.  On your control node, in the `ansible_rpi_homelab` directory, update the submodule pointer:
    ```bash
    git submodule update --remote --merge webapp1_src # Or desired submodule
    git add webapp1_src
    git commit -m "Updated webapp1_src to latest"
    git push # Push change in Ansible project repo
    ```
3.  Re-run `ansible-playbook playbook.yml --ask-become-pass`.

## Troubleshooting with Ansible

* **Syntax Check Playbook:** `ansible-playbook playbook.yml --syntax-check`
* **Dry Run (Check Mode):** `ansible-playbook playbook.yml --check --ask-become-pass`
* **Increase Verbosity:** Add `-v`, `-vv`, `-vvv`, or `-vvvv` (e.g., `ansible-playbook playbook.yml --ask-become-pass -vvv`).
* **Limit to Specific Hosts or Tasks:** Use `--limit` or `--tags`.

## Troubleshooting Docker Services on the Pi

(Run these commands on the Raspberry Pi via SSH, typically in the `{{ project_target_path }}` directory, e.g., `/opt/rpi_homelab_services/`)

* **Check status:** `sudo docker compose ps`
* **View logs (follow):** `sudo docker compose logs -f`
* **View logs (specific service):** `sudo docker compose logs -f <service_name>`
* **Stop all:** `sudo docker compose down`
* **Start all:** `sudo docker compose up -d`
* **Build/Rebuild & start:** `sudo docker compose up -d --build`
* **Force recreate specific service:** `sudo docker compose up -d --build --force-recreate --remove-orphans <service_name>`
* **Access container shell:** `sudo docker exec -it <container_name_or_id> /bin/bash`
* **List images:** `sudo docker images`
* **List volumes:** `sudo docker volume ls`
* **Prune unused resources:** `sudo docker system prune -a --volumes` (Use with caution).