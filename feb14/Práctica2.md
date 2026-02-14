# Pipeline CI/CD con GitHub Actions, Ansible y GCP

## Despliegue Automatizado del Crypto Chatbot (Frontend + Backend)

---

## 1. Arquitectura General

### Flujo del Pipeline

```
Developer Push (main)
        ↓
GitHub Actions
  ├─ Build imagen backend (Python)
  ├─ Build imagen frontend (Nginx)
  ├─ Push ambas imágenes → Artifact Registry
  └─ SSH → Control Node
              ↓
        Control Node (GCP VM) 
          └─ ansible-playbook deploy.yml
                    ↓
              VM de Despliegue
                ├─ agent-backend  (puerto 80:80)
                └─ agent-frontend (puerto 80:80)
```


### Componentes Involucrados

| Componente | Rol | Ubicación |
|---|---|---|
| GitHub Actions | CI — build y push + SSH al control | Cloud (GitHub) | 
| Artifact Registry | Registro de imágenes Docker | GCP (`us-central1`) |
| Control Node | Ejecuta Ansible | GCP VM |
| VM de Despliegue | Hospeda los contenedores | GCP VM |

---

## 2. Prerrequisitos

Antes de comenzar, asegúrate de tener lo siguiente:

- Repositorio en GitHub con el código del chatbot (frontend + backend).
- Control Node en GCP ya configurado en la Práctica 1.
- La VM de despliegue corre ningún otro contenedor (stop el contenedor NGINX).

Configurar las variables de entorno:

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
```

---

##  Paso 1 — Service Account de CI en GCP

Creamos una service account dedicada exclusivamente para GitHub Actions. Esta cuenta se usará para autenticarse contra GCP y subir las imágenes Docker al Artifact Registry.

### Crear la Service Account

```bash
gcloud iam service-accounts create github-ci \
  --display-name="GitHub Actions CI"
```

### Asignar Rol de Escritura en Artifact Registry

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-ci@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"
```

### Generar Clave JSON

```bash
gcloud iam service-accounts keys create github-ci-key.json \
  --iam-account=github-ci@${PROJECT_ID}.iam.gserviceaccount.com
```

> ⚠️ **Importante:** Guarda este archivo JSON !. Lo necesitarás para configurar los secretos de GitHub. No lo subas al repositorio.

---

## Paso 2 — Artifact Registry

Crear el repositorio Docker en Artifact Registry (una sola vez):

```bash
gcloud artifacts repositories create agent-repo \
  --repository-format=docker \
  --location=$REGION \
  --description="Repositorio de imágenes del Crypto Chatbot"
```

La ruta del registro será:

```
us-central1-docker.pkg.dev/PROJECT_ID/agent-repo
```

Verificar que se ha creado correctamente:

```bash
gcloud artifacts repositories list --location=$REGION
```
Lo hacemos público (para no tener que crear service accounts en los nodos clientes, en esta práctica, en producción no)

```bash
gcloud artifacts repositories add-iam-policy-binding agent-repo \
  --location=us-central1 \
  --member="allUsers" \
  --role="roles/artifactregistry.reader"
```

---



### Configurar Docker en el Control Node para Autenticarse

Conéctate al Control Node por SSH y ejecuta:

```bash
gcloud auth configure-docker us-central1-docker.pkg.dev
```

Esto configura Docker para usar `gcloud` como credential helper al hacer pull desde Artifact Registry.





## Paso 3 — Acceso SSH desde GitHub Actions al Control Node

Usamos el modelo de **trigger por SSH**: GitHub Actions, tras construir y subir las imágenes, se conecta por SSH al Control Node para ejecutar Ansible.

### Generar Par de Claves en el Control Node

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github-deploy -C "github-actions-deploy"
```

Esto genera:

- `~/.ssh/github-deploy` — clave privada (irá a GitHub Secrets)
- `~/.ssh/github-deploy.pub` — clave pública (se queda en el Control Node)

### Autorizar la Clave Pública

En el Control Node, añadir la clave pública al archivo de claves autorizadas:

```bash
cat ~/.ssh/github-deploy.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

---

## Paso 4 — Secretos en GitHub

Dentro de nuestro repositorio en GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

Configuramos los siguientes secretos:

| Secreto | Descripción | Valor |
|---|---|---|
| `GCP_SA_KEY` | JSON completo de `github-ci-key.json` | Contenido del archivo JSON |
| `PROJECT_ID` | ID del proyecto de GCP | `tu-project-id` |
| `GOOGLE_API_KEY` | Clave de API de Google (Gemini) | `AIza...` |
| `CONTROL_HOST` | IP pública o hostname del Control Node | `34.xxx.xxx.xxx` |
| `CONTROL_HOST_USER` | Usuario SSH del Control Node | `ansible` o tu usuario |
| `CONTROL_HOST_SSH_KEY` | Clave privada SSH (`github-deploy`) | Contenido de `~/.ssh/github-deploy` |
| `CONTROL_HOST_PORT` | Puerto SSH del Control Node | `22` |

> ⚠️ Para `CONTROL_HOST_SSH_KEY`, pegad el contenido completo de la clave privada, incluyendo las líneas `-----BEGIN OPENSSH PRIVATE KEY-----` y `-----END OPENSSH PRIVATE KEY-----`.

---

## Paso 5 — GitHub Actions Workflow

### `.github/workflows/build-deploy.yml`

```yaml
name: Build and Deploy Crypto Chatbot

on:
  push:
    branches: [main]

env:
  REGION: us-central1
  REGISTRY: us-central1-docker.pkg.dev
  REPO: agent-repo

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # ─────────────────────────────────
      # Checkout del código
      # ─────────────────────────────────
      - name: Checkout repo
        uses: actions/checkout@v4

      # ─────────────────────────────────
      # Crear .env del backend dinámicamente
      # ─────────────────────────────────
      - name: Create backend .env
        run: |
          echo "GOOGLE_API_KEY=${{ secrets.GOOGLE_API_KEY }}" > backend/.env

      # ─────────────────────────────────
      # Autenticación en GCP
      # ─────────────────────────────────
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: '${{ secrets.GCP_SA_KEY }}'

      - name: Configure Docker for Artifact Registry
        run: |
          gcloud auth configure-docker ${{ env.REGISTRY }}

      # ─────────────────────────────────
      # Build y Push — Backend (Python)
      # ─────────────────────────────────
      - name: Build backend image
        run: |
          docker build \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:${{ github.sha }} \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:latest \
            backend/

      - name: Push backend image
        run: |
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:${{ github.sha }}
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:latest

      # ─────────────────────────────────
      # Build y Push — Frontend (Nginx)
      # ─────────────────────────────────
      - name: Build frontend image
        run: |
          docker build \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:${{ github.sha }} \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:latest \
            frontend/

      - name: Push frontend image
        run: |
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:${{ github.sha }}
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:latest

      # ─────────────────────────────────
      # Deploy vía SSH al Control Node
      # ─────────────────────────────────
      - name: Deploy via Control Node
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.CONTROL_HOST }}
          username: ${{ secrets.CONTROL_HOST_USER }}
          key: ${{ secrets.CONTROL_HOST_SSH_KEY }}
          port: ${{ secrets.CONTROL_HOST_PORT }}
          script: |
            source ~/ansible-venv/bin/activate
            cd ~/ansible-gcp
            ansible-playbook -i inventory/gcp.yml \
              playbooks/deploy-bot.yml \
              --extra-vars "image_tag=${{ github.sha }}"
```

### Notas sobre el Workflow

- Se etiquetan las imágenes tanto con el SHA del commit (trazabilidad) como con `latest` (conveniencia).
- En producción se debería hacer un `git pull` en el Control Node asegurando que los playbooks y el inventario estén siempre actualizados. En este ejercicio, al usar el nodo de control también como repositorio git para hacer push, no lo complicamos mas.
- Se usa `google-github-actions/auth@v2` (versión actualizada).

---

## Paso 6 — Playbook de Ansible (Deploy)

Este playbook se ejecuta en el Control Node y despliega los contenedores en la VM de destino.

### `playbooks/deploy-bot.yml`

```yaml
---
- name: Deploy Crypto Chatbot containers
  hosts: all
  become: true

  vars:
    registry_base: "us-central1-docker.pkg.dev/banded-pad-481109-q1/agent-repo"
    docker_network: "chatbot-network"

  tasks:

    - name: Crear red Docker
      community.docker.docker_network:
        name: "{{ docker_network }}"
        state: present

    - name: Desplegar backend
      community.docker.docker_container:
        name: agent-backend
        image: "{{ registry_base }}/agent-backend:{{ image_tag }}"
        state: started
        restart_policy: always
        pull: true
        recreate: true
        networks:
          - name: "{{ docker_network }}"
        ports:
          - "8000:80"

    - name: Desplegar frontend
      community.docker.docker_container:
        name: agent-frontend
        image: "{{ registry_base }}/agent-frontend:{{ image_tag }}"
        state: started
        restart_policy: always
        pull: true
        recreate: true
        networks:
          - name: "{{ docker_network }}"
        ports:
          - "80:80"

    - name: Verificar contenedores
      command: docker ps --format "{{ '{{' }}.Names{{ '}}' }} - {{ '{{' }}.Status{{ '}}' }} - {{ '{{' }}.Image{{ '}}' }}"
      register: docker_status

    - name: Mostrar estado
      debug:
        msg: "{{ docker_status.stdout_lines }}"
```

### Notas del Playbook

- Se usa `delegate_to: localhost` para leer la clave de SA desde el Control Node (no desde la VM remota).
- `recreate: true` fuerza la recreación del contenedor si la imagen ha cambiado.
- `pull: false` porque ya hemos hecho el pull explícito (más control sobre errores).
- Se incluye una tarea de **limpieza** para evitar acumulación de imágenes antiguas.
- La **verificación final** confirma que ambos contenedores están corriendo.

---



---

##  Paso 7 — Verificación y Troubleshooting

### Verificar el Pipeline Completo

**1. Verificar Artifact Registry (desde local o Cloud Shell):**

```bash
gcloud artifacts docker images list \
  $REGION-docker.pkg.dev/$PROJECT_ID/agent-repo
```


**2. Verificar contenedores en la VM de despliegue:**

```bash
# En la VM de despliegue
docker ps
docker logs agent-backend
docker logs agent-frontend
```

**3. Test funcional del chatbot:**

```bash
# Desde la VM
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"input": "What is the price of Bitcoin?"}'

# Desde fuera (IP pública de la VM)
 http://EXTERNAL_IP

```

------------------------------------------------------------------------------------------------------------------


