---
layout: lab
title: "Práctica 14: FTS y búsqueda textual" # CAMBIAR POR CADA PRACTICA
permalink: /lab14/lab14/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab14/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar Couchbase en Docker y dominar **Full Text Search (FTS)** creando un índice textual sobre una colección, cargando documentos con descripciones y etiquetas, y ejecutando búsquedas con **REST FTS** y con **N1QL `SEARCH()`** (incluyendo *highlighting* y validaciones de resultados).
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.
introduction: | # CAMBIAR POR CADA PRACTICA
  **FTS** (Full Text Search) de Couchbase usa índices invertidos para búsquedas de texto libre (coincidencia, *analyzers*, *stemming*, *fuzziness*, *highlighting*). En esta práctica crearás un índice FTS de colección para `products`, probarás consultas por frase y términos, filtrarás por campos y usarás **`SEARCH()`** desde N1QL para combinar texto libre con filtros estructurados.
slug: lab14 # CAMBIAR POR CADA PRACTICA
lab_number: 14 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Has creado un índice **FTS** para una colección, ejecutado búsquedas por **REST** y **N1QL `SEARCH()`**, probado filtros combinados y *highlighting*, e incluso consultas *fuzzy* y *prefix*. Quedas listo para integrar texto libre con filtros estructurados en aplicaciones reales.
notes: | # CAMBIAR POR CADA PRACTICA
  - Para colecciones grandes, planifica *sizing* de FTS, *numReplicas* y *partitioning*.  
  - Ajusta *analyzers* al idioma dominante de tus textos (p. ej., *spanish*).  
  - Monitorea `/api/index/<idx>/stats` y tareas en la UI para ver progreso de *build*.  
  - `SEARCH()` permite *pushdown* de predicados y ordenamientos que combinan KV+FTS.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: FTS — conceptos y administración
    url: https://docs.couchbase.com/server/current/fts/fts-introduction.html   
  - text: La propiedad SEARCH() en N1QL
    url: https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/searchfun.html  
  - text: Analyzers y tipos de consultas FTS
    url: https://docs.couchbase.com/server/current/fts/fts-supported-queries.html
prev: /lab13/lab13/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab15/lab15/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Carpeta de práctica y variables

Crearás una carpeta aislada con variables estándar.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica14-fts/
  mkdir -p practica14-fts/couchbase/{data,logs,config}
  mkdir -p practica14-fts/scripts
  cd practica14-fts
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-fts-n1
  CB_HOST=127.0.0.1
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'
  CB_SERVICES=data,index,query,fts
  CB_RAM=2048
  CB_INDEX_RAM=512

  CB_BUCKET=shop
  CB_BUCKET_RAM=512
  CB_SCOPE=inv
  CB_COLLECTION=products

  # Índice FTS
  FTS_INDEX=idx_products_fts
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Docker Compose y salud del nodo

Definirás un `compose.yaml` con un nodo único que expone `data,index,query,fts` y su *healthcheck*.

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
      hostname: couchbase1
      ports:
        - "8091-8096:8091-8096"   # Web UI / servicios
        - "11210:11210"           # Memcached/SDK (data)
        - "8094:8094"             # FTS REST API
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
  docker ps --filter "name=cb-fts-n1"
  docker inspect -f '{{.State.Health.Status}}' cb-fts-n1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Inicializar clúster, keyspace y datos

Inicializarás el clúster, crearás bucket/scope/collection y cargarás documentos con campos textuales.

#### Tarea 3.1

- **Paso 8.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica14-fts**. Puede tardar unos segundos en inicializar.
  {: .lab-note .important .compact}

  ```bash
  set -a; source .env; set +a
  docker exec -it "$CB_CONTAINER" couchbase-cli cluster-init \
    --cluster 127.0.0.1 \
    --cluster-username "$CB_ADMIN" \
    --cluster-password "$CB_ADMIN_PASS" \
    --services "${SERVICES_NORM:-$CB_SERVICES}" \
    --cluster-ramsize "$CB_RAM" \
    --cluster-index-ramsize "$CB_INDEX_RAM" \
    --index-storage-setting default
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

- **Paso 9.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **NOTA:**
  - Contenedor `cb-index-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-fts-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 10.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${CB_BUCKET}" \
    --bucket-type couchbase \
    --bucket-ramsize ${CB_BUCKET_RAM} \
    --enable-flush 1
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 11.** Ahora crea el *Scope*.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" \
    -d "name=${CB_SCOPE}"
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 12.** Con este comando crea el *Collection*

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLLECTION}"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 13.** Cargar datos de ejemplo (12 documentos)

  > **IMPORTANTE:** El resultado es iterativo para los 10 documentos la salida es muy grande, la imagen representa la sección de la inserción.
  {: .lab-note .important .compact}

  ```bash
  #!/usr/bin/env bash
  set -e

  brands=("Glowco" "Lumina" "Voltix" "ErgoWorks")
  status="active"

  for i in $(seq 1 12); do
    case $i in
      1)  name="Wireless Lamp";       desc="Modern wireless lamp with warm light and touch controls"; tags='["lighting","wireless","home"]';        category="lighting" ;;
      2)  name="Reading Lamp";        desc="Compact reading lamp ideal for desks and bedside tables"; tags='["lighting","reading","desk"]';          category="lighting" ;;
      3)  name="Table Lamp";          desc="Minimalist table lamp suitable for offices and studios";  tags='["lighting","office","minimal"]';        category="lighting" ;;
      4)  name="Smart Bulb";          desc="Smart LED bulb compatible with assistants and dimmers";   tags='["lighting","smart","bulb"]';            category="lighting" ;;
      5)  name="Standing Lamp";       desc="Tall standing lamp with adjustable head and soft shade";  tags='["lighting","floor","adjustable"]';      category="lighting" ;;
      6)  name="Desk Organizer";      desc="Wooden organizer for pens and notebooks";                 tags='["office","wood","accessories"]';        category="office" ;;
      7)  name="USB-C Cable";         desc="Durable braided cable for phones and tablets";            tags='["hardware","cable","usb-c"]';           category="hardware" ;;
      8)  name="Ergonomic Chair";     desc="Ergonomic office chair with lumbar support";              tags='["office","chair","ergonomic"]';         category="office" ;;
      9)  name="Wireless Charger";    desc="Fast wireless charger pad for smartphones";               tags='["hardware","wireless","charger"]';      category="hardware" ;;
      10) name="LED Strip";           desc="Customizable RGB LED strip for ambient lighting";         tags='["lighting","rgb","ambient"]';           category="lighting" ;;
      11) name="Desk Lamp Pro";       desc="High brightness desk lamp with color temperature modes";  tags='["lighting","desk","pro"]';              category="lighting" ;;
      12) name="Clip Lamp";           desc="Clip-on lamp perfect for bookshelves and bunk beds";      tags='["lighting","clip","portable"]';         category="lighting" ;;
    esac

    brand=${brands[$(( (i-1) % ${#brands[@]} ))]}
    price=$(awk -v i="$i" 'BEGIN{printf "%.2f", (i*7)%200 + 19.99}')

    # --- JSON puro: TODAS las claves con comillas dobles ---
    n1ql=$(cat <<EOF
  UPSERT INTO \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` (KEY, VALUE)
  VALUES ("prod::${i}", {
    "type": "product",
    "name": "${name}",
    "description": "${desc}",
    "category": "${category}",
    "price": ${price},
    "status": "${status}",
    "brand": "${brand}",
    "tags": ${tags}
  });
  EOF
  )

    docker exec -i "${CB_CONTAINER}" cbq \
      -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
      -s "$n1ql"
  done
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 14.** Verificación rápida (índice primario temporal)

  > **IMPORTANTE:** El resultado seran 5 productos pero es un poco grande, la imagen presenta una parte de la salida
  {: .lab-note .important .compact}

  ```bash
  docker exec -i "${CB_CONTAINER}" cbq -e http://127.0.0.1:8093 \
    -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "SELECT META().id, name, type, price
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`
        WHERE type = \"product\"
        ORDER BY META().id
        LIMIT 5;"
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 15.** Valida la cantidad de los documentos insertados con el siguiente comando.

  ```bash
  docker exec -i "${CB_CONTAINER}" cbq -e http://127.0.0.1:8093 \
    -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "SELECT COUNT(*) AS total
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`
        WHERE type = \"product\";"
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Crear índice FTS por REST (colección)

Crearás un índice FTS apuntando a la **colección** `products`, con *default mapping* habilitado y un *simple analyzer*.

#### Tarea 4.1

- **Paso 16.** Crea un archivo JSON con la definición del índice FTS apuntando a bucket/scope/collection específicos.

  ```bash
  cat > scripts/${FTS_INDEX}.json << 'JSON'
  {
    "type": "fulltext-index",
    "name": "IDX_PLACEHOLDER",
    "sourceType": "couchbase",
    "sourceName": "BUCKET_PLACEHOLDER",
    "planParams": {
      "numReplicas": 0
    },
    "sourceParams": {
      "scope": "SCOPE_PLACEHOLDER",
      "collections": ["COLLECTION_PLACEHOLDER"]
    },
    "params": {
      "doc_config": {
        "mode": "scope.collection.type_field",
        "type_field": "type"
      },
      "mapping": {
        "default_mapping": {
          "enabled": true,
          "dynamic": true
        },
        "types": {},
        "default_analyzer": "standard"
      },
      "store": {
        "indexType": "scorch"
      }
    }
  }
  JSON
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 17.** Rellena el JSON sustituyendo los placeholders por tus variables reales.

  ```bash
  sed "s/IDX_PLACEHOLDER/${FTS_INDEX}/g; s/BUCKET_PLACEHOLDER/${CB_BUCKET}/g; s/SCOPE_PLACEHOLDER/${CB_SCOPE}/g; s/COLLECTION_PLACEHOLDER/${CB_COLLECTION}/g" \
    scripts/${FTS_INDEX}.json > scripts/${FTS_INDEX}.rendered.json
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 18.** Crea/actualiza el índice en el servicio FTS (puerto 8094) mediante REST PUT.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -H "Content-Type: application/json" \
    -X PUT "http://localhost:8094/api/index/${FTS_INDEX}" \
    -d @scripts/${FTS_INDEX}.rendered.json
  echo
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 19.** Consulta /stats del índice; deberías ver métricas iniciales (document count, pindexes, entre otros). Si ya hay documentos en products, el conteo empezará a subir conforme FTS procese el build inicial.

  ```bash
  for i in {1..5}; do
    curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
      "http://localhost:8094/api/index/${FTS_INDEX}/count" ; echo
    sleep 2
  done
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Consultas FTS por REST

Probarás consultas FTS por REST (con *explain* y *highlighting*) y con N1QL `SEARCH()` combinando texto con filtros estructurados.

#### Tarea 5.1

- **Paso 20.** Busca documentos que contengan los términos “wireless” y “lamp”, devuelve explicación del score y resalta coincidencias en name y description.

  > **IMPORTANTE:** El resultado es un poco grande, la imagen representa una parte de la salida
  {: .lab-note .important .compact}

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" -H "Content-Type: application/json" \
    -X POST "http://localhost:8094/api/index/${FTS_INDEX}/query" \
    -d '{
          "query": { "match": "wireless lamp" },
          "size": 5,
          "explain": true,
          "highlight": { "style":"html", "fields":["name","description"] }
        }' | jq '.status,.total_hits,.hits[0:3]'
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 21.** Búsqueda por frase exacta (query string con comillas). Usa el parser de query string para exigir la frase exacta "desk lamp".

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" -H "Content-Type: application/json" \
    -X POST "http://localhost:8094/api/index/${FTS_INDEX}/query" \
    -d '{
          "query": { "query": "\"desk lamp\"" },
          "size": 5,
          "highlight": { "style":"html", "fields":["name","description"] }
        }' | jq '.total_hits, (.hits[] | {id, fragments})'
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 22.** Búsqueda con filtro por campo (consulta booleana; REST). Interseca la frase "desk lamp" con tags:lighting.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" -H "Content-Type: application/json" \
    -X POST "http://localhost:8094/api/index/${FTS_INDEX}/query" \
    -d '{"query":{"conjuncts":[{"match_phrase":"desk lamp"},{"term":"lighting","field":"tags"}]},"size":5}' | jq '.total_hits,.hits[].id'
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Ajustes del índice y pruebas adicionales

Explorarás *analyzers* y *fuzziness* para tolerancia a errores tipográficos.

#### Tarea 6.1

- **Paso 23.** Consulta *fuzzy* (tolerancia a 1 edición). Busca términos similares a “wireless” permitiendo 1 edición (inserción/eliminación/sustitución).

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" -H "Content-Type: application/json" \
    -X POST "http://localhost:8094/api/index/${FTS_INDEX}/query" \
    -d '{
          "query": { "match": "wirless", "fuzziness": 1, "analyzer": "simple" },
          "size": 5,
          "highlight": { "style":"html", "fields":["name","description"] }
        }' | jq '.total_hits, (.hits[] | {id, fragments})'
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 24.** *Prefix query* para autocompletar. Devuelve documentos con términos que empiezan con lam (ej. lamp, lamps).

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" -H "Content-Type: application/json" \
    -X POST "http://localhost:8094/api/index/${FTS_INDEX}/query" \
    -d '{
          "query": { "prefix": "lam" },
          "size": 5,
          "highlight": { "style":"html", "fields":["name"] }
        }' | jq '.total_hits, (.hits[] | {id})'
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Limpieza

Borrar datos en el entorno para repetir pruebas.

#### Tarea 7.1

- **Paso 25.** En la terminal aplica el siguiente comando para detener el nodo.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

- **Paso 26.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}