---
layout: lab
title: "Práctica 13: Creación de índices secundarios/covering" # CAMBIAR POR CADA práctica
permalink: /lab13/lab13/ # CAMBIAR POR CADA práctica
images_base: /labs/lab13/img # CAMBIAR POR CADA práctica
duration: "25 minutos" # CAMBIAR POR CADA práctica
objective: # CAMBIAR POR CADA práctica
  - Desplegar Couchbase en Docker y dominar la creación y verificación de **índices secundarios** (compuestos, parciales y funcionales) y **índices covering** (que evitan el *FETCH*), midiendo su efecto en planes de ejecución **EXPLAIN** y en consultas N1QL.
prerequisites:  # CAMBIAR POR CADA práctica
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.
introduction: | # CAMBIAR POR CADA práctica
  N1QL usa **índices secundarios** para filtrar y ordenar eficientemente. Un **índice covering** permite responder la consulta **sin leer el documento** (sin operador *FETCH*), siempre que **todas** las columnas referenciadas (predicados, proyecciones, ORDER BY) estén contenidas en el índice (como **llaves** o en **INCLUDE**). También existen **índices parciales** (con *WHERE*) para subconjuntos y **funcionales** (expresiones como `LOWER(name)`). En está práctica crearás cada tipo y comprobarás su uso con **EXPLAIN**.
slug: lab13 # CAMBIAR POR CADA práctica
lab_number: 13 # CAMBIAR POR CADA práctica
final_result: | # CAMBIAR POR CADA práctica
  Has creado y validado **índices secundarios** (compuesto, parcial y funcional) y un **índice covering** con `INCLUDE`, comprobando con **EXPLAIN** el uso de cada índice y la ausencia de *Fetch* cuando corresponde. También probaste `USE INDEX` para forzar planes alternos. Quedas listo para diseñar estrategias de indexación eficientes en N1QL.
notes: | # CAMBIAR POR CADA práctica
  - Para *covering* perfecto con `ORDER BY`, incluye la columna de ordenamiento en las **keys** del índice (no solo en `INCLUDE`).  
  - Evita depender del **índice primario**; úsalo solo para diagnósticos.  
  - Mantén los índices **lo más específicos posible** para tus consultas críticas.  
  - Monitorea el tamaño y tiempo de *build* de cada índice, especialmente en colecciones grandes.
references: # CAMBIAR POR CADA práctica LINKS ADICIONALES DE DOCUMENTACION
  - text: Índices secundarios y covering en N1QL
    url: https://www.couchbase.com/blog/es/introduction-to-covering-indexes/?facet=blog%2Fblogs.facets%2FYear%2F2015%2FMonth%2F11
  - text: Índices parciales y funcionales
    url: https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/createindex.html#partial-indexes  
  - text: EXPLAIN
    url: https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/explain.html
prev: /lab12/lab12/ # CAMBIAR POR CADA práctica MENU DE NAVEGACION HACIA ATRAS        
next: /lab14/lab14/ # CAMBIAR POR CADA práctica MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Carpeta de práctica y variables

Crearás una carpeta aislada y un `.env` con variables estándar.



- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**.

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica13-indexes/
  mkdir -p practica13-indexes/couchbase/{data,logs,config}
  mkdir -p practica13-indexes/scripts
  cd practica13-indexes
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **Nota.** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-index-n1
  CB_HOST=127.0.0.1
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'
  CB_SERVICES=data,index,query
  CB_RAM=2048
  CB_INDEX_RAM=512

  CB_BUCKET=shop
  CB_BUCKET_RAM=512
  CB_SCOPE=inv
  CB_COLLECTION=products
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Docker Compose y salud del nodo

Definirás y levantarás un `compose.yaml` con un nodo que expone UI y servicios `data,index,query`.



- **Paso 1.** Ahora crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente código en la terminal.

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
      hostname: couchbase1
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

- **Paso 2.** Inicia el servicio, dentro de la terminal ejecuta el siguiente comando.

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
  docker ps --filter "name=cb-index-n1"
  docker inspect -f '{{.State.Health.Status}}' cb-index-n1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Inicializar clúster, keyspace y datos

Inicializarás el clúster, crearás bucket/scope/collection y cargarás datos de muestra para pruebas de índices.



- **Paso 1.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **Nota.** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **Importante.** El comando se ejecuta desde el directorio de la práctica **practica13-indexes**. Puede tardar unos segundos en inicializar.
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

- **Paso 2.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **Nota.**
  - Contenedor `cb-index-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - está conexión es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-index-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 3.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${CB_BUCKET}" \
    --bucket-type couchbase \
    --bucket-ramsize ${CB_BUCKET_RAM} \
    --enable-flush 1
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 4.** Ahora crea el *Scope*.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" \
    -d "name=${CB_SCOPE}"
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 5.** Con este comando crea el *Collection*.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLLECTION}"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 6.** Cargar datos de ejemplo (10 documentos).

  > **Importante.** El resultado es iterativo para los 10 documentos la salida es muy grande, la imagen representa la sección de la inserción.
  {: .lab-note .important .compact}

  ```bash
  for i in 1 2 3 4 5 6 7 8 9 10; do
    case $i in
      1)  name="Widget";     catg="hardware";  price=19.99; status="active";   brand="Acme"   ;;
      2)  name="Gadget";     catg="hardware";  price=29.50; status="active";   brand="Acme"   ;;
      3)  name="Doohickey";  catg="hardware";  price=9.99;  status="inactive"; brand="Omni"   ;;
      4)  name="Lamp";       catg="lighting";  price=14.75; status="active";   brand="Bright" ;;
      5)  name="Bulb";       catg="lighting";  price=3.49;  status="active";   brand="Bright" ;;
      6)  name="Sofa";       catg="furniture"; price=499;   status="active";   brand="Sofix"  ;;
      7)  name="Chair";      catg="furniture"; price=79.9;  status="inactive"; brand="Sofix"  ;;
      8)  name="Desk";       catg="furniture"; price=199;   status="active";   brand="WoodCo" ;;
      9)  name="Cable";      catg="hardware";  price=4.99;  status="active";   brand="Omni"   ;;
      10) name="Switch";     catg="hardware";  price=39.95; status="active";   brand="Omni"   ;;
    esac

    json_payload=$(cat <<EOF
  {"type":"product","name":"${name}","category":"${catg}","price":${price},"status":"${status}","brand":"${brand}"}
  EOF
  )

    docker exec -i ${CB_CONTAINER} cbq \
      -e http://${CB_HOST}:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
      -s "UPSERT INTO \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` (KEY,VALUE)
          VALUES (\"prod::${i}\", ${json_payload});"
  done
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Índices secundarios (compuesto, parcial, funcional)

Crearás diferentes tipos de índices secundarios y verificarás su uso con **EXPLAIN**.



- **Paso 1.** Índice compuesto para filtros típicos.

  ```bash
  docker exec -i ${CB_CONTAINER} cbq -e http://${CB_HOST}:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" <<'SQL'
  CREATE INDEX idx_prod_type_cat_status
  ON `shop`.`inv`.`products`(`type`, `category`, `status`);
  \QUIT
  SQL
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 2.** Índice parcial (solo `type='product'`).

  ```bash
  docker exec -i ${CB_CONTAINER} cbq -e http://${CB_HOST}:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" <<'SQL'
  CREATE INDEX idx_prod_partial_active
  ON `shop`.`inv`.`products`(`status`)
  WHERE `type` = 'product';
  \QUIT
  SQL
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 3.** Índice funcional (búsqueda case-insensitive por nombre).

  ```bash
  docker exec -i ${CB_CONTAINER} cbq -e http://${CB_HOST}:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" <<'SQL'
  CREATE INDEX idx_prod_lower_name
  ON `shop`.`inv`.`products`(LOWER(`name`));
  \QUIT
  SQL
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Índice **covering**  y validación con EXPLAIN

Crearás un **índice covering** que cubra predicados y proyección para evitar el *FETCH* del documento.



- **Paso 1.** Crea el índice covering (claves + trailing keys).

  ```bash
  docker exec -it ${CB_CONTAINER} cbq -e http://${CB_HOST}:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "CREATE INDEX idx_covering_product
        ON \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`(\`type\`, \`category\`, \`status\`, \`name\`, \`price\`, \`brand\`);"
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 2.** **EXPLAIN** de una consulta cubierta muestra el plan para verificar que el índice cubre la consulta.

  ```bash
  docker exec -it ${CB_CONTAINER} cbq -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "EXPLAIN SELECT name, price, brand
        FROM ${CB_BUCKET}.${CB_SCOPE}.${CB_COLLECTION}
        WHERE type='product' AND category='hardware' AND status='active'
        ORDER BY name;"
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 3.** Ejecutar la consulta sin EXPLAIN (validación de resultado).

  ```bash
  docker exec -it ${CB_CONTAINER} cbq -e http://${CB_HOST}:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "SELECT p.name, p.price, p.brand
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` AS p
        WHERE p.\`type\`='product' AND p.\`category\`='hardware' AND p.\`status\`='active'
        ORDER BY p.name;"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Hints y planes alternos

Forzarás el uso de un índice con `USE INDEX` y compararás planes.



- **Paso 1.** Sin *hint* (deja que el optimizador elija el plan/índice óptimo).

  > **Nota.** Un `IndexScan` usando el índice que el optimizador considere mejor (el parcial si existe y aplica), y sin Fetch si el índice es covering.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${CB_CONTAINER} cbq \
    -e http://${CB_HOST}:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "EXPLAIN
        SELECT p.name
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` AS p
        WHERE p.\`type\`='product' AND p.\`status\`='active';"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 2.** Con hint (forzar índice parcial).

  > **Nota.** El plan debe mostrar `IndexScan` sobre **idx_prod_partial_active**. Si otro índice era “mejor” según el optimizador, aquí verás cómo cambia el plan al forzarlo.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${CB_CONTAINER} cbq \
    -e http://${CB_HOST}:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "EXPLAIN
        SELECT p.name
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` AS p
        USE INDEX (idx_prod_partial_active USING GSI)
        WHERE p.\`type\`='product' AND p.\`status\`='active';"
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 3.** Con índice funcional (búsqueda case-insensitive).

  > **Nota.** `IndexScan` sobre **idx_prod_lower_name**. Si no aparece, revisa que el índice esté online y que la expresión LOWER(name) coincide con la del índice.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${CB_CONTAINER} cbq \
    -e http://${CB_HOST}:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "EXPLAIN
        SELECT META(p).id AS id, p.name
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` AS p
        WHERE LOWER(p.\`name\`) = 'widget';"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Limpieza

Borrar datos en el entorno para repetir pruebas.



- **Paso 1.** En la terminal aplica el siguiente comando para detener el nodo.

  > **Nota.** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 2.** Apagar y eliminar contenedor (se conservan los datos en ./data).

  > **Nota.** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **Importante.** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
