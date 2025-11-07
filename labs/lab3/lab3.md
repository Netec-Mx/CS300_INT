---
layout: lab
title: "Práctica 3: Creación y prueba de buckets" # CAMBIAR POR CADA PRACTICA
permalink: /lab3/lab3/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab3/img # CAMBIAR POR CADA PRACTICA
duration: "20 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Crear y validar distintos tipos de buckets en Couchbase (Couchbase, Ephemeral) con sus principales parámetros (RAM, replicas, ejection/eviction, compresión), configurar scopes y collections, cargar datos de prueba y consultar con N1QL.
prerequisites: # CAMBIAR POR CADA PRACTICA
  - Visual Studio Code con terminal **Git Bash**.
  - 6–8 GB de RAM libres recomendados para 3 nodos (mínimo ~5 GB).  
  - Puertos locales disponibles **8091–8096**, **11210** (se publicarán solo desde el nodo 1).
  - Conectividad a Internet para descargar la imagen.
  - Opcional `jq` para mejorar salidas JSON.
introduction: # CAMBIAR POR CADA PRACTICA
  - En Couchbase, un *bucket* es la unidad lógica de almacenamiento. En esta práctica crearás dos buckets de ejemplo (tipos **Couchbase**, **Ephemeral**), definirás opciones clave (memoria, replicas, políticas de expulsión, compresión), y practicarás operaciones básicas listar, crear/flush, scopes/collections y consultas con N1QL.
slug: lab3 # CAMBIAR POR CADA PRACTICA
lab_number: 3 # CAMBIAR POR CADA PRACTICA
final_result: > # CAMBIAR POR CADA PRACTICA
  Buckets **Couchbase**, **Ephemeral** creados, configurados y validados; con *scopes* y *collections* operativos en `orders`, datos de prueba insertados y consultas N1QL funcionando. Conocimiento práctico de compresión y políticas de expulsión.
notes: # CAMBIAR POR CADA PRACTICA
  - En **Community Edition** algunas opciones avanzadas (p.ej., **magma**) no están disponibles.  
  - Ajusta RAM por bucket según tu máquina. Si ves errores de cuota, libera memoria o reduce tamaños.  
  - Para producción, evita `flush` y `PRIMARY INDEX`; define índices secundarios específicos y políticas de seguridad.  
  - La nomenclatura de *keys* (`o::1001`) ayuda a segmentar documentos por tipo.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Buckets y tipos
    url: https://docs.couchbase.com/server/current/manage/manage-buckets/create-bucket.html 
  - text: Scopes & Collections
    url: https://docs.couchbase.com/server/current/learn/data/scopes-and-collections.html
  - text: N1QL y colecciones
    url: https://www.couchbase.com/blog/es/couchbase-transactions-with-n1ql/
  - text: Políticas de expulsión
    url: https://docs.couchbase.com/sdk-api/couchbase-node-client/enums/EvictionPolicy.html
prev: /lab2/lab2/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab4/lab4/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1. Preparación del espacio de trabajo

Organizarás la carpeta de la práctica y verificarás que el contenedor de Couchbase esté arriba.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica3-buckets/
  mkdir -p practica3-buckets/scripts
  cd practica3-buckets
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER_NAME=cbnode1
  CB_USER=admin
  CB_PASS='adminlab'
  CB_SERVICES=data,index,query
  CB_RAM=2048
  CB_INDEX_RAM=512
  CB_BUCKET=dev
  CB_BUCKET_RAM=256
  EOF
  ```

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
      container_name: ${CB_CONTAINER_NAME}
      ports:
        - "8091-8096:8091-8096"   # Web UI / servicios
        - "11210:11210"           # Memcached/SDK (data)
      environment:
        # (La inicialización la haremos con CLI; aquí solo variables útiles)
        - CB_USERNAME=${CB_USER}
        - CB_PASSWORD=${CB_PASS}
      volumes:
        - ./data:/opt/couchbase/var
        - ./logs:/opt/couchbase/var/lib/couchbase/logs
        - ./config:/config
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

  ![cbase4]({{ page.images_base | relative_url }}/4.png)

- **Paso 7.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM. Habilitar `flush` permite vaciar el bucket desde la UI o CLI.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica3-buckets**
  {: .lab-note .important .compact}

  ```bash
  set -a; source .env; set +a
  docker exec -it "$CB_CONTAINER_NAME" couchbase-cli cluster-init \
    --cluster 127.0.0.1 \
    --cluster-username "$CB_USER" \
    --cluster-password "$CB_PASS" \
    --services "${SERVICES_NORM:-$CB_SERVICES}" \
    --cluster-ramsize "$CB_RAM" \
    --cluster-index-ramsize "$CB_INDEX_RAM" \
    --index-storage-setting default
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Crear buckets (Couchbase, Ephemeral)

Crearás 2 buckets con diferentes configuraciones para observar su comportamiento y ajustes disponibles.

#### Tarea 2.1 - Bucket tipo *Couchbase* (`orders`)

- **Paso 7.** Crear bucket *Couchbase* con **256MB**, **replicas=1** y *flush* habilitado.

  > **NOTA:** **Couchbase**: persistente, soporta replicas, índices y Query.  
  {: .lab-note .info .compact}

  ```bash
  docker exec -it cbnode1 couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_USER}" -p "${CB_PASS}" \
    --bucket "${CB_BUCKET1:-orders}" \
    --bucket-type couchbase \
    --bucket-ramsize 256 \
    --enable-flush 1 \
    --bucket-replica 1 \
    --compression-mode passive
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

#### Tarea 2.2 - Bucket *Ephemeral* **(`cache`)**

- **Paso 8.** Crear bucket *Ephemeral* **128MB** con política de expulsión por NRU (recomendado para cache).

  > **NOTA:** **Ephemeral**: solo memoria (sin disco), ideal para cache con políticas de expulsión.  
  {: .lab-note .info .compact}

  ```bash
  docker exec -it cbnode1 couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_USER}" -p "${CB_PASS}" \
    --bucket "${CB_BUCKET2:-cache}" \
    --bucket-type ephemeral \
    --bucket-ramsize 128 \
    --enable-flush 1 \
    --bucket-eviction-policy nruEviction \
    --compression-mode off
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Scopes y Collections (en bucket `orders`)

Crearás un *scope* y *collections* lógicas para organizar datos por dominio (p. ej., ventas/clientes).

#### Tarea 3.1

- **Paso 9.** Crear un **scope** `sales`, **collections** `orders` y `customers` (vía REST Management API).

  > **NOTA:** Scopes/collections permiten aislar lógicamente datos dentro de un bucket sin necesitar múltiples buckets.
  {: .lab-note .info .compact}

  ```bash
  # Crear scope
  curl -fsS -u "${CB_USER}:${CB_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET1:-orders}/scopes" \
    -d 'name=sales'
  ```
  ```bash
  # Crear collections dentro de scope 'sales'
  curl -fsS -u "${CB_USER}:${CB_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET1:-orders}/scopes/sales/collections" \
    -d 'name=orders'
  ```
  ```bash
  curl -fsS -u "${CB_USER}:${CB_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET1:-orders}/scopes/sales/collections" \
    -d 'name=customers'
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 10.** Ahora ejecuta el siguiente comando para listar los **scopes/collections**.

  > **NOTA:** El JSON listado debe incluir el `scope` **sales** con `collections` **orders** y **customers**.
  {: .lab-note .info .compact}

  ```bash
  curl -fsS -u "${CB_USER}:${CB_PASS}" \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET1:-orders}/scopes" | jq '.'
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Índices y datos de prueba (N1QL)

Crearás índices primarios para poder consultar con N1QL y cargarás documentos de ejemplo en las nuevas collections.

#### Tarea 4.1

- **Paso 11.** Crea el siguiente índice primario **`orders`._default._default**.

  ```bash
  docker exec -it cbnode1 cbq -e http://127.0.0.1:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s "CREATE PRIMARY INDEX ON \`${CB_BUCKET1:-orders}\`._default._default;"
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 12.** Crea el siguiente índice primario **`orders`.sales.orders**.

  ```bash
  docker exec -it cbnode1 cbq -e http://127.0.0.1:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s "CREATE PRIMARY INDEX ON \`${CB_BUCKET1:-orders}\`.sales.orders;"
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 13.** Crea el siguiente índice primario **`orders`.sales.customers**.
  ```bash
  docker exec -it cbnode1 cbq -e http://127.0.0.1:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s 'CREATE PRIMARY INDEX ON `'${CB_BUCKET1:-orders}'`.sales.customers;'
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 14.** Ahora el siguiente paso es insertar un documento de datos para **`orders`.sales.orders**, escribe el siguiente comando en la terminal.

  ```bash
  # orders
  docker exec -it cbnode1 cbq -e http://127.0.0.1:8093 -u "$CB_USER" -p "$CB_PASS" -s "
  INSERT INTO \`orders\`.sales.orders (KEY, VALUE) VALUES
    ('o::1001', {\"orderId\":1001, \"customerId\":\"c::001\", \"total\":149.9, \"status\":\"PAID\", \"ts\": NOW_MILLIS()}),
    ('o::1002', {\"orderId\":1002, \"customerId\":\"c::002\", \"total\":89.5,  \"status\":\"NEW\",  \"ts\": NOW_MILLIS()});
  "
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 15.** Continua con el siguiente paso para insertar un documento de datos en **`orders`.sales.customers**, escribe el siguiente comando en la terminal.

  ```bash
  # customers
  docker exec -it cbnode1 cbq -e http://127.0.0.1:8093 -u "$CB_USER" -p "$CB_PASS" -s "
  INSERT INTO \`orders\`.sales.customers (KEY, VALUE) VALUES
    ('c::001', {\"customerId\":\"c::001\", \"name\":\"Ana\",   \"tier\":\"GOLD\"}),
    ('c::002', {\"customerId\":\"c::002\", \"name\":\"Diego\", \"tier\":\"SILVER\"});
  "
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 16.** Ahora realiza esta consulta para verificar que se haya almacenado todo correctamente para **`orders`.sales.orders**.

  ```bash
  docker exec -it cbnode1 cbq -e http://127.0.0.1:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s "SELECT orderId,total,status FROM \`${CB_BUCKET1:-orders}\`.sales.orders ORDER BY orderId;"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 17.** Ahora realiza esta consulta para verificar que se haya almacenado todo correctamente para **`orders`.sales.customers**.

  ```bash
  docker exec -it cbnode1 cbq -e http://127.0.0.1:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s "SELECT c.name,o.total FROM \`${CB_BUCKET1:-orders}\`.sales.orders o
        JOIN \`${CB_BUCKET1:-orders}\`.sales.customers c ON o.customerId=c.customerId
        ORDER BY o.orderId;"
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Compresión y políticas de expulsión

Revisarás las configuraciones de compresión/eviction en cada bucket.

#### Tarea 5.1

- **Paso 18.** Revisa las configuraciones en los buckets creados

  > **NOTA:** El JSON muestra `bucketType`, `evictionPolicy`, `compressionMode` conforme a lo configurado. 
  {: .lab-note .info .compact}

  ```bash
  curl -fsS -u "${CB_USER}:${CB_PASS}" http://localhost:8091/pools/default/buckets | jq '.[] | {name, bucketType, evictionPolicy, compressionMode, replicas: .replicaNumber}'
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 19.** Cambia la compresión en `orders` a `active`.

  > **NOTA:** El `bucket-edit` termina sin errores al cambiar compresión.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it cbnode1 couchbase-cli bucket-edit \
    -c 127.0.0.1 -u "${CB_USER}" -p "${CB_PASS}" \
    --bucket "${CB_BUCKET1:-orders}" \
    --compression-mode active
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 20.** Verifica que la compresión se haya activado correctamente, copia y pega el siguiente comando en al terminal.

  ```bash
  curl -fsS -u "$CB_USER:$CB_PASS" \
    http://localhost:8091/pools/default/buckets/${CB_BUCKET1:-orders} \
  | jq -r '.name, .compressionMode'
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Limpieza

Aprenderás a apagar/encender el servicio y a limpiar volúmenes si necesitas empezar desde cero.

#### Tarea 6.1

- **Paso 21.** Ejecuta el siguiente comando para eliminar los buckets creados.

  > **NOTA:** El borrado permite repetir la práctica con distintos parámetros (RAM, compresión, replicas, etc.).
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Puede tardar unos segundos en borrarse, es normal.
  {: .lab-note .important .compact}

  ```bash
  for B in "${CB_BUCKET1:-orders}" "${CB_BUCKET2:-cache}"; do
    docker exec -it cbnode1 couchbase-cli bucket-delete \
      -c 127.0.0.1 -u "${CB_USER}" -p "${CB_PASS}" \
      --bucket "$B" || true
  done
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

- **Paso 22.** Verifica que en efecto ya se hayan eliminado, con el siguiente comando.

  > **NOTA:** Se espera un resultado vacio dado que ya se eliminaron los buckets.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it cbnode1 couchbase-cli bucket-list \
    -c 127.0.0.1 -u "${CB_USER}" -p "${CB_PASS}"
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

- **Paso 23.** Ahora en la terminal de **Git Bash** aplica el siguiente comando.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

- **Paso 24.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}

  ```bash
  docker compose down
  ```
  ![cbase24]({{ page.images_base | relative_url }}/24.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}