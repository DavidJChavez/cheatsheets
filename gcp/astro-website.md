# 🚀 GCP Static Website Deployment - Complete Cheatsheet

## 📋 Overview

Desplegar un sitio estático de Astro en Google Cloud Platform usando Cloud Storage + Load Balancer con SSL.

---

## 🏗️ Arquitectura Final

```
DNS (Wix) → Load Balancer (GCP) → Cloud Storage Bucket → Sitio Astro
```

---

## 1️⃣ Preparación del Bucket

### Crear y configurar bucket

```bash
# Crear bucket (nombre puede ser diferente al dominio)
gsutil mb gs://www.tudominio.com

# Subir archivos del build de Astro
gsutil -m rsync -r -d ./dist gs://www.tudominio.com

# Hacer bucket público
gsutil iam ch allUsers:objectViewer gs://www.tudominio.com

# Configurar como sitio web
gsutil web set -m index.html -e 404.html gs://www.tudominio.com

# Verificar configuración web
gsutil web get gs://www.tudominio.com
```

### Verificar contenido

```bash
# Listar archivos
gsutil ls gs://www.tudominio.com/

# Probar acceso directo
curl -I "https://storage.googleapis.com/www.tudominio.com/index.html"
```

---

## 2️⃣ Configuración del Load Balancer

### Crear certificado SSL

```bash
gcloud compute ssl-certificates create tudominio-ssl \
    --domains=tudominio.com,www.tudominio.com \
    --global
```

### Crear backend bucket

```bash
gcloud compute backend-buckets create tudominio-backend \
    --gcs-bucket-name=www.tudominio.com \
    --global
```

### Crear Load Balancer (vía Console)

1. **Network Services** → **Load Balancing** → **Create Load Balancer**
2. **Application Load Balancer (HTTP/HTTPS)**
3. **Frontend:** HTTPS + Certificado SSL + IP estática
4. **Backend:** Backend bucket creado
5. **Opcional:** HTTP → HTTPS redirect

### Verificar configuración

```bash
# Ver Load Balancer
gcloud compute url-maps list
gcloud compute url-maps describe tudominio-lb --global

# Ver forwarding rules
gcloud compute forwarding-rules list --global

# Ver certificado SSL
gcloud compute ssl-certificates describe tudominio-ssl --global
```

---

## 3️⃣ Configuración DNS (Wix)

### Eliminar registros antiguos

❌ Eliminar:

- IPs de Wix (185.x.x.x)
- CNAME a `c.storage.googleapis.com`

### Configurar registros correctos

✅ Agregar:

```
Tipo: A
Host: @ (o tudominio.com)
Value: [IP_DEL_LOAD_BALANCER]
TTL: 1 Hour

Tipo: A
Host: www
Value: [IP_DEL_LOAD_BALANCER]
TTL: 1 Hour
```

### Verificar propagación DNS

```bash
# Verificar DNS
nslookup tudominio.com 8.8.8.8
nslookup www.tudominio.com 8.8.8.8

# Verificar en línea
# dnschecker.org
```

---

## 4️⃣ GitHub Actions - Auto Deploy

### Archivo: `.github/workflows/deploy.yml`

```yaml
name: Deploy to Google Cloud Storage

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Bun
        uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest

      - name: Install dependencies
        run: bun install --frozen-lockfile

      - name: Build Astro site
        run: bun run build

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2
        with:
          version: ">= 390.0.0"

      - name: Deploy to Cloud Storage
        run: |
          gsutil -m rsync -r -d ./dist gs://www.tudominio.com
          gsutil -m setmeta -h "Cache-Control:public, max-age=31536000" gs://www.tudominio.com/**/*.{js,css,png,jpg,jpeg,gif,svg,ico,woff,woff2}
          gsutil -m setmeta -h "Cache-Control:public, max-age=3600" gs://www.tudominio.com/**/*.html
          gsutil web set -m index.html -e 404.html gs://www.tudominio.com
```

### Configurar Service Account

```bash
# Crear service account
gcloud iam service-accounts create github-actions-sa \
    --description="Service account for GitHub Actions"

# Dar permisos
gcloud projects add-iam-policy-binding TU_PROJECT_ID \
    --member="serviceAccount:github-actions-sa@TU_PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.admin"

# Crear key
gcloud iam service-accounts keys create key.json \
    --iam-account=github-actions-sa@TU_PROJECT_ID.iam.gserviceaccount.com
```

### Secrets en GitHub

- **GCP_SA_KEY:** Contenido completo del archivo `key.json`

---

## 🔧 Troubleshooting Common Issues

### DNS no propaga

```bash
# Cambiar TTL a 5 minutos temporalmente
# Usar DNS públicos: 8.8.8.8, 1.1.1.1
# Verificar en dnschecker.org
```

### SSL Certificate FAILED_NOT_VISIBLE

```bash
# Verificar que HTTP funcione primero
curl -H "Host: tudominio.com" http://[IP_LOAD_BALANCER]

# Verificar DNS apunte a la IP correcta
nslookup tudominio.com

# Recrear certificado si es necesario
gcloud compute ssl-certificates delete tudominio-ssl --global
# Esperar 5 minutos
gcloud compute ssl-certificates create tudominio-ssl-new \
    --domains=tudominio.com,www.tudominio.com \
    --global
```

### Timeout en el sitio

```bash
# Verificar que DNS apunte a la IP correcta del Load Balancer
gcloud compute forwarding-rules list --global

# Verificar backend bucket configuración
gcloud compute backend-buckets describe tudominio-backend

# Probar acceso directo al bucket
curl -I "https://storage.googleapis.com/www.tudominio.com/index.html"
```

### Error CNAME en Wix

- **Problema:** "Este nombre de host ya se utiliza"
- **Solución:** Usar registro A en lugar de CNAME para `www`

---

## ⏰ Tiempos Típicos

| Proceso         | Tiempo               |
| --------------- | -------------------- |
| DNS Propagation | 10 minutos - 6 horas |
| SSL Certificate | 15-90 minutos        |
| Load Balancer   | 5-15 minutos         |
| GitHub Actions  | 3-5 minutos          |

---

## 🎯 Comandos de Verificación Rápida

```bash
# Estado del certificado SSL
gcloud compute ssl-certificates describe tudominio-ssl --global --format="value(managed.status,managed.domainStatus)"

# Probar sitio por IP
curl -H "Host: tudominio.com" http://[IP_LOAD_BALANCER]

# DNS actual
nslookup tudominio.com 8.8.8.8

# Contenido del bucket
gsutil ls gs://www.tudominio.com/

# Load Balancer config
gcloud compute url-maps describe tudominio-lb --global
```

---

## 💡 Pro Tips

1. **Firebase Hosting** es más fácil para sitios estáticos simples
2. **Reduce TTL a 300 segundos** durante configuración inicial
3. **Usa HTTP primero** para validar antes de configurar HTTPS
4. **Prueba desde móvil** para verificar DNS más rápido
5. **Mantén nombres consistentes** entre recursos
6. **Documenta las IPs** de tus Load Balancers

---

## 🚨 Never Forget

- ✅ Bucket público con `allUsers:objectViewer`
- ✅ Web configuration con `index.html`
- ✅ DNS apuntando a la IP **correcta** del Load Balancer
- ✅ HTTP configurado **antes** que HTTPS
- ✅ Certificado SSL para **ambos** dominios (con y sin www)

---

_¡Y recuerda: siempre verificar que la IP del DNS coincida con la IP del Load Balancer!_ 🎯
