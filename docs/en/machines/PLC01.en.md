# PLC01 — OpenPLC Runtime

## 1. Overview

**PLC01** is the virtual machine responsible for executing the control logic of the simulated industrial process in **OT-Splunk-Lab**, using **OpenPLC Runtime** as the Programmable Logic Controller (PLC) execution environment.

The system runs **Debian 12** as its base operating system and uses **Docker Engine** to deploy OpenPLC Runtime inside a container, simplifying deployment, administration, and maintenance.

### System Architecture

```text
PLC01
└── Debian 12
    └── Docker Engine
        └── OpenPLC Runtime
```

OpenPLC Runtime enables the execution of industrial control programs and provides communication services through protocols such as **Modbus TCP**.

## 2. System Specifications

### 2.1. Virtual Machine Resources

| Parameter | Configuration |
|---|---|
| Hostname | `PLC01` |
| Operating System | Debian GNU/Linux 12 (bookworm) |
| CPU | 2 vCPU |
| RAM | 2 GB |
| Storage | 20 GB |

## 3. Network Configuration

### 3.1. IPv4 Addressing

| Parameter | Value |
|---|---|
| IPv4 Address | `192.168.50.140` |
| Subnet Mask | `255.255.255.0` |
| Prefix | `/24` |

### 3.2. Ports and Services

| Port | Protocol | Service | Purpose |
|---|---|---|---|
| 502 | TCP | Modbus TCP | Industrial communication service |
| 8443 | TCP | OpenPLC Runtime API | HTTPS access to the Runtime API |

Both ports are published from the Docker container to the host system.

## 4. Prerequisites

Before starting the deployment, ensure the following requirements are met:

- A virtual machine running Debian 12.
- A user account with `sudo` privileges.
- Network connectivity and Internet access.
- Hardware resources specified in Section 2.
- Access to the official Docker repository.
- Access to GitHub Container Registry.

## 5. Installation & Configuration

### 5.1. Operating System Preparation

Update the package repositories and upgrade installed packages:

```bash
sudo apt update
sudo apt full-upgrade -y
```

Install the required dependencies:

```bash
sudo apt install -y ca-certificates curl
```

### 5.2. Docker Engine Installation

#### 5.2.1. Configure the Official Docker Repository

Create the directory for storing repository signing keys:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download the official Docker GPG key:

```bash
curl -fsSL https://download.docker.com/linux/debian/gpg \
  | sudo tee /etc/apt/keyrings/docker.asc > /dev/null

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the official Docker repository for Debian 12:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: bookworm
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Update the package index:

```bash
sudo apt update
```

#### 5.2.2. Install Docker

Install Docker Engine and its components:

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Enable Docker and configure it to start automatically:

```bash
sudo systemctl enable --now docker
```

#### 5.2.3. Configure User Permissions

Add the current user to the `docker` group:

```bash
sudo usermod -aG docker $USER
```

Log out and log back in to apply the updated permissions.

> **Note:** membership in the `docker` group grants privileges equivalent to root access on the host system. This configuration is intended for a controlled laboratory environment.

#### 5.2.4. Verify Docker

Verify that Docker Engine is working correctly:

```bash
docker ps
```

### 5.3. OpenPLC Runtime Deployment

#### 5.3.1. Prepare the Working Directory

Create the service directory:

```bash
sudo mkdir -p /opt/openplc
sudo chown $USER:$USER /opt/openplc
cd /opt/openplc
```

#### 5.3.2. Configure Docker Compose

Create the configuration file:

```bash
nano compose.yml
```

Add the following configuration:

```yaml
services:
  openplc-runtime:
    image: ghcr.io/autonomy-logic/openplc-runtime:latest
    container_name: openplc-runtime
    restart: unless-stopped

    ports:
      - "8443:8443"
      - "502:502"

    cap_add:
      - SYS_NICE
      - SYS_RESOURCE

    volumes:
      - openplc-runtime-data:/var/run/runtime
      - ./plugins.conf:/workdir/plugins.conf

volumes:
  openplc-runtime-data:
```

This configuration provides:

- Automatic container restart using `unless-stopped`.
- TCP port 8443 exposure for the Runtime API.
- TCP port 502 exposure for Modbus TCP.
- Additional `SYS_NICE` and `SYS_RESOURCE` capabilities.
- Persistent storage through the `openplc-runtime-data` volume.
- Mounting of the local `plugins.conf` configuration file.

#### 5.3.3. Prepare the plugins.conf File

The `plugins.conf` file determines which plugins are enabled in OpenPLC Runtime. In this deployment, only **Modbus Slave** is enabled.

The functional configuration of the Modbus server, including variable mapping and parameters associated with the control program, is performed later through **OpenPLC Editor (ENG01)**.

Therefore, PLC01 only requires preparing the file needed for deployment, without manually configuring Modbus registers or variables.

Create the file:

```bash
nano plugins.conf
```

Add the following content:

```text
modbus_slave,./core/src/drivers/plugins/python/modbus_slave/simple_modbus.py,1,0,./core/src/drivers/plugins/python/modbus_slave/modbus_slave.json,./venvs/modbus_slave
modbus_master,./core/src/drivers/plugins/python/modbus_master/modbus_master_plugin.py,0,0,./core/src/drivers/plugins/python/modbus_master/modbus_master.json,./venvs/modbus_master
opcua,./core/src/drivers/plugins/python/opcua/plugin.py,0,0,./core/src/drivers/plugins/python/opcua/opcua.json,./venvs/opcua
s7comm,./build/plugins/libs7comm_plugin.so,0,1,./core/src/drivers/plugins/native/s7comm/s7comm_config.json
ethercat,./build/plugins/libethercat_plugin.so,0,1,./build/plugins/ethercat.json
```

This configuration enables the **Modbus Slave** plugin, while Modbus Master, OPC UA, S7comm, and EtherCAT remain disabled.

| Plugin | Status |
|---|---|
| Modbus Slave | Enabled |
| Modbus Master | Disabled |
| OPC UA | Disabled |
| S7comm | Disabled |
| EtherCAT | Disabled |

#### 5.3.4. Validate Docker Compose

Validate the configuration:

```bash
docker compose config
```

#### 5.3.5. Download the Image

Pull the OpenPLC Runtime image:

```bash
docker compose pull
```

#### 5.3.6. Start the Container

Deploy the service:

```bash
docker compose up -d
```

## 6. Service Management

The following commands are executed from `/opt/openplc`.

### Start OpenPLC Runtime

```bash
cd /opt/openplc
docker compose up -d
```

### Stop OpenPLC Runtime

```bash
docker compose down
```

### Restart OpenPLC Runtime

```bash
docker compose restart
```

### Check Service Status

```bash
docker compose ps
```

### View Service Logs

```bash
docker logs -f openplc-runtime
```

## 7. Validation

### 7.1. Check Container Status

Run:

```bash
cd /opt/openplc
docker compose ps
```

The container should be running and display `healthy` when its health check is successful.

Status observed during validation:

```text
NAME              IMAGE                                           COMMAND                  SERVICE           CREATED          STATUS                    PORTS
openplc-runtime   ghcr.io/autonomy-logic/openplc-runtime:latest   "bash ./start_openpl…"   openplc-runtime   12 minutes ago   Up 12 minutes (healthy)   0.0.0.0:502->502/tcp, [::]:502->502/tcp, 0.0.0.0:8443->8443/tcp, [::]:8443->8443/tcp
```

### 7.2. Verify Published Ports

Run:

```bash
ss -lnt | grep -E "502|8443"
```

TCP ports `502` and `8443` should appear in the listening state.

### 7.3. Verify the Modbus Server

Inspect the container logs:

```bash
docker logs openplc-runtime 2>&1 | grep -i "MODBUS_SLAVE"
```

During validation, the Modbus TCP server initialization was confirmed through log messages equivalent to:

```text
[INFO] [MODBUS_SLAVE] Configuration loaded - Host: 0.0.0.0, Port: 502
[INFO] [MODBUS_SLAVE] Server listening on 0.0.0.0:502
[INFO] Plugin modbus_slave started successfully
```

These messages confirm that the Modbus Slave plugin was started and was listening on the configured port during the recorded execution.

### 7.4. Verify the Runtime API

Run:

```bash
curl -k https://localhost:8443/api/version
```

This command checks access to the OpenPLC Runtime version endpoint.

The `-k` option disables TLS certificate validation and is used only for local laboratory testing.

**Observed result:**

```json
{"version":"v4.3.1"}
```

The response confirms that the Runtime API is accessible and successfully returns the installed version.

## 8. References

- [Docker Engine — Debian](https://docs.docker.com/engine/install/debian/)
- [Docker Compose](https://docs.docker.com/compose/)
- [OpenPLC](https://autonomylogic.com/)
- [OpenPLC Runtime — Container Registry](https://github.com/autonomy-logic/openplc-runtime/pkgs/container/openplc-runtime)