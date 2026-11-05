# PLC01 — OpenPLC Runtime

## 1. Descripción general

**PLC01** es la máquina virtual encargada de ejecutar la lógica de control del proceso industrial simulado en **OT-Lab**, utilizando **OpenPLC Runtime** como entorno de ejecución del controlador lógico programable (PLC).

El sistema utiliza **Debian 12** como sistema operativo base y **Docker Engine** para ejecutar OpenPLC Runtime dentro de un contenedor, facilitando su despliegue, administración y mantenimiento.

### Arquitectura del sistema

```text
PLC01
└── Debian 12
    └── Docker Engine
        └── OpenPLC Runtime
```

OpenPLC Runtime permite ejecutar programas de control industrial y proporcionar servicios de comunicación mediante protocolos como **Modbus TCP**.

## 2. Características del sistema

### 2.1. Recursos de la máquina virtual

| Parámetro | Configuración |
|---|---|
| Hostname | `PLC01` |
| Sistema operativo | Debian GNU/Linux 12 (bookworm) |
| CPU | 2 vCPU |
| RAM | 2 GB |
| Almacenamiento | 20 GB |

## 3. Configuración de red

### 3.1. Direccionamiento IPv4

| Parámetro | Valor |
|---|---|
| Dirección IPv4 | `192.168.50.140` |
| Máscara | `255.255.255.0` |
| Prefijo | `/24` |

### 3.2. Puertos y servicios

| Puerto | Protocolo | Servicio | Función |
|---|---|---|---|
| 502 | TCP | Modbus TCP | Servicio de comunicación industrial |
| 8443 | TCP | OpenPLC Runtime API | Acceso HTTPS a la API del Runtime |

Los puertos están publicados desde el contenedor Docker hacia el sistema anfitrión.

## 4. Requisitos previos

Antes de iniciar el despliegue se requiere:

- Una máquina virtual con Debian 12 instalado.
- Un usuario con permisos `sudo`.
- Conectividad de red y acceso a Internet.
- Recursos de hardware especificados en el apartado 2.
- Acceso al repositorio oficial de Docker.
- Acceso a GitHub Container Registry.

## 5. Instalación y configuración

### 5.1. Preparación del sistema operativo

Actualizar los repositorios y paquetes del sistema:

```bash
sudo apt update
sudo apt full-upgrade -y
```

Instalar las dependencias necesarias:

```bash
sudo apt install -y ca-certificates curl
```

### 5.2. Instalación de Docker Engine

#### 5.2.1. Configurar el repositorio oficial de Docker

Crear el directorio para almacenar las claves de los repositorios:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Descargar la clave GPG oficial de Docker:

```bash
curl -fsSL https://download.docker.com/linux/debian/gpg \
  | sudo tee /etc/apt/keyrings/docker.asc > /dev/null

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Añadir el repositorio oficial para Debian 12:

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

Actualizar el índice de paquetes:

```bash
sudo apt update
```

#### 5.2.2. Instalar Docker

Instalar Docker Engine y sus componentes:

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Habilitar Docker y configurar su inicio automático:

```bash
sudo systemctl enable --now docker
```

#### 5.2.3. Configurar permisos de usuario

Añadir el usuario actual al grupo `docker`:

```bash
sudo usermod -aG docker $USER
```

Cerrar la sesión y volver a iniciarla para aplicar los permisos.

> **Nota:** el grupo `docker` concede privilegios equivalentes a root sobre el sistema anfitrión. Esta configuración está destinada a un entorno de laboratorio controlado.

#### 5.2.4. Verificar Docker

Comprobar que Docker Engine funciona correctamente:

```bash
docker ps
```


### 5.3. Despliegue de OpenPLC Runtime

#### 5.3.1. Preparar el directorio de trabajo

Crear el directorio destinado al servicio:

```bash
sudo mkdir -p /opt/openplc
sudo chown $USER:$USER /opt/openplc
cd /opt/openplc
```

#### 5.3.2. Configurar Docker Compose

Crear el archivo:

```bash
nano compose.yml
```

Añadir la siguiente configuración:

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

Esta configuración establece:

- Reinicio automático del contenedor mediante `unless-stopped`.
- Publicación del puerto TCP 8443 para la API del Runtime.
- Publicación del puerto TCP 502 para Modbus TCP.
- Capacidades adicionales `SYS_NICE` y `SYS_RESOURCE`.
- Persistencia mediante el volumen `openplc-runtime-data`.
- Montaje del archivo local `plugins.conf`.

#### 5.3.3. Preparar el archivo plugins.conf

El archivo `plugins.conf` determina qué plugins están habilitados en OpenPLC Runtime. En este despliegue se habilita únicamente **Modbus Slave**.

La configuración funcional del servidor Modbus, incluido el mapeo de variables y los parámetros asociados al programa de control, se realiza posteriormente desde **OpenPLC Editor (ENG01)**.

Por tanto, en PLC01 únicamente se prepara el archivo necesario para el despliegue, sin configurar manualmente los registros ni las variables Modbus.

Crear el archivo:

```bash
nano plugins.conf
```

Añadir el siguiente contenido:

```text
modbus_slave,./core/src/drivers/plugins/python/modbus_slave/simple_modbus.py,1,0,./core/src/drivers/plugins/python/modbus_slave/modbus_slave.json,./venvs/modbus_slave
modbus_master,./core/src/drivers/plugins/python/modbus_master/modbus_master_plugin.py,0,0,./core/src/drivers/plugins/python/modbus_master/modbus_master.json,./venvs/modbus_master
opcua,./core/src/drivers/plugins/python/opcua/plugin.py,0,0,./core/src/drivers/plugins/python/opcua/opcua.json,./venvs/opcua
s7comm,./build/plugins/libs7comm_plugin.so,0,1,./core/src/drivers/plugins/native/s7comm/s7comm_config.json
ethercat,./build/plugins/libethercat_plugin.so,0,1,./build/plugins/ethercat.json
```

La configuración habilita el plugin **Modbus Slave**, mientras que Modbus Master, OPC UA, S7comm y EtherCAT permanecen deshabilitados.

| Plugin | Estado |
|---|---|
| Modbus Slave | Habilitado |
| Modbus Master | Deshabilitado |
| OPC UA | Deshabilitado |
| S7comm | Deshabilitado |
| EtherCAT | Deshabilitado |

#### 5.3.4. Validar Docker Compose

Comprobar la configuración:

```bash
docker compose config
```

#### 5.3.5. Descargar la imagen

Descargar OpenPLC Runtime:

```bash
docker compose pull
```

#### 5.3.6. Iniciar el contenedor

Desplegar el servicio:

```bash
docker compose up -d
```

## 6. Gestión del servicio

Los siguientes comandos se ejecutan desde `/opt/openplc`.

### Iniciar OpenPLC Runtime

```bash
cd /opt/openplc
docker compose up -d
```

### Detener OpenPLC Runtime

```bash
docker compose down
```

### Reiniciar OpenPLC Runtime

```bash
docker compose restart
```

### Consultar el estado

```bash
docker compose ps
```

### Consultar los registros

```bash
docker logs -f openplc-runtime
```

## 7. Validación

### 7.1. Comprobar el estado del contenedor

Ejecutar:

```bash
cd /opt/openplc
docker compose ps
```

El contenedor debe encontrarse en estado `Up` y mostrar `healthy` cuando su comprobación de salud sea satisfactoria.

Estado observado durante la documentación:

```text
NAME              IMAGE                                           COMMAND                  SERVICE           CREATED          STATUS                    PORTS
openplc-runtime   ghcr.io/autonomy-logic/openplc-runtime:latest   "bash ./start_openpl…"   openplc-runtime   12 minutes ago   Up 12 minutes (healthy)   0.0.0.0:502->502/tcp, [::]:502->502/tcp, 0.0.0.0:8443->8443/tcp, [::]:8443->8443/tcp

```

### 7.2. Verificar los puertos publicados

Ejecutar:

```bash
ss -lnt | grep -E "502|8443"
```

Los puertos TCP `502` y `8443` deben aparecer en estado de escucha.

### 7.3. Verificar el servidor Modbus

Consultar los registros:

```bash
docker logs openplc-runtime 2>&1 | grep -i "MODBUS_SLAVE"
```

Durante la validación se confirmó la inicialización del servidor Modbus TCP mediante mensajes equivalentes a:

```text
[INFO] [MODBUS_SLAVE] Configuration loaded - Host: 0.0.0.0, Port: 502
[INFO] [MODBUS_SLAVE] Server listening on 0.0.0.0:502
[INFO] Plugin modbus_slave started successfully
```

Estas evidencias confirman que el plugin Modbus Slave fue iniciado y quedó escuchando en el puerto configurado durante la ejecución registrada.

### 7.4. Comprobar la API del Runtime

Ejecutar:

```bash
curl -k https://localhost:8443/api/version
```

El comando permite comprobar el acceso al endpoint de versión de OpenPLC Runtime.

```bash
{"version":"v4.3.1"}
```

La opción `-k` deshabilita la validación del certificado TLS y se utiliza únicamente para pruebas locales del laboratorio.

Resultado obtenido:

```json
{"version":"v4.3.1"}
```

La respuesta confirma que la API del Runtime está accesible y devuelve correctamente la versión instalada.

## 8. Referencias

- [Docker Engine — Debian](https://docs.docker.com/engine/install/debian/)
- [Docker Compose](https://docs.docker.com/compose/)
- [OpenPLC](https://autonomylogic.com/)
- [OpenPLC Runtime — Container Registry](https://github.com/autonomy-logic/openplc-runtime/pkgs/container/openplc-runtime)
