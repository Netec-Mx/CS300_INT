---
layout: lab
title: "Práctica 16: Replicación unidireccional" # CAMBIAR POR CADA PRACTICA
permalink: /lab16/lab16/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab16/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar **dos clústeres** Couchbase en una misma máquina (Docker) y configurar **XDCR unidireccional** del clúster **Origen → Destino**. Crearás buckets, configurarás la referencia remota, iniciarás la replicación, probarás inserciones y validarás la llegada de los datos.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 6–8 GB de RAM libres (2×2 nodos con `data,index,query`).  
introduction: | # CAMBIAR POR CADA PRACTICA
  **XDCR** (Cross Data Center Replication) replica de manera continua datos entre buckets de distintos clústeres. En esta práctica levantarás **dos clústeres independientes** en Docker (Origen y Destino), crearás un **bucket** con **colección** en ambos, configurarás la **referencia remota** en Origen y activarás una **replicación unidireccional**. Luego insertarás documentos en Origen y verificarás su llegada al Destino usando N1QL.
slug: lab16 # CAMBIAR POR CADA PRACTICA
lab_number: 16 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Instalaste **dos clústeres Couchbase**, configuraste **XDCR unidireccional** (Origen → Destino), replicaste documentos. Validaste en Destino mediante consultas N1QL que los datos llegaron correctamente.
notes: | # CAMBIAR POR CADA PRACTICA
  - Para producción, activa **TLS** en XDCR y usa **Alternate Addresses** si hay balanceadores o redes mixtas.  
  - Ajusta **compresión**, **lotes** y **límites de conexiones** según latencia/throughput.  
  - **Filtros de replicación** (por colección o expresión) permiten seleccionar subconjuntos cuando el destino no debe recibir todo.  
  - Si cambias nombres de scope/collection, verifica mapeos de colecciones en la replicación.  
  - Usa la UI (*XDCR*) para ver métricas de envío/recepción y lag.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Conceptos y operación de XDCR
    url: https://docs.couchbase.com/server/current/learn/clusters-and-availability/xdcr-overview.html 
  - text: CLI xdcr-setup / xdcr-replicate
    url: https://docs.couchbase.com/server/current/cli/cbcli/couchbase-cli-xdcr-setup.html
  - text: Replicación entre centros de datos (XDCR)
    url: https://www.couchbase.com/blog/es/couchbase-replication-and-sync/
  - text: Puertos de Couchbase (útil para mapeos)
    url: https://docs.couchbase.com/server/current/install/install-ports.html
prev: /lab15/lab15/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab17/lab17/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Carpeta de práctica y variables

Crear una carpeta para la práctica y un `.env` con variables para ambos clústeres, buckets y colecciones.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica16-xdcr-uni/
  mkdir -p practica16-xdcr-uni/cbs/{data1,data2,logs,config}
  mkdir -p practica16-xdcr-uni/cbd/{data1,data2,logs,config}
  mkdir -p practica16-xdcr-uni/scripts
  cd practica16-xdcr-uni
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise

  # --- Cluster ORIGEN (Source) ---
  SRC_NODE1=cb-src-n1
  SRC_NODE2=cb-src-n2
  SRC_HOST1=couchbases1
  SRC_HOST2=couchbases2
  SRC_UI_RANGE=18091-18096      # se mapea al 8091-8096 internos
  SRC_QUERY_PORT=18093          # opcional: acceso directo a 8093 interno
  SRC_KV_PORT=11211             # KV externo opcional (si lo requieres distinto a 11210)

  # --- Cluster DESTINO (Target) ---
  DST_NODE1=cb-dst-n1
  DST_NODE2=cb-dst-n2
  DST_HOST1=couchbased1
  DST_HOST2=couchbased2
  DST_UI_RANGE=28091-28096
  DST_QUERY_PORT=28093
  DST_KV_PORT=11212

  # Credenciales admin
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'
  CB_NETWORK=net-xdcr

  # Servicios por nodo
  CB_SERVICES=data,index,query
  SRC_CLUSTER_RAM=2048
  SRC_INDEX_RAM=512
  DST_CLUSTER_RAM=2048
  DST_INDEX_RAM=512

  # Bucket/Scope/Collection
  CB_BUCKET=app
  CB_BUCKET_RAM=512
  CB_SCOPE=shop
  CB_COLLECTION=products
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Docker Compose con **dos** clústeres

Definir un `compose.yaml` que levante **2×2 nodos** (Origen y Destino) con *healthchecks* y puertos diferenciados.

#### Tarea 2.1

- **Paso 5.** Ahora crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente codigo en la terminal.

  > **NOTA:**
  - El archivo `compose.yaml` mapea puertos 8091–8096 para la consola web y 11210 para clientes.
  - El healthcheck consulta el endpoint `/pools` que responde cuando el servicio está arriba (aunque aún no inicializado).
  {: .lab-note .info .compact}

  ```bash
  cat > compose.yaml << 'YAML'
  name: couchbase-xdcr-lab
  services:
    # --------- CLUSTER ORIGEN ----------
    src-n1:
      image: couchbase/server:${CB_TAG}
      container_name: ${SRC_NODE1}
      hostname: ${SRC_HOST1}
      ports:
        - "${SRC_UI_RANGE}:8091-8096"
        - "${SRC_KV_PORT:-11210}:11210"
      environment:
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      volumes:
        - ./cbs/data1:/opt/couchbase/var
        - ./cbs/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./cbs/config:/config
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 20

    src-n2:
      image: couchbase/server:${CB_TAG}
      container_name: ${SRC_NODE2}
      hostname: ${SRC_HOST2}
      environment:
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      volumes:
        - ./cbs/data2:/opt/couchbase/var
        - ./cbs/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./cbs/config:/config
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 20

    # --------- CLUSTER DESTINO ----------
    dst-n1:
      image: couchbase/server:${CB_TAG}
      container_name: ${DST_NODE1}
      hostname: ${DST_HOST1}
      ports:
        - "${DST_UI_RANGE}:8091-8096"
        - "${DST_KV_PORT:-11210}:11210"
      environment:
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      volumes:
        - ./cbd/data1:/opt/couchbase/var
        - ./cbd/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./cbd/config:/config
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 20

    dst-n2:
      image: couchbase/server:${CB_TAG}
      container_name: ${DST_NODE2}
      hostname: ${DST_HOST2}
      environment:
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      volumes:
        - ./cbd/data2:/opt/couchbase/var
        - ./cbd/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./cbd/config:/config
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 20

  networks:
    default:
      name: ${CB_NETWORK}
      driver: bridge
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

- **Paso 7.** Verifica la salud de los contenedores que se hayan creado correctamente.

  {%raw%}
  ```bash
  docker inspect -f '{{.State.Health.Status}}' cb-src-n1
  docker inspect -f '{{.State.Health.Status}}' cb-src-n2
  docker inspect -f '{{.State.Health.Status}}' cb-dst-n1
  docker inspect -f '{{.State.Health.Status}}' cb-dst-n2
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Inicializar clúster **Origen** (2 nodos)

Crea el clúster en `src-n1`, agrega `src-n2`, rebalancea y preparar el keyspace.

#### Tarea 3.1

- **Paso 8.** Inicializa el clúster con el nodo **src-n1**, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica16-xdcr-uni**. Puede tardar unos segundos en inicializar.
  {: .lab-note .important .compact}

  ```bash
  set -a; source .env; set +a
  docker exec -it ${SRC_NODE1} couchbase-cli cluster-init \
    --cluster ${SRC_HOST1} \
    --cluster-username "${CB_ADMIN}" \
    --cluster-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}" \
    --cluster-ramsize ${SRC_CLUSTER_RAM} \
    --cluster-index-ramsize ${SRC_INDEX_RAM} \
    --index-storage-setting default
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

- **Paso 9.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **NOTA:**
  - Contenedor `cb-src-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-src-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:18091/pools | head -c 200; echo
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 10.** Ahora agrega al nodo **src-n2**.

  {%raw%}
  ```bash
  SN2_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-src-n2)
  docker exec -it ${SRC_NODE1} couchbase-cli server-add \
    -c ${SRC_HOST1} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-add ${SN2_IP} \
    --server-add-username "${CB_ADMIN}" --server-add-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}"
  ```
  {%endraw%}
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 11.** Realiza el **rebalance** de los nodos.

  {%raw%}
  ```bash
  SNIP1=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-src-n1)
  docker exec -it ${SRC_NODE1} couchbase-cli rebalance \
    -c ${SNIP1} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ``` 
  {%endraw%}
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 12.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${SRC_NODE1} couchbase-cli bucket-create \
    -c ${SRC_HOST1} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${CB_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize ${CB_BUCKET_RAM} --enable-flush 1
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 13.** Ahora crea el *Scope* **shop**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:18091/pools/default/buckets/${CB_BUCKET}/scopes" \
    -d "name=${CB_SCOPE}"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 14.** Con este comando crea el *Collection* **products**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:18091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLLECTION}"
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Inicializar clúster **Destino** (2 nodos)

Repetir pasos equivalentes para el **Destino** y crear el bucket/colección destino.

#### Tarea 4.1

- **Paso 15.** Inicializa el clúster con el nodo **dst-n1**, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica16-xdcr-uni**. Puede tardar unos segundos en inicializar.
  {: .lab-note .important .compact}

  ```bash
  set -a; source .env; set +a
  docker exec -it ${DST_NODE1} couchbase-cli cluster-init \
    --cluster ${DST_HOST1} \
    --cluster-username "${CB_ADMIN}" \
    --cluster-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}" \
    --cluster-ramsize ${DST_CLUSTER_RAM} \
    --cluster-index-ramsize ${DST_INDEX_RAM} \
    --index-storage-setting default
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 16.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **NOTA:**
  - Contenedor `cb-dst-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-dst-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:28091/pools | head -c 200; echo
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 17.** Ahora agrega al nodo **dst-n2**.

  {%raw%}
  ```bash
  DN2_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-dst-n2)
  docker exec -it ${DST_NODE1} couchbase-cli server-add \
    -c ${DST_HOST1} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-add ${DN2_IP} \
    --server-add-username "${CB_ADMIN}" --server-add-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}"
  ```
  {%endraw%}
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 18.** Realiza el **rebalance** de los nodos.

  {%raw%}
  ```bash
  DNIP1=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-dst-n1)
  docker exec -it ${DST_NODE1} couchbase-cli rebalance \
    -c ${DNIP1} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ``` 
  {%endraw%}
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 19.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${DST_NODE1} couchbase-cli bucket-create \
    -c ${DST_HOST1} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${CB_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize ${CB_BUCKET_RAM} --enable-flush 1
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 20.** Ahora crea el *Scope* **shop**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:28091/pools/default/buckets/${CB_BUCKET}/scopes" \
    -d "name=${CB_SCOPE}"
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 21.** Con este comando crea el *Collection* **products**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:28091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLLECTION}"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Configurar **XDCR** unidireccional (Origen → Destino)

Crear una **referencia remota** hacia el Destino desde el **Origen**, y luego crear la replicación **bucket→bucket** (collection-aware por defecto en Server moderno).

#### Tarea 5.1

- **Paso 22.** Crea la **referencia remota** en el nodo Origen.

  ```bash
  docker exec -it ${SRC_NODE1} couchbase-cli xdcr-setup \
    -c ${SRC_HOST1} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --create \
    --xdcr-cluster-name dst-cluster \
    --xdcr-hostname ${DST_HOST1} \
    --xdcr-username "${CB_ADMIN}" \
    --xdcr-password "${CB_ADMIN_PASS}"
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 23.** Crea la **replicación** bucket→bucket

  ```bash
  docker exec -it ${SRC_NODE1} couchbase-cli xdcr-replicate \
    -c ${SRC_HOST1} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --create \
    --xdcr-cluster-name dst-cluster \
    --xdcr-from-bucket ${CB_BUCKET} \
    --xdcr-to-bucket ${CB_BUCKET} \
    --enable-compression 1
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 24.** Verfica la replicación con el siguiente comando. **Copia el valor de la propiedad `stream id:` lo usaras en el siguiente comando.**

  ```bash
  docker exec -it ${SRC_NODE1} couchbase-cli xdcr-replicate \
    -c ${SRC_HOST1} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" --list
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

- **Paso 25.** Obten los detalles de la replica. **Sustituye la frase `STREAM-ID` por el valor que copiaste del paso anterior.**

  > **IMPORTANTE:** Puedes apoyarte de la imagen, no elimines las comillas. El resultado es muy extenso la imagen representa una parte.
  {: .lab-note .important .compact}

  ```bash
  docker exec -it ${SRC_NODE1} couchbase-cli xdcr-replicate \
    -c ${SRC_HOST1} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" --get \
    --xdcr-cluster-name dst-cluster --xdcr-from-bucket ${CB_BUCKET} --xdcr-replicator "STREAM-AQUI"
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Pruebas de inserción y verificación en Destino

Insertar documentos en Origen y verificarlos en Destino (N1QL). Incluye escapes de backticks para Git Bash.

#### Tarea 6.1

- **Paso 26.** Inserta los siguientes datos desde el **Origen** 

  ```bash
  docker exec -it "${SRC_NODE1}" cbq -e "http://${SRC_HOST1}:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "UPSERT INTO \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` (KEY,VALUE) VALUES
        ('prod::101', {\"type\":\"product\", \"name\":\"XDCR Test Lamp\", \"category\":\"lighting\", \"price\":29.99, \"ts\":NOW_MILLIS()}),
        ('prod::102', {\"type\":\"product\", \"name\":\"XDCR Cable\",     \"category\":\"hardware\", \"price\": 4.99, \"ts\":NOW_MILLIS()});"
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

- **Paso 27.** Ahora consulta los datos en el **Destino** 

  > **NOTA:** La replicación puede tardar unos momentos.
  {: .lab-note .info .compact}

  ```bash
  sleep 10
  docker exec -it "${DST_NODE1}" cbq -e "http://${DST_HOST1}:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT META().id, name, category, price, ts
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`
        WHERE META().id IN ['prod::101','prod::102'];"
  ```
  ![cbase24]({{ page.images_base | relative_url }}/24.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Limpieza

Borrar datos en el entorno para repetir pruebas.

#### Tarea 7.1

- **Paso 28.** En la terminal aplica el siguiente comando para detener el nodo.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase25]({{ page.images_base | relative_url }}/25.png)

- **Paso 29.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase26]({{ page.images_base | relative_url }}/26.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
