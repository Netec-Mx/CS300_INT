---
layout: lab
title: "Práctica 18: Backup completo" # CAMBIAR POR CADA PRACTICA
permalink: /lab18/lab18/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab18/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar Couchbase en Docker, generar datos de ejemplo y realizar un **backup completo** con `cbbackupmgr`. Validarás el repositorio de respaldos, inspeccionarás metadatos.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 3–4 GB de RAM libres (servicios `data,index,query`).  
introduction: | # CAMBIAR POR CADA PRACTICA
  La propiedad `cbbackupmgr` es la herramienta oficial de **backup/restore** para Couchbase Server Enterprise. Permite administrar **archivos de respaldo** organizados por **archive** y **repo**, incluyendo ciclos completos/incrementales, retención y restauraciones granulares (por bucket/scope/collection). En esta práctica, crearás un **repositorio de backups**, ejecutarás un **backup completo** del bucket de aplicación.
slug: lab18 # CAMBIAR POR CADA PRACTICA
lab_number: 18 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Has creado un **repositorio de backups**, ejecutado un **backup completo** e inspeccionado su contenido.
notes: | # CAMBIAR POR CADA PRACTICA
  - Para producción, programa backups con un **scheduler** externo y almacén **durable** (NAS/NFS/s3fs).  
  - Considera **encriptación a nivel de volumen** y control de accesos a la ruta de `archive`.  
  - Documenta políticas de retención y pruebas periódicas de **restore** (no solo backup).  
  - Si el cluster usa **TLS**, conecta con `https://` y certificados válidos.  
  - Evita usar índices primarios en producción; aquí se emplean solo para validaciones rápidas.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Propiedad cbbackupmgr (comandos y opciones)
    url: https://docs.couchbase.com/server/current/backup-restore/cbbackupmgr.html 
  - text: Conceptos de archive/repo y retención
    url: https://docs.couchbase.com/server/current/backup-restore/backup-restore.html
prev: /lab17/lab17/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab19/lab19/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1. Estructura de práctica y variables.

Crear directorios de trabajo y un `.env` con variables reutilizables para el clúster y el repositorio de backups.

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**.
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, da clic en el ícono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha**.

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando, debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica18-backup/
  mkdir -p practica18-backup/couchbase/{data,logs,config}
  mkdir -p practica18-backup/backups
  mkdir -p practica18-backup/scripts
  cd practica18-backup
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **Nota.** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-backup-n1
  CB_HOST=couchbase1

  # Credenciales admin
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'

  # Servicios
  CB_SERVICES=data,index,query
  CLUSTER_RAM=3072
  INDEX_RAM=768


  # Datos de ejemplo
  APP_BUCKET=app
  APP_SCOPE=shop
  APP_COLLECTION=products
  APP_BUCKET_RAM=512

  # Destino de restore
  RESTORE_BUCKET=restore
  RESTORE_SCOPE=shop
  RESTORE_COLLECTION=products
  RESTORE_BUCKET_RAM=512

  # Rutas de backup (montadas en el contenedor)
  BK_ARCHIVE=/backups/archive1
  BK_REPO=repo_full
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Docker Compose y salud del nodo.

Definir `compose.yaml` para un nodo Couchbase con volúmenes persistentes y un volumen extra para **backups**.

- **Paso 1.** Ahora, crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente codigo en la terminal.

  > **Nota.**
  - El archivo `compose.yaml` mapea puertos 8091–8096 para la consola web y 11210 para clientes.
  - El healthcheck consulta el endpoint `/pools` que responde cuando el servicio está arriba (aunque aún no inicializado).
  {: .lab-note .info .compact}

  ```bash
  cat > compose.yaml << 'YAML'
  services:
    couchbase:
      image: couchbase/server:${CB_TAG}
      container_name: ${CB_CONTAINER}
      hostname: ${CB_HOST}
      ports:
        - "8091-8096:8091-8096"   # Web UI / servicios
        - "11210:11210"           # Memcached/SDK (data)
      environment:
        # (La inicialización la haremos con CLI; aquí solo variables útiles)
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      volumes:
        - ./couchbase/data:/opt/couchbase/var
        - ./couchbase/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./couchbase/config:/config
        - ./backups:/backups              # <— carpeta de backups en host
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 12
      restart: unless-stopped
  YAML
  ```

- **Paso 2.** Inicia el servicio, dentro de la terminal, ejecuta el siguiente comando.

  > **Importante.** Para agilizar los procesos, la imagen ya esta descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  {: .lab-note .important .compact}
  > **Importante.** El `docker compose up -d` corre en segundo plano. El healthcheck del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en 8091 antes de continuar.
  {: .lab-note .important .compact}

  ```bash
  docker compose up -d
  ```
  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 3.** Verifica que el contenedor se haya creado correctamente.

  {%raw%}
  ```bash
  docker ps --filter "name=cb-backup-n1"
  docker inspect -f '{{.State.Health.Status}}' cb-backup-n1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Inicializar clúster y preparar datos.

Inicializar el clúster, crear bucket/scope/collection y cargar datos de ejemplo que luego se respaldarán.

- **Paso 1.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **Nota.** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **Importante.** El comando se ejecuta desde el directorio de la practica **practica18-backup**. Puede tardar unos segundos en inicializar.
  {: .lab-note .important .compact}

  ```bash
  set -a; source .env; set +a
  docker exec -it "$CB_CONTAINER" couchbase-cli cluster-init \
    --cluster 127.0.0.1 \
    --cluster-username "$CB_ADMIN" \
    --cluster-password "$CB_ADMIN_PASS" \
    --services "${SERVICES_NORM:-$CB_SERVICES}" \
    --cluster-ramsize "$CLUSTER_RAM" \
    --cluster-index-ramsize "$INDEX_RAM" \
    --index-storage-setting default
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

- **Paso 2.** Verifica que el clúster esté **healthy** y que se muestre el json con las propiedades del nodo.

  > **Nota.**
  - Contenedor `cb-backup-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-backup-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 3.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${APP_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize ${APP_BUCKET_RAM} --enable-flush 1
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 4.** Ahora, crea el *Scope* **shop**.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes" \
    -d "name=${APP_SCOPE}"
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 5.** Con este comando crea el *Collection* **products**.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes/${APP_SCOPE}/collections" \
    -d "name=${APP_COLLECTION}"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 6.** Crea el índice primario temporal para validaciones.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON ${APP_BUCKET}.${APP_SCOPE}.${APP_COLLECTION};"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 7.** Carga 1 000 documentos (loop con N1QL).

  > **Importante.** Es normal que la terminal se quede en espera, ya que está insertando los 1 000 registros. El proceso finalizará solo. Espera unos minutos.
  {: .lab-note .important .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`"
  for i in $(seq 1 1000); do
    name="Item-$i"

    # Genera precio entre 1.00 y 9.99
    cents=$((RANDOM%900 + 100))             # 100..999  => 1.00..9.99
    price=$(printf "%d.%02d" $((cents/100)) $((cents%100)))

    # Arma la sentencia N1QL con claves JSON
    n1ql=$(printf 'UPSERT INTO %s (KEY,VALUE) VALUES ("product::%d", {"type":"product","name":"%s","price":%s,"ts":NOW_MILLIS()});' \
                  "$KEYSPACE" "$i" "$name" "$price")

    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "$n1ql" >/dev/null
  done
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 8.** Verifica que se hayan cargado correctamente, realiza la consulta de conteo.

  ```bash
  docker exec -i "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT COUNT(*) AS n
        FROM \`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`
        WHERE META().id LIKE 'product::%';"
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear **archive/repo** y ejecutar **backup completo**.

Crearás el **archive** y el **repo** de `cbbackupmgr` y realizarás un **backup** completo del clúster.

- **Paso 1.** Crea el archive y repo (dentro del contenedor).

  ```bash
  docker exec -it "${CB_CONTAINER}" bash -lc "
    mkdir -p '${BK_ARCHIVE}' &&
    cbbackupmgr config --archive '${BK_ARCHIVE}' --repo '${BK_REPO}'
  "
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 2.** Ejecuta el **backup** completo.

  > **Nota.** Es normal que el proceso tarde, espera unos minutos.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it "${CB_CONTAINER}" bash -lc "
    cbbackupmgr backup \
      --archive '${BK_ARCHIVE}' --repo '${BK_REPO}' \
      --cluster http://127.0.0.1:8091 \
      --username '${CB_ADMIN}' --password '${CB_ADMIN_PASS}' \
      --no-progress-bar
  "
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 3.** Ahora, inspecciona el repo.

  ```bash
  MSYS2_ARG_CONV_EXCL="*" docker exec -it "${CB_CONTAINER}" cbbackupmgr info \
    --archive "/backups/archive1" --repo "${BK_REPO}"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Limpieza.

Borrar datos en el entorno para repetir pruebas.

- **Paso 1.** En la terminal, aplica el siguiente comando para detener el nodo.

  > **Nota.** Si es necesario, puedes volver a encender los contenedores con el comando **`docker compose start`**.
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 2.** Apagar y eliminar contenedor (se conservan los datos en ./data).

  > **Nota.** Si es necesario, puedes volver a activar los contenedores con el comando **`docker compose up -d`**.
  {: .lab-note .info .compact}
  > **Importante.** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
