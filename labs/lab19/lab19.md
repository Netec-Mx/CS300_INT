---
layout: lab
title: "Práctica 19: Backup incremental y restauración parcial" # CAMBIAR POR CADA PRACTICA
permalink: /lab19/lab19/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab19/img # CAMBIAR POR CADA PRACTICA
duration: "35 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Configurar un clúster Couchbase en Docker, generar un **backup completo** inicial y después un **backup incremental** con `cbbackupmgr`. Practicarás **restauración parcial** por **colección** (collection-aware) y por **subconjunto de documentos** usando filtros de claves. Validarás conteos y muestras antes y después para comprobar la integridad.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 3–4 GB de RAM libres (servicios `data,index,query`).  
introduction: | # CAMBIAR POR CADA PRACTICA
  La propiedad `cbbackupmgr` administra backups mediante la estructura **archive → repo → backups**. En un **repo** puedes tener múltiples ejecuciones: un **completo** seguido de **incrementales** que capturan sólo los cambios desde la ejecución previa. La **restauración parcial** permite traer únicamente una **colección** específica y/o un **rango de documentos** (por clave) a un **bucket** destino, útil para recuperación selectiva o ambientes de prueba.
slug: lab19 # CAMBIAR POR CADA PRACTICA
lab_number: 19 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Construiste un flujo **full + incremental** y ejecutaste **restauraciones parciales** por **colección** y por **subconjunto de claves**. Verificaste conteos antes y después y generaste evidencia de que la restauración selectiva se aplica correctamente sobre el destino.
notes: | # CAMBIAR POR CADA PRACTICA
  - En producción, externaliza el **archive** a almacenamiento duradero (NFS/Objetos) y define **retención** (merge/remove).  
  - Si usas **TLS**, cambia `http://` por `https://` y configura certificados.  
  - Para restaurar a un punto en concreto, usa `cbbackupmgr list` para identificar el backup y la opción `--time` en `restore`.  
  - Evita dejar índices primarios; aquí se usan solo para validaciones rápidas.  
  - Documenta tareas programadas (cron/CI) para ejecutar incrementales con frecuencia adecuada.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Propiedad `cbbackupmgr` (comandos y opciones)
    url: https://docs.couchbase.com/server/current/backup-restore/cbbackupmgr.html 
  - text: Restauración por colección (collection-aware)
    url: https://www.couchbase.com/blog/es/import-mongodb-collection-data-couchbase-server-golang/
    
prev: /lab18/lab18/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab20/lab20/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1. Estructura del proyecto y variables.

Crearás la carpeta de práctica con subdirectorios persistentes y un `.env` centralizando variables del clúster, datos y rutas de backup.

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**.
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, da clic en el ícono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha**.

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica19-backup-inc/
  mkdir -p practica19-backup-inc/couchbase/{data,logs,config}
  mkdir -p practica19-backup-inc/backups
  mkdir -p practica19-backup-inc/scripts
  cd practica19-backup-inc
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **Nota.** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-inc-n1
  CB_HOST=couchbase1

  # Credenciales admin
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'

  # Servicios
  CB_SERVICES=data,index,query
  CLUSTER_RAM=3072
  INDEX_RAM=768


  # Bucket origen (datos v1 y v2)
  SRC_BUCKET=app
  SRC_SCOPE=shop
  SRC_COLLECTION=products
  SRC_BUCKET_RAM=512

  # Bucket destino para restauración parcial
  DST_BUCKET=restore
  DST_SCOPE=shop
  DST_COLLECTION=products
  DST_BUCKET_RAM=512

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

Definirás un `compose.yaml` para 1 nodo Couchbase con volúmenes persistentes e iniciarás el contenedor.

- **Paso 1.** Crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente codigo en la terminal.

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

  > **Importante.** Para agilizar los procesos, la imagen ya está descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
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
  docker ps --filter "name=cb-inc-n1"
  docker inspect -f '{{.State.Health.Status}}' cb-inc-n1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Inicialización, datos v1 e índice para validaciones.

Inicializa el clúster, crea bucket/scope/collection y carga **v1** de datos (1000 documentos) para respaldar.

- **Paso 1.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **Nota.** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **Importante.** El comando se ejecuta desde el directorio de la práctica **practica19-backup-inc**. Puede tardar unos segundos en inicializar.
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
  - Contenedor `cb-inc-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexión es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-inc-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 3.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${SRC_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize ${SRC_BUCKET_RAM} --enable-flush 1
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 4.** Ahora, crea el *Scope* **shop**.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${SRC_BUCKET}/scopes" \
    -d "name=${SRC_SCOPE}"
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 5.** Con este comando crea el *Collection* **products**.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${SRC_BUCKET}/scopes/${SRC_SCOPE}/collections" \
    -d "name=${SRC_COLLECTION}"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 6.** Crea el índice primario temporal para validaciones.

  ```bash
  docker exec -it ${CB_CONTAINER} cbq -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON ${SRC_BUCKET}.${SRC_SCOPE}.${SRC_COLLECTION};"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 7.** Carga 1 000 documentos (loop con N1QL).

  > **Importante.** Es normal que la terminal se quede en espera, ya que está insertando los 1 000 registros. El proceso finalizará solo. Espera unos minutos.
  {: .lab-note .important .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${SRC_BUCKET}\`.\`${SRC_SCOPE}\`.\`${SRC_COLLECTION}\`"
  for i in $(seq 1 1000); do
    name="Item-$i"

    # Precio aleatorio entre 1.00 y 9.99
    cents=$((RANDOM%900 + 100))            # 100..999  -> 1.00..9.99
    price=$(printf "%d.%02d" $((cents/100)) $((cents%100)))

    # Arma N1QL
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
        FROM \`${SRC_BUCKET}\`.\`${SRC_SCOPE}\`.\`${SRC_COLLECTION}\`
        WHERE META().id LIKE 'product::%';"
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear **archive/repo** y ejecutar **backup completo**.

Configura el `archive` y `repo` de `cbbackupmgr` y realiza el primer backup (completo).

- **Paso 1.** Crea el archive y repo.

  ```bash
  docker exec -it "${CB_CONTAINER}" bash -lc "
    mkdir -p '${BK_ARCHIVE}' && \
    cbbackupmgr config --archive '${BK_ARCHIVE}' --repo '${BK_REPO}'
  "
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 2.** Ejecuta el **backup** completo (captura *v1*).

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

### Tarea 5. Mutaciones (v2) y **backup incremental**.

Generarás cambios: **actualizaciones** de 100 docs y **nuevos** 50 docs. Luego, harás un **backup incremental** que capture solo esas mutaciones.

- **Paso 1.** Ahora, realiza una mutación para **v2**, 100 actualizaciones (product::1..100).

  > **Nota.** Es normal que el proceso tarde, está realizando las actualizaciones.
  {: .lab-note .info .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${SRC_BUCKET}\`.\`${SRC_SCOPE}\`.\`${SRC_COLLECTION}\`"
  for i in $(seq 1 100); do
    n1ql=$(printf 'UPSERT INTO %s (KEY,VALUE) VALUES ("product::%d", {"type":"product","name":"Item-%d-v2","price":99.99,"ts":NOW_MILLIS()});' \
                  "$KEYSPACE" "$i" "$i")
    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "$n1ql" >/dev/null
  done
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 2.** Otra mutación más para **v2**, 50 nuevas inserciones (product::1001..1050).

  > **Nota.** Es normal que el proceso tarde, está realizando las inserciones adicionales.
  {: .lab-note .info .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${SRC_BUCKET}\`.\`${SRC_SCOPE}\`.\`${SRC_COLLECTION}\`"
  for i in $(seq 1001 1050); do
    n1ql=$(printf 'UPSERT INTO %s (KEY,VALUE) VALUES ("product::%d", {"type":"product","name":"Item-%d","price":19.99,"ts":NOW_MILLIS()});' \
                  "$KEYSPACE" "$i" "$i")
    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "$n1ql" >/dev/null
  done
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 3.** Verifica que se hayan cargado correctamente, realiza la consulta de conteo tras el cambio para **v2**.

  ```bash
  docker exec -i "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT COUNT(*) AS n
        FROM \`${SRC_BUCKET}\`.\`${SRC_SCOPE}\`.\`${SRC_COLLECTION}\`
        WHERE META().id LIKE 'product::%';"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 4.** Realiza el Backup **incremental**.

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
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Crear bucket **destino** y **restauración parcial (por colección)**.

Restaurarás **solo la colección** `${SRC_SCOPE}.${SRC_COLLECTION}` al bucket `${DST_BUCKET}` usando **collection-aware restore**.

- **Paso 1.** Ejecuta el siguiente comando para la creación del bucket destino.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${DST_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize ${DST_BUCKET_RAM} --enable-flush 1
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 2.** Ahora, crea el *Scope* **shop**.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${DST_BUCKET}/scopes" \
    -d "name=${DST_SCOPE}"
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

- **Paso 3.** Con este comando, crea el *Collection* **products**.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${DST_BUCKET}/scopes/${DST_SCOPE}/collections" \
    -d "name=${DST_COLLECTION}"
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

- **Paso 4.** Restaura **solo la colección** (`--map-data`).

  > **Nota.** Espera unos minutos por la restauración en el bucket de respaldo.
  {: .lab-note .info .compact}

  ```bash
  # Define mapeo de bucket y filtro de colección
  MAP_DATA="${SRC_BUCKET}=${DST_BUCKET}"
  INCLUDE="${SRC_BUCKET}.${SRC_SCOPE}.${SRC_COLLECTION}"

  # Restore solo esa colección, cambiando de bucket
  MSYS2_ARG_CONV_EXCL="*" docker exec -it \
    -e BK_ARCHIVE="${BK_ARCHIVE}" \
    -e BK_REPO="${BK_REPO}" \
    -e CB_ADMIN="${CB_ADMIN}" \
    -e CB_ADMIN_PASS="${CB_ADMIN_PASS}" \
    -e MAP_DATA="${MAP_DATA}" \
    -e INCLUDE="${INCLUDE}" \
    "${CB_CONTAINER}" bash -lc '
    cbbackupmgr restore \
      --archive "$BK_ARCHIVE" --repo "$BK_REPO" \
      --cluster http://127.0.0.1:8091 \
      --username "$CB_ADMIN" --password "$CB_ADMIN_PASS" \
      --map-data "$MAP_DATA" \
      --include-data "$INCLUDE"
  '
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

- **Paso 5.** Crea el índex en el bucket destino.

  ```bash
  docker exec -it ${CB_CONTAINER} cbq -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON ${DST_BUCKET}.${DST_SCOPE}.${DST_COLLECTION};"
  ```
  ![cbase24]({{ page.images_base | relative_url }}/24.png)

- **Paso 6.** Realiza el conteo en el destino con la siguiente consulta.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT COUNT(*) AS n
        FROM \`${DST_BUCKET}\`.\`${SRC_SCOPE}\`.\`${SRC_COLLECTION}\`;"
  ```
  ![cbase25]({{ page.images_base | relative_url }}/25.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Limpieza.

Borrar datos en el entorno para repetir pruebas.

- **Paso 1.** En la terminal, aplica el siguiente comando para detener el nodo.

  > **Nota.** Si es necesario, puedes volver a encender los contenedores con el comando **`docker compose start`**.
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase26]({{ page.images_base | relative_url }}/26.png)

- **Paso 2.** Apagar y eliminar el contenedor (se conservan los datos en ./data).

  > **Nota.** Si es necesario, puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **Importante.** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase27]({{ page.images_base | relative_url }}/27.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

