---
layout: lab
title: "Práctica 22: Compaction manual" # CAMBIAR POR CADA PRACTICA
permalink: /lab22/lab22/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab22/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar Couchbase en Docker, generar **fragmentación** de datos mediante mutaciones y eliminaciones y ejecutar **compaction manual** del bucket con validaciones **antes** y **después** usando métricas REST/N1QL. Documentarás evidencia de reducción de `diskUsed` vs `dataUsed`.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 3–4 GB de RAM libres (servicios `kv,index,query`).  
introduction: | # CAMBIAR POR CADA PRACTICA
  En Couchbase, cada mutación/eliminación puede dejar **segmentos obsoletos** en disco. La **compaction** (compactación) reescribe archivos de datos para recuperar espacio y purgar *tombstones*. En **couchstore** se observa directamente `diskUsed` > `dataUsed`; en **magma** la relación es distinta pero sigue existiendo proceso de *garbage collection/compaction*. Ejecutarás **compaction manual** del bucket y (opcionalmente) de **views** para ver cómo cambian los indicadores de almacenamiento.
slug: lab22 # CAMBIAR POR CADA PRACTICA
lab_number: 22 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Generaste un dataset con **fragmentación**, ejecutaste **compaction manual** del bucket y mediste **antes/después** con métricas REST. Confirmaste la reducción del espacio 
notes: | # CAMBIAR POR CADA PRACTICA
  - En producción, programa compaction **automática** por ventana horaria y umbrales de fragmentación.  
  - Evita compaction durante picos de tráfico; puede impactar I/O.  
  - En **magma**, revisa los parámetros de *compaction/garbage collection* y retenciones de *tombstones*.  
  - No dejes índices primarios; aquí se usan solo para validar.  
  - Si tu bucket tiene **réplicas** y **durabilidad**, considera el efecto en tiempos de compaction.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Compaction (overview y APIs)
    url: https://docs.couchbase.com/server/current/learn/buckets-memory-and-storage/storage.html
  - text: CLI bucket-compact y view-compact
    url: https://docs.couchbase.com/server/current/cli/cbcli/couchbase-cli-bucket-compact.html
  - text: Tareas del cluster (/pools/default/tasks)
    url: https://docs.couchbase.com/server/current/rest-api/rest-get-cluster-tasks.html

prev: /lab21/lab21/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab23/lab23/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1. Carpeta de la práctica y variables.

Crearás una carpeta aislada con subdirectorios y un `.env` con variables reutilizables.

#### Tarea 1.1.

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**.
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, da clic en el ícono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha**.

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando, debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica22-compaction/
  mkdir -p practica22-compaction/couchbase/{data,logs,config}
  mkdir -p practica22-compaction/scripts
  cd practica22-compaction
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC**, copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** el archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-compaction-n1
  CB_HOST=couchbase1

  # Credenciales admin
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'

  # Servicios
  CB_SERVICES=data,index,query
  CLUSTER_RAM=3072
  INDEX_RAM=768


  # Datos de ejemplo
  APP_BUCKET=compactionlab
  APP_SCOPE=shop
  APP_COLLECTION=products
  APP_BUCKET_RAM=512

  # Cantidades para generar fragmentación
  DOCS_BASE=500
  DELETES=250
  UPDATES=300
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Docker Compose y salud del nodo.

Definirás `compose.yaml` para un nodo Couchbase con volúmenes persistentes.

#### Tarea 2.1.

- **Paso 5.** Ahora, crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente codigo en la terminal.

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
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 12
      restart: unless-stopped
  YAML
  ```

- **Paso 6.** Inicia el servicio, dentro de la terminal, ejecuta el siguiente comando.

  > **IMPORTANTE:** para agilizar los procesos, la imagen ya está descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  {: .lab-note .important .compact}
  > **IMPORTANTE:** el `docker compose up -d` corre en segundo plano. El healthcheck del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en 8091 antes de continuar.
  {: .lab-note .important .compact}

  ```bash
  docker compose up -d
  ```
  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 7.** Verifica que el contenedor se haya creado correctamente.

  {%raw%}
  ```bash
  docker ps --filter "name=cb-compaction-n1"
  docker inspect -f '{{.State.Health.Status}}' cb-compaction-n1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Inicializar clúster, crear datos en el bucket.

Inicializarás el clúster, crearás la estructura del bucket y poblarás datos para **inducir fragmentación** (updates y deletes).

#### Tarea 3.1.

- **Paso 8.** Inicializa el clúster y ejecuta el siguiete comando en la terminal.

  > **NOTA:** el `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** el comando se ejecuta desde el directorio de la practica **practica22-compaction**. Puede tardar unos segundos en inicializar.
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

- **Paso 9.** Verifica que el clúster esté **healthy** y que se muestre el json con las propiedades del nodo.

  > **NOTA:**
  - Contenedor `cb-compaction-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-compaction-n1"
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

- **Paso 11.** Ahora crea el *Scope* **shop**.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes" \
    -d "name=${APP_SCOPE}"
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 12.** Con este comando, crea el *Collection* **products**.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes/${APP_SCOPE}/collections" \
    -d "name=${APP_COLLECTION}"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 13.** Crea el índice primario temporal para validaciones.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON ${APP_BUCKET}.${APP_SCOPE}.${APP_COLLECTION};"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 14.** Carga 500 documentos (loop con N1QL).

  > **IMPORTANTE:** es normal que la terminal se quede en espera, ya que está insertando los 500 registros. El proceso finalizara solo. Espera unos minutos.
  {: .lab-note .important .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`"
  for i in $(seq 1 500); do
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

- **Paso 16.** Genera algunas **actualizaciones** para realizar la fragmentación.

  > **IMPORTANTE:** es normal que la terminal se quede en espera, ya que está actualizando los registros. El proceso finalizará solo. Espera unos minutos.
  {: .lab-note .important .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`"
  for i in $(seq 1 "${UPDATES}"); do
    n1ql=$(printf 'UPDATE %s USE KEYS "product::%d" SET price = price * 1.15, ts = NOW_MILLIS();' \
                  "$KEYSPACE" "$i")
    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "$n1ql" >/dev/null
  done
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 17.** Ahora genera algunas **eliminaciones** para realizar la fragmentación.

  > **IMPORTANTE:** es normal que la terminal se quede en espera, ya que está eliminando 250 registros. El proceso finalizará solo. Espera unos minutos.
  {: .lab-note .important .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`"

  for i in $(seq 1 "${DELETES}"); do
    n1ql=$(printf 'DELETE FROM %s USE KEYS "product::%d";' "$KEYSPACE" "$i")
    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "$n1ql" >/dev/null
  done
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Medir **antes** de compaction (REST/N1QL).

Consultarás métricas y derivarás **fragmentación%** estimada para el bucket.

#### Tarea 4.1.

- **Paso 18.** Obtén los resultados de la métrica `basicStats` del bucket (REST).

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}" | jq '.basicStats'
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 19.** Extrae los valores de las métricas `diskUsed` y `dataUsed` para calcular la fragmentación(%) aproximada.

  ```bash
  DISK=$(curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}" | jq -r '.basicStats.diskUsed')
  DATA=$(curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}" | jq -r '.basicStats.dataUsed')
  FRAG=$(python - << 'PY'
  import os
  d=int(os.environ.get("DISK",0)); u=int(os.environ.get("DATA",0))
  print(round(100*(d-u)/d,2) if d>0 else 0)
  PY
  )
  echo "Fragmentación aproximada: ${FRAG}% (diskUsed=${DISK}, dataUsed=${DATA})"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 20.** Ahora, realiza más inflaciones y mutaciones para simular la fragmentación.

  > **IMPORTANTE:** ejecuta este comando **al menos 5 veces**, pero puedes requerir un poco más.
  {: .lab-note .important .compact}

  ```bash
  KEYSPACE="\`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`"

  for r in $(seq 1 5); do
    docker exec -it "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "UPDATE ${KEYSPACE}
          SET pad = REPEAT('X', 65536), ts = NOW_MILLIS()
          WHERE META().id LIKE 'product::%';"
  done
  ```

- **Paso 21.** Realiza un borrado de un 40–60 % de la data.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq \
    -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
    -s "DELETE FROM ${KEYSPACE}
        WHERE META().id LIKE 'product::1%' OR META().id LIKE 'product::2%' OR META().id LIKE 'product::3%' OR META().id LIKE 'product::4%';"
  sleep 5
  ```

- **Paso 22.** Verifica el porcentaje de fragmentación.

  > **NOTA:** el porcentaje de la fragmentación puede ser **0 %**, aun así, continua con la compactación en la siguiente tarea.
  {: .lab-note .info .compact}

  ```bash
  DISK=$(curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}" | jq -r '.basicStats.diskUsed')
  DATA=$(curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}" | jq -r '.basicStats.dataUsed')
  FRAG=$(python - << 'PY'
  import os
  d=int(os.environ.get("DISK",0)); u=int(os.environ.get("DATA",0))
  print(round(100*(d-u)/d,2) if d>0 else 0)
  PY
  )
  echo "Fragmentación aproximada: ${FRAG}% (diskUsed=${DISK}, dataUsed=${DATA})"
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Ejecutar **compaction manual** del bucket.

Dispararás la compaction manual del bucket y observarás el progreso.

#### Tarea 5.1.

- **Paso 23.** Inicia la compaction del bucket (CLI).

  ```bash
  docker exec -it "${CB_CONTAINER}" couchbase-cli bucket-compact \
    -c "http://127.0.0.1:8091" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${APP_BUCKET}"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 24.** Verificar el estado periódicamente hasta que termine de ocupar la terminal.

  ```bash
  for i in {1..10}; do
    curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
      "http://localhost:8091/pools/default/tasks" | jq '.[] | select(.type=="bucket_compaction") | {status: .status, bucket: .bucket}'
    sleep 3
  done
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Medir **después** de compaction y comparar.

Repetirás las métricas para evidenciar la mejora.

#### Tarea 6.1.

- **Paso 25.** Repite la ejecución de la metrica `basicStats` y obtén los tamaños de discos usados.

  > **IMPORTANTE:** es normal que siga marcando 0 % de fragmentación, solo es un cálculo de referencia y depende de las configuraciones del clúster. Lo importante es que observes que la cantidad de **`diskUsed`** y **`dataUsed`** ha reducido su tamaño.
  {: .lab-note .important .compact}

  ```bash
  DISK2=$(curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}" | jq -r '.basicStats.diskUsed')
  DATA2=$(curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}" | jq -r '.basicStats.dataUsed')
  FRAG2=$(python - << 'PY'
  import os
  d=int(os.environ.get("DISK2",0)); u=int(os.environ.get("DATA2",0))
  print(round(100*(d-u)/d,2) if d>0 else 0)
  PY
  )
  echo "Fragmentación (post): ${FRAG2}% (diskUsed=${DISK2}, dataUsed=${DATA2})"
  echo "Mejora aproximada: $(python - << 'PY'
  import os
  f=float(os.environ.get("FRAG",0)); f2=float(os.environ.get("FRAG2",0)); print(round(f-f2,2))
  PY
  )%"
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 26.** Valida las consultas y ausencia de errores.

  ```bash
  docker exec -it ${CB_CONTAINER} cbq -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "SELECT COUNT(*) AS n FROM ${APP_BUCKET}.${APP_SCOPE}.${APP_COLLECTION};"
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Limpieza.

Borrar datos en el entorno para repetir pruebas.

#### Tarea 7.1.

- **Paso 27.** En la terminal, aplica el siguiente comando para detener el nodo.

  > **NOTA:** si es necesario, puedes encender de nuevo los contenedores con el comando **`docker compose start`**.
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

- **Paso 28.** Apagar y eliminar contenedor (se conservan los datos en ./data).

  > **NOTA:** si es necesario, puedes activar otra vez los contenedores con el comando **`docker compose up -d`**.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
