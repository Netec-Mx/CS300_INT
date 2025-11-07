---
layout: lab
title: "Práctica 21: Integración con Prometheus y Grafana" # CAMBIAR POR CADA PRACTICA
permalink: /lab21/lab21/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab21/img # CAMBIAR POR CADA PRACTICA
duration: "35 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar Couchbase en Docker y **exponer métricas** hacia **Prometheus**, visualizándolas en **Grafana** con un dashboard básico. Configurarás `prometheus.yml`, levantarás tres contenedores (Couchbase, Prometheus y Grafana), **validarás el scraping**, crearás **carga sintética** y construirás **paneles** para KPIs de KV/Query/Recursos.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`) `9090` (Prometheus), `3000` (Grafana).
  - Conectividad a Internet para descargar imágenes.  
  - 4–6 GB de RAM libres (Couchbase + Prometheus + Grafana).  
introduction: | # CAMBIAR POR CADA PRACTICA
  Couchbase Server expone **métricas tipo Prometheus** via HTTP en el puerto de administración. Prometheus ejecuta *scrapes* periódicos y almacena series temporales; Grafana consulta ese repositorio para construir **dashboards** interactivos. En esta práctica configurarás el scraping directo del **endpoint de métricas** de Couchbase y prepararás un panel inicial con gráficos de **operaciones KV**, **latencias**, **documentos por segundo** y **recursos**.
slug: lab21 # CAMBIAR POR CADA PRACTICA
lab_number: 21 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Levantaste un stack **Couchbase + Prometheus + Grafana**, configuraste el **scraping** del endpoint de métricas de Couchbase, generaste **carga sintética** y construiste un **dashboard** con KPIs básicos. 
notes: | # CAMBIAR POR CADA PRACTICA
  - Para producción, protege Grafana con SSO/RBAC y preserva su almacenamiento en un volumen dedicado.  
  - Considera **TLS** y *reverse proxies* si expones Prometheus/Grafana fuera de la red local.  
  - Ajusta `scrape_interval` y *retenciones* según cardinalidad y volumen.  
  - Grafana permite *alerting* sobre las queries; configura canales (email/Teams/Slack) según políticas.  
  - Si usas múltiples nodos, añade sus hostnames a `targets` o descubre dinámicamente con *service discovery*.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Prometheus — configuración básica
    url: https://prometheus.io/docs/prometheus/latest/configuration/configuration/
  - text: Grafana — provisión de datasources/dashboards
    url: https://grafana.com/docs/grafana/latest/administration/provisioning/
  - text: Couchbase — monitoreo y métricas (Prometheus/Grafana)
    url: https://www.couchbase.com/blog/es/couchbase-monitoring-integration-with-prometheus-and-grafana/

prev: /lab20/lab20/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab22/lab22/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Estructura base y variables

Crearás una carpeta aislada con subdirectorios para Couchbase, Prometheus y Grafana, además de un `.env` reutilizable.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica21-observability/
  mkdir -p practica21-observability/couchbase/{data,logs,config}
  mkdir -p practica21-observability/prometheus
  mkdir -p practica21-observability/grafana/{provisioning/{datasources,dashboards},dashboards}
  mkdir -p practica21-observability/scripts
  cd practica21-observability
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  PROMETHEUS_TAG=v2.53.0
  GRAFANA_TAG=11.2.0
  
  # Couchbase (admin)
  CB_CONTAINER=cb-obsv-n1
  CB_HOST=localhost

  # Credenciales admin
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'
  CB_NETWORK=obsnetwork

  # Servicios
  CB_SERVICES=data,index,query
  CLUSTER_RAM=3072
  INDEX_RAM=768

  # Datos de ejemplo
  APP_BUCKET=app
  APP_SCOPE=shop
  APP_COLLECTION=products
  APP_BUCKET_RAM=512

  # Prometheus/Grafana
  PROM_CONTAINER=prometheus
  GRAF_CONTAINER=grafana
  PROM_PORT=9090
  GRAF_PORT=3000
  GRAF_ADMIN=admin
  GRAF_PASS='adminlab'

  # Metricas
  CB_METRICS_PATH=/_prometheusMetrics
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Configuración de Prometheus (scrape Couchbase)

Crearás `prometheus.yml` apuntando al endpoint de métricas del contenedor de Couchbase.

#### Tarea 2.1

- **Paso 5.** Crea el archivo de prometheus llamado `prometheus/prometheus.yml` con el siguiente contenido para las metricas.

  > **NOTA:** El comando se ejecuta desde el directorio **practica21-observability**
  {: .lab-note .info .compact}

  ```bash
  cat > prometheus/prometheus.yml <<EOF
  global:
    scrape_interval: 5s
    evaluation_interval: 5s

  scrape_configs:
    # ns_server + sistema (lo que ya ves: cm_*, sys_*, couch_views_*)
    - job_name: 'couchbase-ns'
      metrics_path: /_prometheusMetrics
      params:
        type: [all]
      basic_auth:
        username: admin
        password: adminlab
      static_configs:
        - targets: ['cb-obsv-n1:8091']
          labels:
            cluster: 'lab-cs300'
            service: 'couchbase'

    # KV (por bucket)
    - job_name: 'couchbase-kv'
      metrics_path: /_prometheusMetrics
      params:
        namespace: [kv]
        bucket: [app]         # <-- tu bucket APP_BUCKET
      basic_auth:
        username: admin
        password: adminlab
      static_configs:
        - targets: ['cb-obsv-n1:8091']

    # Index
    - job_name: 'couchbase-index'
      metrics_path: /_prometheusMetrics
      params:
        namespace: [index]
      basic_auth:
        username: admin
        password: adminlab
      static_configs:
        - targets: ['cb-obsv-n1:8091']

    # Query (N1QL)
    - job_name: 'couchbase-query'
      metrics_path: /_prometheusMetrics
      params:
        namespace: [query]
      basic_auth:
        username: admin
        password: adminlab
      static_configs:
        - targets: ['cb-obsv-n1:8091']
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Docker Compose (Couchbase + Prometheus + Grafana)

Definirás `compose.yaml` para los tres servicios, con volúmenes y puertos adecuados.

#### Tarea 3.1

- **Paso 6.** Ahora crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente codigo en la terminal.

  > **NOTA:**
  - El archivo `compose.yaml` mapea puertos 8091–8096 para la consola web y 11210 para clientes.
  - El healthcheck consulta el endpoint `/pools` que responde cuando el servicio está arriba (aunque aún no inicializado).
  {: .lab-note .info .compact}

  ```bash
  cat > compose.yaml << 'YAML'
  name: couchbase-observability-lab
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

    prometheus:
      image: prom/prometheus:${PROMETHEUS_TAG}
      container_name: ${PROM_CONTAINER}
      command:
        - '--config.file=/etc/prometheus/prometheus.yml'
      volumes:
        - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      ports:
        - "${PROM_PORT}:9090"
      depends_on:
        - couchbase

    grafana:
      image: grafana/grafana:${GRAFANA_TAG}
      container_name: ${GRAF_CONTAINER}
      environment:
        - GF_SECURITY_ADMIN_USER=${GRAF_ADMIN}
        - GF_SECURITY_ADMIN_PASSWORD=${GRAF_PASS}
        - GF_USERS_ALLOW_SIGN_UP=false
      volumes:
        - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources
        - ./grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards
        - ./grafana/dashboards:/var/lib/grafana/dashboards
      ports:
        - "${GRAF_PORT}:3000"
      depends_on:
        - prometheus

  networks:
    default:
      name: ${CB_NETWORK}
      driver: bridge
  YAML
  ```

- **Paso 7.** Configura el **datasource** y **dashboard provider** para Grafana.

  ```bash
  # Datasource Prometheus
  cat > grafana/provisioning/datasources/datasource.yml << 'YAML'
  apiVersion: 1
  datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus:9090
      isDefault: true
      editable: false
  YAML

  # Provider para cargar dashboards desde carpeta
  cat > grafana/provisioning/dashboards/provider.yml << 'YAML'
  apiVersion: 1
  providers:
    - name: 'Couchbase Lab Dashboards'
      orgId: 1
      folder: 'Couchbase Lab'
      type: file
      options:
        path: /var/lib/grafana/dashboards
        recursive: true
  YAML
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Levantar el stack y validar salud

Arrancarás los servicios y validarás que Prometheus y Grafana están operativos.

#### Tarea 4.1

- **Paso 8.** Inicia el servicio, dentro de la terminal ejecuta el siguiente comando.

  > **IMPORTANTE:** Para agilizar los procesos, la imagen ya esta descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  {: .lab-note .important .compact}
  > **IMPORTANTE:** El `docker compose up -d` corre en segundo plano. El healthcheck del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en 8091 antes de continuar.
  {: .lab-note .important .compact}

  ```bash
  set -a; source .env; set +a
  docker compose up -d
  ```
  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 9.** Verifica que el contenedor se haya creado correctamente.

  {%raw%}
  ```bash
  docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
  docker inspect -f '{{.State.Health.Status}}' cb-obsv-n1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

- **Paso 10.** Valida que Prometheus este disponible y listo para trabajar.

  {%raw%}
  ```bash
  curl -fsS "http://localhost:${PROM_PORT}/-/ready" && echo " ^ Prometheus ready"
  ```
  {%endraw%}
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Inicializar Couchbase, crear datos y generar carga

Inicializarás el clúster, crearás un bucket/colección y generarás carga KV/N1QL para que las métricas cambien.

#### Tarea 5.1

- **Paso 11.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica21-observavility**. Puede tardar unos segundos en inicializar.
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
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 12.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **NOTA:**
  - Contenedor `cb-obsv-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-obsv-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 13.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${APP_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize ${APP_BUCKET_RAM} --enable-flush 1
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 14.** Ahora crea el *Scope* **shop**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes" \
    -d "name=${APP_SCOPE}"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 15.** Con este comando crea el *Collection* **products**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes/${APP_SCOPE}/collections" \
    -d "name=${APP_COLLECTION}"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 16.** Crea el índice primario temporal para validaciones

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON ${APP_BUCKET}.${APP_SCOPE}.${APP_COLLECTION};"
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 17.** Carga 500 documentos (loop con N1QL)

  > **IMPORTANTE:** Es normal que la terminal se quede en espera, ya que esta insertando los 500 registros. El proceso finalizara solo, espera unos minutos.
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
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 18.** Verifica que se hayan cargado correctamente, realiza la consulta de conteo.

  ```bash
  docker exec -i "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT COUNT(*) AS n
        FROM \`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`
        WHERE META().id LIKE 'product::%';"
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Dashboard en Grafana (provisionado) y validaciones

Crearás un dashboard JSON mínimo con gráficas típicas (KV ops rate, latencias y uso de memoria) y lo provisionarás.

#### Tarea 6.1

- **Paso 19.** Ahora define el archivo que creara el **Dashboard** para couchbase (`grafana/dashboards/couchbase-kv.json`)

  > **NOTA:** Los nombres exactos de métricas pueden variar por versión. Si alguna no aparece en Prometheus, en la UI de Prometheus usa **“/graph” → “Insert metric at cursor”**
  {: .lab-note .info .compact}

  ```bash
  cat > grafana/dashboards/couchbase-kv.json << 'JSON'
  {
    "id": null,
    "uid": "cb-lab-kv",
    "title": "Couchbase — Lab CS300 (sys/cm/views)",
    "timezone": "browser",
    "schemaVersion": 39,
    "version": 2,
    "refresh": "5s",
    "panels": [
      {
        "type": "timeseries",
        "title": "E/S de disco (bytes/s)",
        "targets": [
          {
            "expr": "sum(rate(sys_disk_read_bytes[1m]) + rate(sys_disk_write_bytes[1m]))",
            "legendFormat": "read+write"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 }
      },
      {
        "type": "timeseries",
        "title": "CPU host (%)",
        "targets": [
          {
            "expr": "100 * sum by (instance)(rate(sys_cpu_host_seconds_total{mode!=\"idle\"}[1m])) / sum by (instance)(rate(sys_cpu_host_seconds_total[1m]))",
            "legendFormat": "{{instance}}"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 }
      },
      {
        "type": "timeseries",
        "title": "Tamaño de views en disco (por bucket)",
        "targets": [
          {
            "expr": "couch_views_actual_disk_size{bucket=\"app\"}",
            "legendFormat": "{{bucket}}"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 8 }
      },
      {
        "type": "timeseries",
        "title": "Memoria usada (bytes)",
        "targets": [
          {
            "expr": "sys_mem_used_sys",
            "legendFormat": "host used"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 8 }
      }
    ]
  }
  JSON
  ```

- **Paso 20.** Tambien reiniciar sólo Grafana para cargar el dashboard correctamente.

  > **IMPORTANTE:**
  - En Grafana → *Dashboards* → *Couchbase Lab* aparece **“Couchbase — Lab CS300”**.
  - Los paneles muestran series (si alguna está vacía, ajustar la query con la métrica disponible).
  - Grafana recarga dashboards provisionados al reiniciar el contenedor o cambiar archivos.
  {: .lab-note .important .compact}

  ```bash
  docker compose restart grafana
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Visualizacion del Dashboard en Grafana

Accederas a la interfaz grafica de grafana para identificar las metricas recoletadas por Prometheus.

#### Tarea 7.1

- **Paso 21.** Ahora abre la siguiente URL para entrar al panel de **Grafana**, puedes usar Google Chrome u otro navegador disponible.

  ```bash
  http://localhost:3000
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 22.** Usa los siguientes datos para ingresar a Grafana.

  - **Usuario:** `admin`
  - **Contraseña:** `adminlab`

  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 23.** Ahora da clic en la opcion lateral izquierdo **Dashboards**
  
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 24.** Ahora da clic en la opcion lateral izquierdo **Dashboards**

- **Paso 25.** Entra a **Couchbase Lab** y luego a **Couchbase - Lab CS300...**
  
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 26.** Observaras las 4 metricas de ejemplo que recolecto Prometheus y las envio a Grafana para su visualización.

  ![cbase18]({{ page.images_base | relative_url }}/18.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8: Limpieza

Borrar datos en el entorno para repetir pruebas.

#### Tarea 8.1

- **Paso 27.** En la terminal aplica el siguiente comando para detener el nodo.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 28.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}