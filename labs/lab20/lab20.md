---
layout: lab
title: "Práctica 20: Restore completo" # CAMBIAR POR CADA PRACTICA
permalink: /lab20/lab20/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab20/img # CAMBIAR POR CADA PRACTICA
duration: "35 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Levantar Couchbase en Docker, crear un dataset de ejemplo, realizar un **backup completo** con `cbbackupmgr`, simular un **incidente** (borrado del bucket) y ejecutar un **restore completo** del repo al mismo clúster. Validarás integridad por conteos y muestreos N1QL y documentarás evidencias del proceso.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 3–4 GB de RAM libres (servicios `data,index,query`).  
introduction: | # CAMBIAR POR CADA PRACTICA
  La propiedad `cbbackupmgr` es la herramienta de **Backup & Restore** oficial de Couchbase (edición Enterprise). Un **restore completo** recupera íntegramente los datos (y metadatos aplicables) desde un **archive/repo** hacia el clúster de destino. En este ejercicio crearás primero un **backup full**, simularás un **borrado** del bucket de aplicación y luego **restaurarás** todo el contenido, comprobando que el conteo coincide y que las consultas vuelven a funcionar.
slug: lab20 # CAMBIAR POR CADA PRACTICA
lab_number: 20 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Realizaste un **backup completo**, simulaste la pérdida del bucket y ejecutaste un **restore completo** con `cbbackupmgr`, validando que los datos vuelven a estar íntegros y consultables.
notes: | # CAMBIAR POR CADA PRACTICA
  - En entornos productivos, aloja el **archive** en almacenamiento duradero (NFS/objetos) con **retención** definida.  
  - Si tu clúster usa **TLS**, emplea `https://` y certificados válidos en `backup/restore`.  
  - `cbbackupmgr` puede restaurar **GSI** y otros metadatos; asegúrate de que el servicio `index` está disponible y con RAM suficiente.  
  - Evita depender de índices primarios; crea los índices secundarios necesarios tras el restore según tus consultas.  
  - Documenta un **runbook** de DR con pasos y tiempos medidos.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Propiedad cbbackupmgr (comandos y opciones)
    url: https://docs.couchbase.com/server/current/backup-restore/cbbackupmgr.html 
  - text: Conceptos de archive/repo
    url: https://docs.couchbase.com/server/current/backup-restore/backup-restore.html
    
prev: /lab19/lab19/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab21/lab21/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Estructura base y variables

Crear la carpeta de práctica, subdirectorios persistentes y un `.env` con variables del clúster, datos y rutas del repositorio de backups.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica20-restore-full/
  mkdir -p practica20-restore-full/couchbase/{data,logs,config}
  mkdir -p practica20-restore-full/backups
  mkdir -p practica20-restore-full/scripts
  cd practica20-restore-full
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-restore-n1
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

  # Rutas de backup (montadas en el contenedor)
  BK_ARCHIVE=/backups/archive1
  BK_REPO=repo_full
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Docker Compose y salud del nodo

Definir `compose.yaml` para 1 nodo Couchbase con volúmenes persistentes y un volumen adicional para **backups**.

#### Tarea 2.1

- **Paso 5.** Ahora crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente codigo en la terminal.

  > **NOTA:**
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

- **Paso 6.** Inicia el servicio, dentro de la terminal ejecuta el siguiente comando.

  > **IMPORTANTE:** Para agilizar los procesos, la imagen ya esta descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  {: .lab-note .important .compact}
  > **IMPORTANTE:** El `docker compose up -d` corre en segundo plano. El healthcheck del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en 8091 antes de continuar.
  {: .lab-note .important .compact}

  ```bash
  docker compose up -d
  ```
  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 7.** Verifica que el contenedor se haya creado correctamente.

  {%raw%}
  ```bash
  docker ps --filter "name=cb-restore-n1"
  docker inspect -f '{{.State.Health.Status}}' cb-restore-n1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Inicializar clúster, crear datos e índice para validaciones

Inicializa el clúster, crea bucket/scope/collection y carga 1 000 documentos de ejemplo para luego respaldarlos.

#### Tarea 3.1

- **Paso 8.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica20-restore-full**. Puede tardar unos segundos en inicializar.
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

- **Paso 9.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **NOTA:**
  - Contenedor `cb-restore-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-restore-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 10.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${APP_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize ${APP_BUCKET_RAM} --enable-flush 1
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 11.** Ahora crea el *Scope* **shop**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes" \
    -d "name=${APP_SCOPE}"
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 12.** Con este comando crea el *Collection* **products**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes/${APP_SCOPE}/collections" \
    -d "name=${APP_COLLECTION}"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 13.** Crea el índice primario temporal para validaciones

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON ${APP_BUCKET}.${APP_SCOPE}.${APP_COLLECTION};"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 14.** Carga 1000 documentos (loop con N1QL)

  > **IMPORTANTE:** Es normal que la terminal se quede en espera, ya que esta insertando los 1000 registros. El proceso finalizara solo, espera unos minutos.
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

- **Paso 15.** Verifica que se hayan cargado correctamente, realiza la consulta de conteo.

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

### Tarea 4: Crear **archive/repo** y ejecutar **backup completo**

Configura el `archive` y el `repo` de `cbbackupmgr` y realiza un **backup completo** del bucket de aplicación.

#### Tarea 4.1

- **Paso 16.** Crea el archive y repo (dentro del contenedor)

  ```bash
  docker exec -it "${CB_CONTAINER}" bash -lc "
    mkdir -p '${BK_ARCHIVE}' &&
    cbbackupmgr config --archive '${BK_ARCHIVE}' --repo '${BK_REPO}'
  "
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 17.** Ejecuta el **backup** completo.

  > **NOTA:** Es normal que el proceso tarde, espera unos minutos.
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

- **Paso 18.** Ahora inspecciona el repo

  ```bash
  MSYS2_ARG_CONV_EXCL="*" docker exec -it "${CB_CONTAINER}" cbbackupmgr info \
    --archive "/backups/archive1" --repo "${BK_REPO}"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: **Simular incidente** y ejecutar **Restore completo**

Simularás pérdida del bucket borrándolo del clúster y **restaurarás** el contenido desde el repo. Verificarás conteos y muestreos tras el restore.

#### Tarea 5.1

- **Paso 19.** Borra el bucket para la simulación del desastre.

  > **NOTA:** Espera unos minutos en lo que se elimina el bucket.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Realiza los siguientes puntos **solo despues** en caso de que te aparezca un error y se cierre sola la ventana.
  - Abre la ventana en el icono superior derecho
  - Entra al directorio de la practica `cd practica20-restore-full`
  - Carga las variables: `set -a; source .env; set +a`
  - Vuelve a intenar eliminar el bucket y debera indicar **Bucket not found** ya que si lo elimino la primera vez. En ocasiones la terminal se llena de recursos y se cierra.
  {: .lab-note .important .compact}

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-delete \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${APP_BUCKET}"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 20.** Realiza la restauracion **completa** desde el repo.

  > **NOTA:** Si tu backup contiene múltiples buckets o colecciones, `--map-data` te permite controlar exactamente a dónde se restauran. Para esta practica no es necesario.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Espera unos minutos en lo que se restaura el bucket y la información
  {: .lab-note .important .compact}

  ```bash
  MSYS2_ARG_CONV_EXCL="*" docker exec -it "${CB_CONTAINER}" bash -lc "
    cbbackupmgr restore \
      --archive '${BK_ARCHIVE}' --repo '${BK_REPO}' \
      --cluster http://127.0.0.1:8091 \
      --username '${CB_ADMIN}' --password '${CB_ADMIN_PASS}' \
      --auto-create-buckets \
      --purge
  "
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 21.** Ahora recrea el índice primario para validar.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON ${APP_BUCKET}.${APP_SCOPE}.${APP_COLLECTION};"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 22.** Valida la restauracion con la siguiente consulta de conteo.

  ```bash
  docker exec -i "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT COUNT(*) AS n
        FROM \`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`
        WHERE META().id LIKE 'product::%';"
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 23.** Muestra 3 registos para reconfirmar que la restauración fue un exito.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq \
    -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "SELECT META().id, name, price
        FROM \`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`
        ORDER BY META().id
        LIMIT 3;"
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Limpieza

Borrar datos en el entorno para repetir pruebas.

#### Tarea 6.1

- **Paso 24.** En la terminal aplica el siguiente comando para detener el nodo.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

- **Paso 25.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}