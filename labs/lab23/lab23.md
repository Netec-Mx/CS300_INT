---
layout: lab
title: "Práctica 23: Simulación de carga y ejection" # CAMBIAR POR CADA PRACTICA
permalink: /lab23/lab23/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab23/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar Couchbase en Docker con un bucket configurado con **eviction policy** controlada, generar **carga** hasta provocar **ejection** (expulsión de valores desde memoria a disco) y validar el comportamiento con métricas clave (memoria, resident ratio, ejections y bg fetches).
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 3–4 GB de RAM libres (servicios `kv,index,query`).  
introduction: | # CAMBIAR POR CADA PRACTICA
  Cuando la **working set** (datos activos) supera la RAM asignada a un bucket, Couchbase puede **expulsar** valores desde memoria (Value‑Only Eviction) o expulsar claves y valores (Full Eviction, con implicaciones de latencia). La **ejection** se dispara por umbrales de memoria globales (high/low watermarks). En este laboratorio forzarás la presión de memoria para observar **`vb_active_perc_mem_resident`**, **`ep_num_value_ejects`**, **`ep_bg_fetched`** y las relaciones con **`mem_used`**, **`ep_mem_high_wat`** y **`ep_mem_low_wat`**.
slug: lab23 # CAMBIAR POR CADA PRACTICA
lab_number: 23 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Provocaste **ejection** en un bucket de Couchbase ajustando la RAM y generando carga, observaste **resident ratio**, **ejections** y **bg fetches**, y aplicaste **mitigaciones** para recuperar desempeño. 
notes: | # CAMBIAR POR CADA PRACTICA
  - `fullEviction` reduce más RAM pero aumenta latencias por *fetch*; solo úsalo cuando aceptes ese trade‑off.  
  - En cargas reales, usa **SDKs** y **durabilidad/replicas** con criterio; incrementan uso de RAM.  
  - Evita índices primarios salvo en validaciones.  
  - Monitorea continuamente `vb_active_perc_mem_resident` y define alertas en Prometheus/Grafana.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Two Ejection Methods Value-only vs. Full
    url: https://www.couchbase.com/blog/a-tale-of-two-ejection-methods-value-only-vs-full/
  - text: Eviction Policies (Couchbase bucket)
    url: https://docs.couchbase.com/sdk-api/couchbase-php-client/classes/Couchbase-Management-EvictionPolicy.html
  - text: Estadísticas REST de bucket
    url: https://docs.couchbase.com/server/current/rest-api/rest-bucket-stats.html

prev: /lab22/lab22/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab24/lab24/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Carpeta de la práctica y variables

Crea estructura independiente y define variables del clúster, bucket y parámetros de carga.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica23-ejection/
  mkdir -p practica23-ejection/couchbase/{data,logs,config}
  mkdir -p practica23-ejection/scripts
  cd practica23-ejection
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-ejection-n1
  CB_HOST=couchbase1

  # Credenciales admin
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'

  # Servicios
  CB_SERVICES=data,index,query
  CLUSTER_RAM=3072
  INDEX_RAM=768


  # Datos de ejemplo
  APP_BUCKET=ejectionlab
  APP_SCOPE=shop
  APP_COLLECTION=products
  APP_BUCKET_RAM=100
  EVICTION=valueOnly 

  # Carga
  DOCS_BATCH1=20000           # primera oleada
  DOCS_BATCH2=20000           # segunda oleada (empuja a ejection)
  DOC_SIZE_BYTES=20000        # tamaño aproximado de cada doc
  READ_SAMPLES=500            # lecturas aleatorias para forzar bg fetch
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Docker Compose y salud del nodo

Definir el `compose.yaml`, levantar servicios y validar estado.

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
  docker ps --filter "name=cb-ejection-n1"
  docker inspect -f '{{.State.Health.Status}}' cb-ejection-n1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Inicializar clúster y crear bucket “estrecho”

Inicializa el clúster y crea un bucket con **RAM limitada** y **política de ejection** específica.

#### Tarea 3.1

- **Paso 8.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica23-ejection**. Puede tardar unos segundos en inicializar.
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
  - Contenedor `cb-ejection-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-ejection-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 10.** Ejecuta el siguiente comando para la creación del bucket con `--eviction-policy`.

  ```bash
  docker exec -it "${CB_CONTAINER}" couchbase-cli bucket-create \
    -c "http://127.0.0.1:8091" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${APP_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize "${APP_BUCKET_RAM}" \
    --bucket-eviction-policy "${EVICTION}" \
    --compression-mode off \
    --bucket-replica 0 \
    --enable-flush 1
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

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Cargar documentos por oleadas (hasta provocar ejection)

Insertarás dos oleadas de documentos con tamaño aproximado `${DOC_SIZE_BYTES}` para crecer el working set y cruzar los watermarks.

#### Tarea 4.1

- **Paso 14.** Ejecuta la siguiente función bash para generar un payload de tamaño fijo

  ```bash
  mkpayload() {
    python - << 'PY'
  import os, json, sys
  target = int(os.environ.get("DOC_SIZE_BYTES", "1500"))
  obj = {"type":"product","name":"Item","price":9.99,"pad":""}
  dump = lambda o: json.dumps(o, separators=(",",":"), ensure_ascii=True)
  base_len = len(dump(obj))  # con pad=""
  pad_len = max(0, target - base_len)
  obj["pad"] = "x"*pad_len
  sys.stdout.write(dump(obj))
  PY
  }
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 15.** Carga el primer **Batch #1** (no debería eyectar aún, o muy poco)

  > **IMPORTANTE:** El proceso tardara varios minutos ya que debe insertar 20000 registros. No hara el ejection aun. Calcula 4 minutos y rompe el proceso con **CTRL+C**.
  {: .lab-note .important .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`"
  for i in $(seq 1 "${DOCS_BATCH1}"); do
    VAL=$(DOC_SIZE_BYTES="${DOC_SIZE_BYTES}" mkpayload)
    n1ql=$(printf 'UPSERT INTO %s (KEY,VALUE) VALUES ("item::%d", %s);' "$KEYSPACE" "$i" "$VAL")
    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "$n1ql" >/dev/null
  done
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 16.** Revisa las métricas después o durante el Batch #1. Ejecuta los comandos 1 por 1.

  > **NOTA:** Realiza los siguientes pasos:
  - Abre otra terminal en VSC
  - Entra al directorio `cd practica23-ejection`
  - Carga las variables `set -a; source .env; set +a`
  - Ejecuta los comandos de abajo.
  {: .lab-note .info .compact}

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}" \
  | jq '{quota: .quota, basicStats: .basicStats}'
  ```
  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/stats" \
  | jq '.op.samples
        | { mem_used: (.mem_used[-1] // 0),
            vb_active_perc_mem_resident: (.vb_active_perc_mem_resident[-1] // 0) }'
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 17.** Realiza la carga del **Batch #2** (empuja a ejection)

  > **IMPORTANTE:**
  - Con ese comando se inyectara mas payload size para que simule la saturación y realice la eyección.
  - El proceso puede tardar mas de 20 minutos este laboratorio es una simulación.
  - Si deseas avanzar puedes hacerlo sin problema. En otro momento puedes dejarlo mas de 30 minutos.
  {: .lab-note .important .compact}

  ```bash
  for i in $(seq $((DOCS_BATCH1+1)) $((DOCS_BATCH1+DOCS_BATCH2))); do
    VAL=$(DOC_SIZE_BYTES="${DOC_SIZE_BYTES}" mkpayload)
    n1ql=$(printf 'UPSERT INTO %s (KEY,VALUE) VALUES ("item::%d", %s);' "$KEYSPACE" "$i" "$VAL")
    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "$n1ql" >/dev/null
  done
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Forzar lecturas aleatorias y observar **bg fetch**

Realiza lecturas aleatorias; si el documento fue expulsado de RAM, el valor se trae desde disco (**background fetch**).

#### Tarea 5.1

- **Paso 18.** Realiza lecturas aleatorias para observar el comportamiento.

  > **IMPORTANTE:** Si dejaste en ejecucion el comando anterior, realiza los siguientes pasos:
  - Abre otra terminal en VSC
  - Entra al directorio `cd practica23-ejection`
  - Carga las variables `set -a; source .env; set +a`
  {: .lab-note .important .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`"
  for i in $(shuf -i 1-$((DOCS_BATCH1 + DOCS_BATCH2)) -n "${READ_SAMPLES}"); do
    n1ql=$(printf 'SELECT META().id, SUBSTR(pad,1,4) AS hint FROM %s USE KEYS "item::%d";' \
                  "$KEYSPACE" "$i")
    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "$n1ql" >/dev/null
  done
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 19.** Observa las métricas de ejection y bg fetch

  > **IMPORTANTE:** Abre una tercera terminal y realiza los siguientes pasos:
  - Abre otra terminal en VSC
  - Entra al directorio `cd practica23-ejection`
  - Carga las variables `set -a; source .env; set +a`
  {: .lab-note .important .compact}

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/stats" \
  | jq '.op.samples
        | {
            mem_used: (.mem_used[-1] // 0),
            vb_active_perc_mem_resident: (.vb_active_perc_mem_resident[-1] // 0),
            ep_num_value_ejects: (.ep_num_value_ejects[-1] // 0),
            ep_bg_fetched: (.ep_bg_fetched[-1] // 0)
          }'
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Acciones de mitigación.

Aplicarás mitigaciones comunes: aumentar RAM del bucket o eliminar datos fríos, y compararás efectos en métricas.

#### Tarea 6.1

- **Paso 20.** Aumentar RAM del bucket (mitigación 1)

  ```bash
  # Subir RAM a 512MB
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-edit \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${APP_BUCKET}" --bucket-ramsize 512
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 21.** Eliminar registros fríos (mitigación 2)

  > **IMPORTANTE:** Este comando puede tardar varios minutos. Espera 3 minutos y ejecuta `CTRL + C`
  {: .lab-note .important .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`"
  START=$((DOCS_BATCH1 + (DOCS_BATCH2 * 3 / 4)))
  END=$((DOCS_BATCH1 + DOCS_BATCH2))

  for i in $(seq "$START" "$END"); do
    n1ql=$(printf 'DELETE FROM %s USE KEYS "item::%d";' "$KEYSPACE" "$i")
    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "$n1ql" >/dev/null
  done
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 22.** Obten de nuevo las metricas.

  > **IMPORTANTE:** El restultado puede variar recuerda que esta es una practica simulada.
  {: .lab-note .important .compact}

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/stats" | jq '.op.samples | {
      mem_used: .mem_used[-1],
      vb_active_perc_mem_resident: .vb_active_perc_mem_resident[-1],
      ep_num_value_ejects: .ep_num_value_ejects[-1],
      ep_bg_fetched: .ep_bg_fetched[-1]
    }'
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Limpieza

Borrar datos en el entorno para repetir pruebas.

#### Tarea 7.1

- **Paso 23.** En la terminal aplica el siguiente comando para detener el nodo.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 24.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}