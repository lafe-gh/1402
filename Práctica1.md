# Gestión de Google Cloud Compute Engine con Ansible

---


## 1. Descripción del Escenario

En esta práctica construiremos una infraestructura automatizada en Google Cloud Platform utilizando Ansible como herramienta de gestión de configuración. El escenario es el siguiente:

- **Nodo de control:** Una instancia de Compute Engine con Debian que ya existe en nuestro proyecto de GCP. Desde esta máquina ejecutaremos todos los playbooks de Ansible.
- **Nodo gestionado:** Una segunda instancia de Compute Engine (Debian 12) que crearemos mediante Ansible a través de la API de GCP, y que posteriormente configuraremos de forma remota vía SSH.

El flujo completo de la práctica abarca desde la preparación del entorno Python y la autenticación con la API de Google Cloud, hasta el despliegue de un contenedor Docker Nginx en el nodo gestionado — todo orquestado con playbooks de Ansible.

### Arquitectura de la Práctica


<img width="562" height="399" alt="image" src="https://github.com/user-attachments/assets/41b88222-6db2-4df5-97a2-e5f69983ad17" />


---

## 2. Preparación del Nodo de Control

El nodo de control es la máquina desde la cual Ansible ejecuta los playbooks. Necesitamos un entorno Python aislado para evitar conflictos de dependencias con el sistema operativo.

### 2.1 Crear un Entorno Virtual de Python

Es una buena práctica instalar Ansible dentro de un **entorno virtual (venv)** dedicado. Esto aísla las dependencias del proyecto y evita conflictos con los paquetes del sistema:

```bash
# Instalar el paquete de venv si no está disponible
sudo apt install python3.11-venv

# Crear el entorno virtual dedicado para Ansible
python3 -m venv ~/ansible-venv

# Activar el entorno virtual
source ~/ansible-venv/bin/activate

# Instalar ansible-core junto con las dependencias para GCP
pip install ansible-core requests google-auth

# Verificar que Ansible se instaló correctamente
ansible --version
```

> **Nota:** Para no tener que recordar la ruta del entorno virtual cada vez, podemos crear un alias en nuestro `.bashrc`:

```bash
echo 'alias ansenv="source ~/ansible-venv/bin/activate"' >> ~/.bashrc
source ~/.bashrc
```

A partir de ahora, basta con escribir `ansenv` para activar el entorno, y `deactivate` para salir de él.

### 2.2 Verificar los Prerrequisitos

Antes de continuar, debemos asegurarnos de que las herramientas básicas están disponibles:

```bash
# Verificar la versión de Ansible (se requiere 2.16+)
ansible --version

# Verificar Python 3.10+ (requisito de la colección google.cloud)
python3 --version

# Verificar que pip está disponible
pip --version
```

---

## 3. Instalación de la Colección `google.cloud` y Dependencias

La colección `google.cloud` de Ansible Galaxy proporciona todos los módulos necesarios para interactuar con los servicios de GCP (Compute Engine, VPC, firewalls, etc.). Estos módulos se comunican con la API de Google Cloud, por lo que requieren bibliotecas Python específicas.

### 3.1 Instalar las Dependencias de Python

Los módulos de GCP dependen de las bibliotecas `requests` (para llamadas HTTP) y `google-auth` (para la autenticación OAuth2 con la API de Google):

```bash
pip install requests google-auth --user
```

### 3.2 Instalar la Colección desde Ansible Galaxy

Ansible Galaxy es el repositorio oficial de colecciones y roles de la comunidad. Instalamos la colección de Google Cloud con:

```bash
ansible-galaxy collection install google.cloud
```

Verificamos que la instalación ha tenido éxito:

```bash
ansible-galaxy collection list | grep google.cloud
```

La salida debe ser similar a:

```
google.cloud   1.11.0
```

---

## 4. Configuración de la Cuenta de Servicio en GCP (Autenticación API)

Ansible necesita autenticarse contra la API de Google Cloud para crear y gestionar recursos. Para ello se utiliza una **Cuenta de Servicio (Service Account)**, que es una identidad no humana con permisos específicos.

> **Importante:** Esta autenticación es para la API de GCP (crear VMs, redes, etc.). Es completamente independiente de la autenticación SSH que usaremos más adelante para conectarnos a las máquinas.

### 4.1 Crear la Cuenta de Servicio

Ejecutamos los siguientes comandos desde el nodo de control (donde ya tenemos `gcloud` autenticado):

```bash
# Obtener el ID del proyecto activo
export PROJECT_ID=$(gcloud config get-value project)

# Crear la cuenta de servicio
gcloud iam service-accounts create ansible-sa \
  --display-name "Cuenta de Servicio para Automatización con Ansible"

# Asignar los roles necesarios
for role in \
  roles/compute.instanceAdmin.v1 \
  roles/compute.networkAdmin \
  roles/compute.securityAdmin \
  roles/iam.serviceAccountUser; do

  gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:ansible-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="$role"
done
```

### Explicación de los Roles Asignados

Cada rol otorga un conjunto específico de permisos siguiendo el principio de mínimo privilegio:

| Rol | Propósito |
|---|---|
| `compute.instanceAdmin.v1` | Crear, modificar y eliminar instancias de VM |
| `compute.networkAdmin` | Gestionar redes VPC, subredes y reglas de firewall |
| `compute.securityAdmin` | Administrar reglas de firewall y certificados SSL |
| `iam.serviceAccountUser` | Permitir que la cuenta de servicio actúe en nombre de otras cuentas de servicio asociadas a las VMs |

### 4.2 Verificar la Configuración

Es recomendable confirmar que la cuenta de servicio se creó correctamente y tiene los roles asignados:

```bash
# Listar todas las cuentas de servicio del proyecto
gcloud iam service-accounts list

# Verificar los roles asignados a nuestra cuenta de servicio
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:ansible-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --format="table(bindings.role)"
```

### 4.3 Generar y Descargar la Clave de la Cuenta de Servicio

Ansible necesita un archivo JSON con las credenciales de la cuenta de servicio para autenticarse programáticamente:

```bash
# Generar la clave y guardarla localmente
gcloud iam service-accounts keys create ~/ansible-sa-key.json \
  --iam-account="ansible-sa@${PROJECT_ID}.iam.gserviceaccount.com"

# Restringir los permisos del archivo (solo lectura para el propietario)
chmod 600 ~/ansible-sa-key.json
```

> **⚠️ Buena Práctica de Seguridad:** Este archivo JSON contiene credenciales sensibles. **Nunca** lo incluyas en un repositorio de control de versiones (Git). Considera cifrar el archivo con `ansible-vault`:


---

## 5. Configuración de Claves SSH

SSH es el protocolo de transporte que Ansible utiliza para conectarse a los nodos gestionados y ejecutar tareas en ellos. Dado que ambas máquinas están en la misma VPC por defecto de GCP, la comunicación es interna.

### 5.1 Generar un Par de Claves SSH Ed25519

El algoritmo **Ed25519** es el recomendado actualmente: es más rápido, más seguro y genera claves más cortas que RSA:

```bash
ssh-keygen -t ed25519 \
  -f ~/.ssh/ansible-ed25519 \
  -C "ansible@nodo-de-control" \
  -a 100
```

### Desglose de los Parámetros

| Parámetro | Función |
|---|---|
| `-t ed25519` | Algoritmo de curva elíptica Ed25519 |
| `-f ~/.ssh/ansible-ed25519` | Archivo dedicado para la clave (evita sobrescribir claves por defecto) |
| `-C "ansible@nodo-de-control"` | Comentario descriptivo para identificar la clave |
| `-a 100` | 100 rondas de KDF (Key Derivation Function) para mayor resistencia a fuerza bruta |

### 5.2 Establecer Permisos Correctos

Los permisos de archivos SSH deben ser estrictos; de lo contrario, el cliente SSH rechazará las claves:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/ansible-ed25519       # Clave privada: solo propietario
chmod 644 ~/.ssh/ansible-ed25519.pub   # Clave pública: lectura para todos
```

---

## 6. Creación de la Estructura del Proyecto y Aprovisionamiento de una VM

### 6.1 Crear la Estructura de Directorios

Organizamos el proyecto siguiendo las convenciones recomendadas por Ansible:

```bash
mkdir -p ansible-gcp/{playbooks,inventory,group_vars}
cd ansible-gcp
```

La estructura resultante es:

```
ansible-gcp/
├── playbooks/       # Playbooks de Ansible (las "recetas" de automatización)
├── inventory/       # Archivos de inventario (lista de máquinas a gestionar)
└── group_vars/      # Variables compartidas entre grupos de hosts
```

### 6.2 Mover la Clave de la Cuenta de Servicio al Proyecto

```bash
mv ~/ansible-sa-key.json group_vars/
chmod 600 group_vars/ansible-sa-key.json
```

### 6.3 Definir las Variables Globales

Creamos el archivo de variables que será compartido por todos los playbooks:

```bash
nano group_vars/all.yml
```

Contenido de `all.yml`:

```yaml
# Variables globales del proyecto Ansible-GCP
gcp_project: YOUR_PROJECT_ID
gcp_region: us-central1
gcp_zone: us-central1-a
gcp_credentials_file: "{{ playbook_dir }}/../group_vars/ansible-sa-key.json"
```

> **Nota:** La variable `playbook_dir` es una variable mágica de Ansible que apunta al directorio donde se encuentra el playbook en ejecución. Esto nos permite usar rutas relativas portables.

### 6.4 Crear el Inventario Inicial

El inventario define qué máquinas gestiona Ansible. Inicialmente, solo incluimos `localhost` (el propio nodo de control), ya que la VM remota aún no existe:

```bash
nano inventory/gcp.yml
```

```yaml
# Inventario de infraestructura GCP
all:
  vars:
    gcp_project: YOUR_PROJECT_ID
    gcp_region: us-central1
    gcp_zone: us-central1-a
    gcp_credentials_file: "{{ playbook_dir }}/../group_vars/ansible-sa-key.json"

  hosts:
    localhost:
      ansible_connection: local
```

### 6.5 Crear el Playbook de Provisión de la nueva VM

Este playbook utiliza el módulo `gcp_compute_instance` de la colección `google.cloud` para crear una instancia de Compute Engine a través de la API de GCP:

```bash
nano playbooks/create_vm.yml
```

```yaml
# Playbook: Crear una instancia de Compute Engine (capa gratuita)
- name: Crear instancia GCE de capa gratuita
  hosts: localhost
  gather_facts: false

  collections:
    - google.cloud

  vars:
    instance_name: ans-test1          # Nombre de la instancia
    machine_type: e2-micro            # Tipo de máquina (free-tier)
    disk_size_gb: 10                  # Tamaño del disco en GB

  tasks:
    - name: Crear instancia VM con Debian 12
      gcp_compute_instance:
        name: "{{ instance_name }}"
        project: "{{ gcp_project }}"
        zone: "{{ gcp_zone }}"
        machine_type: "{{ machine_type }}"

        disks:
          - auto_delete: true
            boot: true
            initialize_params:
              source_image: projects/debian-cloud/global/images/family/debian-12
              disk_size_gb: "{{ disk_size_gb }}"

        network_interfaces:
          - network:
              name: default
            access_configs:
              - name: External NAT
                type: ONE_TO_ONE_NAT

        auth_kind: serviceaccount
        service_account_file: "{{ gcp_credentials_file }}"
        state: present
```

### Explicación de los Campos Clave

- **`machine_type: e2-micro`** — Tipo de máquina elegible para free-tier de GCP (2 vCPUs compartidas, 1 GB RAM).
- **`source_image`** — Imagen base del disco; usamos la familia `debian-12` para obtener siempre la versión más reciente.
- **`access_configs`** — Configura una IP pública efímera (`ONE_TO_ONE_NAT`) para acceso externo.
- **`auth_kind: serviceaccount`** — Indica que la autenticación se realiza con una cuenta de servicio.
- **`state: present`** — Asegura que la instancia existe; si ya existe, no se modifica (idempotencia).

### 6.6 Ejecutar el Playbook

```bash
ansible-playbook \
  -i inventory/gcp.yml \
  playbooks/create_vm.yml
```

Si todo funciona correctamente, Ansible creará la instancia `ans-test1` en la zona `us-central1-a`.

---

## 7. Inyección de Claves SSH para Gestionar el Nodo Remoto

Una vez creada la VM, necesitamos inyectar nuestra clave pública SSH en ella para que Ansible pueda conectarse vía SSH.

### 7.1 Actualizar el Inventario con Datos SSH

```bash
nano inventory/gcp.yml
```

Añadimos las variables SSH al inventario:

```yaml
# Inventario de infraestructura GCP (con variables SSH)
all:
  vars:
    gcp_project: YOUR_PROJECT_ID
    gcp_region: us-central1
    gcp_zone: us-central1-a
    gcp_credentials_file: "{{ playbook_dir }}/../group_vars/ansible-sa-key.json"

    ssh_user: ansible
    ssh_public_key_path: "{{ lookup('env','HOME') }}/.ssh/ansible-ed25519"

  hosts:
    localhost:
      ansible_connection: local
```

### 7.2 Inyectar la Clave Pública en la VM

Usamos `gcloud` para añadir nuestra clave pública SSH a los metadatos de la instancia. GCP distribuye automáticamente las claves de los metadatos al sistema operativo de la VM:

```bash
# Verificar el estado actual de la instancia
gcloud compute instances describe ans-test1 --zone us-central1-a

# Inyectar la clave pública SSH en los metadatos
gcloud compute instances add-metadata ans-test1 \
  --zone us-central1-a \
  --metadata ssh-keys="ansible:$(cat ~/.ssh/ansible-ed25519.pub)"

# Confirmar que la clave se añadió correctamente
gcloud compute instances describe ans-test1 \
  --zone us-central1-a \
  --format="get(metadata.items)"
```

### 7.3 Probar la Conexión SSH Manualmente

Antes de usar Ansible, verificamos que podemos conectarnos directamente:

```bash
ssh -i ~/.ssh/ansible-ed25519 ansible@<ip-pública>
```

> **Nota:** Sustituye <ip-pública> por la IP pública real de la nueva instancia, que puedes obtener con:
> ```bash
> gcloud compute instances describe ans-test1 --zone us-central1-a --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
> ```

---

## 8. Instalación de Docker en la Máquina Remota

Ahora que tenemos conectividad SSH, podemos usar Ansible para configurar el nodo remoto. El primer paso es instalar Docker.

### 8.1 Añadir el Nodo Remoto al Inventario

Actualizamos el inventario para incluir la VM recién creada como host gestionable.
También eliminaremos el localhost para prevenir cualquier actuación sobre el nodo de control

```bash
nano inventory/gcp.yml
```

Archivo de inventario modificado completo:

```yaml
# Inventario completo de infraestructura GCP
all:
  vars:
    gcp_project: YOUR_PROJECT_ID
    gcp_region: us-central1
    gcp_zone: us-central1-a
    gcp_credentials_file: "{{ playbook_dir }}/../group_vars/ansible-sa-key.json"

    ssh_user: ansible
    ssh_public_key_path: "{{ lookup('env','HOME') }}/.ssh/ansible-ed25519"

  hosts:
    ans-test1:
      ansible_host: 34.135.170.17
      ansible_user: ansible
      ansible_ssh_private_key_file: ~/.ssh/ansible-ed25519
```

### 8.2 Verificar la Conectividad con Ansible

Usamos el módulo `ping` de Ansible para confirmar que la comunicación funciona:

```bash
ansible -i inventory/gcp.yml ans-test1 -m ping
```

La respuesta esperada es:

```
ans-test1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### 8.3 Crear el Playbook de Instalación de Docker

```bash
nano playbooks/install_docker.yml
```

```yaml
# Playbook: Instalar Docker en la VM remota
- name: Instalar Docker en la VM de prueba
  hosts: ans-test1
  become: yes          # Ejecutar con privilegios de superusuario (sudo)
  gather_facts: yes    # Recopilar datos del sistema antes de ejecutar

  tasks:

    - name: Actualizar la caché de paquetes APT
      apt:
        update_cache: yes

    - name: Instalar Docker y sus dependencias
      apt:
        name:
          - docker.io
          - docker-compose
        state: present

    - name: Asegurar que el servicio Docker está activo y habilitado al inicio
      service:
        name: docker
        state: started
        enabled: yes

    - name: Añadir el usuario ansible al grupo docker
      user:
        name: ansible
        groups: docker
        append: yes
```

### Explicación de las Tareas

- **`become: yes`** — Ansible ejecuta los comandos con `sudo`, necesario para instalar paquetes y modificar servicios del sistema.
- **`update_cache`** — Equivale a ejecutar `apt update`; asegura que la lista de paquetes esté actualizada.
- **`state: present`** — Garantiza que los paquetes están instalados. Si ya lo están, no se reinstalan (idempotencia).
- **`enabled: yes`** — Configura Docker para arrancar automáticamente cuando la máquina se reinicie.
- **`append: yes`** — Añade el usuario al grupo `docker` sin eliminarlo de sus otros grupos.

### 8.4 Ejecutar el Playbook

```bash
ansible-playbook -i inventory/gcp.yml playbooks/install_docker.yml
```

### 8.5 Verificación Manual (Opcional)

Podemos conectarnos por SSH y comprobar la instalación:

```bash
ssh -i ~/.ssh/ansible-ed25519 ansible@<ip_pública>

# Dentro de la VM remota:
docker --version
docker-compose --version
docker ps
```
Y adicionalmente, en preparación de la práctica 2 (autenticar docker con el Artifact Registry):
```bash
gcloud auth configure-docker us-central1-docker.pkg.dev

exit
```
---

## 9. Despliegue de un Contenedor en la Máquina Remota

Como paso final, desplegamos un contenedor Nginx utilizando la colección `community.docker` de Ansible.

### 9.1 Instalar las Dependencias Necesarias

En el **nodo de control**, instalamos la colección de Docker para Ansible y la biblioteca Python correspondiente:

```bash
# Instalar la colección de Docker para Ansible
ansible-galaxy collection install community.docker

# Instalar la biblioteca Python de Docker (necesaria en el nodo de control)
pip install docker
```

### 9.2 Crear el Playbook de Despliegue del Contenedor

```bash
nano playbooks/run_nginx.yml
```

```yaml
# Playbook: Desplegar contenedor Nginx en la VM remota
- name: Ejecutar contenedor Nginx en la VM remota
  hosts: ans-test1
  become: yes
  gather_facts: yes

  collections:
    - community.docker

  tasks:

    - name: Asegurar que el contenedor Nginx está en ejecución
      docker_container:
        name: demo-nginx              # Nombre del contenedor
        image: nginx:latest            # Imagen a utilizar
        state: started                 # Estado deseado: en ejecución
        restart_policy: unless-stopped # Reiniciar salvo parada manual
        ports:
          - "80:80"                    # Mapear puerto 80 del host al contenedor
```

### Explicación del Módulo `docker_container`

- **`name: demo-nginx`** — Nombre identificativo del contenedor; permite a Ansible rastrear su estado.
- **`image: nginx:latest`** — Docker descargará la imagen oficial de Nginx si no está disponible localmente.
- **`restart_policy: unless-stopped`** — El contenedor se reiniciará automáticamente salvo que se detenga manualmente con `docker stop`.
- **`ports: "80:80"`** — Expone el puerto 80 del contenedor en el puerto 80 del host, permitiendo acceso HTTP externo.

### 9.3 Ejecutar el Playbook

```bash
ansible-playbook -i inventory/gcp.yml playbooks/run_nginx.yml
```

Una vez ejecutado con éxito, el servidor Nginx estará disponible en `http://<IP_PÚBLICA_DE_LA_VM>:80`.

> **Nota:** Para acceder desde el exterior, asegúrate de que existe una regla de firewall en GCP que permita tráfico TCP en el puerto 80. La red VPC por defecto suele incluir la regla `default-allow-http`, pero puedes verificarlo con:
> ```bash
> gcloud compute firewall-rules list --filter="allowed[].ports:80"
> ```

---

## Resumen de la Estructura Final del Proyecto

```
ansible-gcp/
├── group_vars/
│   ├── all.yml                  # Variables globales (proyecto, región, zona)
│   └── ansible-sa-key.json     # Credenciales de la cuenta de servicio (¡SECRETO!)
├── inventory/
│   └── gcp.yml                  # Inventario de hosts (localhost + ans-test1)
└── playbooks/
    ├── create_vm.yml            # Aprovisionamiento de la VM en GCP
    ├── install_docker.yml       # Instalación de Docker en la VM remota
    └── run_nginx.yml            # Despliegue del contenedor Nginx
```
