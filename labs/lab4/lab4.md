---
layout: lab
title: "Práctica 4. Creación de roles personalizados" # CAMBIAR POR CADA PRACTICA
permalink: /lab4/lab4/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab4/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Diseñar y aplicar **roles personalizados** (combinaciones de roles RBAC a nivel `bucket`, `scope`, `collection`) para distintos perfiles de usuario (analista de solo lectura, aplicación escritora y DBA de scope), verificando permisos permitidos o denegados mediante CLI, REST y N1QL.
prerequisites: # CAMBIAR POR CADA PRACTICA
  - Visual Studio Code con terminal **Git Bash**.
  - 6–8 GB de RAM libres recomendados para tres nodos (mínimo ~5 GB).  
  - Puertos locales disponibles **`8091-8096`**, **`11210`** (se publicarán solo desde el nodo 1).
  - Conectividad a Internet para descargar la imagen.
  - Opcional `jq` para mejorar salidas JSON.
introduction: | # CAMBIAR POR CADA PRACTICA
  Couchbase implementa RBAC con **roles predefinidos** que pueden acotarse por *`bucket`, `scope`, `collection`. No se “crean roles nuevos” arbitrarios; en su lugar, se **combinan** roles existentes con el alcance mínimo necesario (principio de **menor privilegio**). En esta práctica configuraremos tres perfiles:  
  - 1) **`analyst_ro`** (solo lectura en una `collection`),  
  - 2) **`writer_app`** (inserta/actualiza en una `collection` pero no puede leer),  
  - 3) **`dba_scope`** (administra índices y el bucket `orders`).
slug: lab4 # CAMBIAR POR CADA PRACTICA
lab_number: 4 # CAMBIAR POR CADA PRACTICA
final_result: > # CAMBIAR POR CADA PRACTICA
  Has implementado RBAC granular con tres perfiles representativos y validado permisos permitidos o denegados sobre *collections* específicas. Queda un patrón reutilizable para diseñar combinaciones de roles de **menor privilegio** para distintos casos (BI, microservicios, administración).
notes: | # CAMBIAR POR CADA PRACTICA
  En Couchbase 7.x+ los roles pueden restringirse hasta **`collection-level`** (`role[bucket:scope:collection]`).  
  - Si recibes errores de permisos, valida 
    - 1) *keyspace* correcto en N1QL, 
    - 2) existencia de índices para SELECT
    - 3) combinación de roles.  
  - Evita otorgar `bucket_full_access` a aplicaciones; prefiere los roles mínimos (`data_reader`, `data_writer`, `query_*`).  
  - Contraseñas en texto claro en labs son por simplicidad; usa secretos/variables seguras en producción.  
  - Cambios en roles pueden tardar unos segundos en surtir efecto en sesiones activas de `cbq`.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: RBAC (roles y alcance)
    url: https://docs.couchbase.com/server/current/learn/security/roles.html  
  - text: couchbase-cli user-manage
    url: https://docs.couchbase.com/server/current/cli/cbcli/couchbase-cli-user-manage.html  
  - text: Scopes y Collections
    url: https://docs.couchbase.com/server/current/learn/data/scopes-and-collections.html 
  - text: N1QL y colecciones
    url: https://www.couchbase.com/blog/es/query-couchbase-data-structures-with-n1ql-sql-for-json/
  - text: Query/Indexes (gestión)
    url: https://www.couchbase.com/blog/understanding-index-grouping-aggregation-couchbase-n1ql-query/
prev: /lab3/lab3/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab5/lab5/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1. Preparación del entorno y variables

Organizarás la carpeta de la práctica, cargarás variables y verificarás que el contenedor esté arriba. Si aún no existen `orders/sales/{orders,customers}`, se crearán.


- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**.

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **`cs300-labs`**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica4-rbac/
  mkdir -p practica4-rbac/scripts
  cd practica4-rbac
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **Nota.** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER_NAME=cbnode1
  CB_HOST=127.0.0.1
  CB_USER=admin
  CB_PASS='adminlab'
  CB_SERVICES=data,index,query
  CB_RAM=2048
  CB_INDEX_RAM=512
  CB_BUCKET_RAM=256

  # Recursos
  CB_BUCKET=orders
  CB_SCOPE=sales
  CB_COLL_ORDERS=orders
  CB_COLL_CUSTOMERS=customers

  # Usuarios para la práctica
  RBAC_ANALYST=analyst_ro
  RBAC_ANALYST_PASS='Ana!2025'

  RBAC_WRITER=writer_app
  RBAC_WRITER_PASS='Writ3r!2025'

  RBAC_DBA=dba_scope
  RBAC_DBA_PASS='DbA!2025'
  EOF
  ```

- **Paso 5.** Crea el archivo **Docker Compose** llamado **`compose.yaml`**. Copia y pega el siguiente código en la terminal.

  > **Nota.**
  - El archivo `compose.yaml` mapea puertos `8091-8096` para la consola web y `11210` para clientes.
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

- **Paso 6.** Inicia el servicio. Dentro de la terminal, ejecuta el siguiente comando.

  > **Importante**
  > - Para agilizar los procesos, la imagen debe estar descargada descargada en el ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  > - El `docker compose up -d` corre en segundo plano. El *healthcheck* del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en `8091` antes de continuar.
  {: .lab-note .important .compact}

  ```bash
  docker compose up -d
  ```

  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 7.** Inicializa el clúster y ejecuta el siguiente comando en la terminal.

  > **Nota.** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM. Habilitar `flush` permite vaciar el bucket desde la UI o CLI.
  {: .lab-note .info .compact}
  > **Importante.** El comando se ejecuta desde el directorio de la práctica **practica4-rbac**
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
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

- **Paso 8.** Verifica que el clúster esté *healthy* y que se muestre el `json` con las propiedades del nodo.

  > **Notas**
  - El contenedor `cbnode1` aparece **`Up`**.  
  - `curl` devuelve `JSON` de la información del nodo.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cbnode1"
  curl -fsS -u "$CB_USER:$CB_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Asegurar `bucket`, `scope` y `collections`

Crearás el bucket `orders`, el `Scope` `sales` y las *collections* `orders` y `customers`. También crearás índices primarios necesarios para las consultas.


- **Paso 1.** Crear el bucket `orders` con **256 MB** y *flush* habilitado.

  ```bash
  docker exec -it cbnode1 couchbase-cli bucket-create \
    -c ${CB_HOST} -u "${CB_USER}" -p "${CB_PASS}" \
    --bucket "${CB_BUCKET}" \
    --bucket-type couchbase \
    --bucket-ramsize 256 \
    --enable-flush 1 \
    --bucket-replica 1 \
    --compression-mode passive || true
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 2.** Crea el `Scope` **`sales`** vía `REST`, copia y pega el comando en la terminal.

  ```bash
  # Scope
  curl -fsS -u "${CB_USER}:${CB_PASS}" -X POST \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" \
    -d "name=${CB_SCOPE}" || true
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 3.** Crea el `collections` `orders` vía `REST`, copia y pega el comando en la terminal.

  ```bash
  # Collection: orders
  curl -fsS -u "${CB_USER}:${CB_PASS}" -X POST \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLL_ORDERS}" || true
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 4.** Crea el *`collections`* **`customers`** vía `REST`, copia y pega el comando en la terminal.

  ```bash
  # Collection: customers
  curl -fsS -u "${CB_USER}:${CB_PASS}" -X POST \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLL_CUSTOMERS}" || true
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 5.** Crea el índice primario para **`orders`** y realizar las pruebas de N1QL.

  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON \`${CB_BUCKET}\`.${CB_SCOPE}.${CB_COLL_ORDERS};"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 6.** Crea el índice primario para **`customers`** y realizar las pruebas de `N1QL`.

  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON \`${CB_BUCKET}\`.${CB_SCOPE}.${CB_COLL_CUSTOMERS};"
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 7.** Ejecuta el siguiente comando para listar **`sales`** con **`orders`** y **`customers`** correctamente.

  > **Nota.** Configuraste el *keyspace* `orders.sales.orders` y `orders.sales.customers` para probar permisos granulares por *collection*.
  {: .lab-note .info .compact}

  ```bash
  curl -fsS -u "${CB_USER}:${CB_PASS}" \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" | jq '.'
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear usuarios y asignar roles (RBAC granular)

Definirás tres usuarios locales con combinaciones de roles predefinidos acotados por `bucket`, `scope` y `collection`.


- **Paso 1.** Crea el usuario **`analyst_ro`** (solo lectura en `orders.sales.orders`), luego copia y pega el siguiente comando en la terminal.

  ```bash
  docker exec -it cbnode1 couchbase-cli user-manage \
    -c ${CB_HOST} -u "${CB_USER}" -p "${CB_PASS}" \
    --set --rbac-username "${RBAC_ANALYST}" --rbac-password "${RBAC_ANALYST_PASS}" \
    --auth-domain local \
    --roles 'data_reader[orders:sales:orders],query_select[orders:sales:orders]'
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 2.** Crea el usuario **`writer_app`** (escritura en `orders.sales.orders`, sin permisos de lectura/`SELECT`).

  ```bash
  docker exec -it cbnode1 couchbase-cli user-manage \
    -c "$CB_HOST" -u "$CB_USER" -p "$CB_PASS" \
    --set --rbac-username "$RBAC_WRITER" --rbac-password "$RBAC_WRITER_PASS" \
    --auth-domain local \
    --roles "data_writer[${CB_BUCKET}:${CB_SCOPE}:${CB_COLL_ORDERS}],query_insert[${CB_BUCKET}:${CB_SCOPE}:${CB_COLL_ORDERS}],query_update[${CB_BUCKET}:${CB_SCOPE}:${CB_COLL_ORDERS}]"
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 3.** Crear **`dba_scope`** (admin de bucket + gestión de índices en `orders`).

  ```bash
  docker exec -it cbnode1 couchbase-cli user-manage \
    -c ${CB_HOST} -u "${CB_USER}" -p "${CB_PASS}" \
    --set --rbac-username "${RBAC_DBA}" --rbac-password ${RBAC_DBA_PASS} \
    --auth-domain local \
    --roles "bucket_admin[${CB_BUCKET}],query_manage_index[${CB_BUCKET}]"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 4.** Lista los usuarios y roles asignados creados previamente.

  > **Nota.** No existen “roles personalizados” arbitrarios: se **combinan roles** predefinidos con alcance granular.  
  {: .lab-note .info .compact}

  ```bash
  docker exec cbnode1 curl -fsS -u "$CB_USER:$CB_PASS" \
    http://127.0.0.1:8091/settings/rbac/users/local \
  | jq -r '(["name","id","type"]),
          (.[] | [(.name // .id), .id, .domain]) | @tsv' \
  | column -t -s $'\t'
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Cargar datos de ejemplo y probar permisos

Insertarás datos con `writer_app`, intentarás y validarás operaciones permitidas o denegadas para cada usuario.


- **Paso 1.** Inserta un documento con el usuario **`writer_app`** (permitido).

  ```bash
  docker exec -i cbnode1 cbq \
    -e "http://${CB_HOST}:8093" \
    -u "${RBAC_WRITER}" -p "${RBAC_WRITER_PASS}" \
    -s "INSERT INTO \`${CB_BUCKET}\`.${CB_SCOPE}.${CB_COLL_ORDERS} (KEY, VALUE) VALUES
        (\"o::2001\", {\"orderId\":2001, \"customerId\":\"c::010\", \"total\":120.5, \"status\":\"NEW\",  \"ts\":NOW_MILLIS()}),
        (\"o::2002\", {\"orderId\":2002, \"customerId\":\"c::011\", \"total\":75.0,  \"status\":\"PAID\", \"ts\":NOW_MILLIS()});"
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 2.** Ahora con `writer_app`, intenta realizar un **`SELECT`** (debería **fallar** por falta de `query_select`).

  > **Importante.** Cuando ejecutas el **`SELECT`** con el usuario **`writer`** (sin el rol `query_select`), el servicio de Query responde `HTTP 401` (no autorizado). Es normal el mensaje **`ERROR 174...`**
  {: .lab-note .important .compact}

  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 \
    -u "$RBAC_WRITER" -p "$RBAC_WRITER_PASS" -s \
  "SELECT orderId,total FROM \`${CB_BUCKET}\`.${CB_SCOPE}.${CB_COLL_ORDERS} LIMIT 2;"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 3.** Ahora, prueba con `analyst_ro` ejecuta **`SELECT`** (debe permitirlo).

  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 -u "${RBAC_ANALYST}" -p ${RBAC_ANALYST_PASS} \
    -s "SELECT META().id, orderId, total FROM \`${CB_BUCKET}\`.${CB_SCOPE}.${CB_COLL_ORDERS} ORDER BY orderId LIMIT 5;"
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 4.** Ahora, prueba con `analyst_ro` ejecuta **`INSERT`** (debe **fallar**).

  > **Importante.** Cuando ejecutas el **`INSERT`** con el usuario **`analyst_ro`** (sin el _rol writer_), el servicio de `insert` responde `HTTP 401` (no autorizado). Es normal el mensaje **`ERROR 174...`**.
  {: .lab-note .important .compact}

  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 -u "${RBAC_ANALYST}" -p ${RBAC_ANALYST_PASS} \
    -s "INSERT INTO \`${CB_BUCKET}\`.${CB_SCOPE}.${CB_COLL_ORDERS} (KEY, VALUE) VALUES ('o::9999', {test:true});"
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 5.** Con el usuario `dba_scope` crea un índice secundario, copia y pega el siguiente comando en la terminal.

  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 -u "${RBAC_DBA}" -p ${RBAC_DBA_PASS} \
    -s "CREATE INDEX idx_status ON \`${CB_BUCKET}\`.${CB_SCOPE}.${CB_COLL_ORDERS}(status);"
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

- **Paso 6.** Con el usuario `dba_scope`, consulta el índice secundario.

  > **Importante**
  - El primer comando le da permiso de lectura al **`dba_scope`**.
  - El segundo realiza la consulta.
  {: .lab-note .important .compact}

  ```bash
  docker exec -it cbnode1 couchbase-cli user-manage \
    -c "$CB_HOST" -u "$CB_USER" -p "$CB_PASS" \
    --set --rbac-username "$RBAC_DBA" --rbac-password "$RBAC_DBA_PASS" \
    --auth-domain local \
    --roles "bucket_admin[${CB_BUCKET}],query_manage_index[${CB_BUCKET}:${CB_SCOPE}],query_select[${CB_BUCKET}:${CB_SCOPE}:${CB_COLL_ORDERS}]"
  ```
  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 -u "${RBAC_DBA}" -p ${RBAC_DBA_PASS} \
    -s "SELECT status, COUNT(*) AS n FROM \`${CB_BUCKET}\`.${CB_SCOPE}.${CB_COLL_ORDERS} GROUP BY status;"
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Auditoría de permisos y `REST`

Consultarás por `REST` los usuarios y sus roles y verificarás el `scope map` del bucket.


- **Paso 1.** Lista los usuarios locales mediante las operaciones `REST`.

  > **Nota.** Debes ver a `analyst_ro`, `writer_app` y `dba_scope` con los roles configurados.
  {: .lab-note .info .compact}

  ```bash
  curl -fsS -u "$CB_USER:$CB_PASS" http://localhost:8091/settings/rbac/users \
  | jq -r '
    ["id","role","bucket","scope","collection"],
    ( .[] as $u
      | ($u.roles[]? // [])
      | [ $u.id, .role, (.bucket_name//"-"), (.scope_name//"-"), (.collection_name//"-") ]
    )
    | @tsv' \
  | column -t -s $'\t'
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

- **Paso 2.** Revisa los `scopes`/`collections` del bucket y ejecuta el siguiente comando.

  > **Nota.** El mapa de `scopes`/`collections` coincide con lo creado.
  {: .lab-note .info .compact}

  ```bash
  curl -fsS -u "$CB_USER:$CB_PASS" \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" \
  | jq -r '
    ["scope","collection","scope_uid","coll_uid","history","maxTTL"],
    ( .scopes[] as $s
      | $s.collections[]?
      | [ $s.name, .name, $s.uid, .uid, (.history//"-"), (.maxTTL//"-") ]
    )
    | @tsv' \
  | column -t -s $'\t'
  ```
  ![cbase24]({{ page.images_base | relative_url }}/24.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Limpieza

Eliminarás usuarios creados en está práctica y toda la configuración creada.


- **Paso 1.** Borra los usuarios RBAC creados para la práctica.

  ```bash
  for U in "${RBAC_ANALYST}" "${RBAC_WRITER}" "${RBAC_DBA}"; do
    docker exec -it cbnode1 couchbase-cli user-manage \
      -c ${CB_HOST} -u "${CB_USER}" -p "${CB_PASS}" \
      --delete --rbac-username "$U" --auth-domain local || true
  done
  ```
  ![cbase25]({{ page.images_base | relative_url }}/25.png)

- **Paso 2.** Ahora, borra el índice secundario.

  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 -u "$CB_USER" -p "$CB_PASS" -s \
  "DROP INDEX \`idx_status\` ON \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLL_ORDERS}\`;"
  ```
  ![cbase26]({{ page.images_base | relative_url }}/26.png)

- **Paso 3.** Elimina el bucket con el siguiente comando.

  > **Nota.** Espera unos segundos, es normal que tarde en eliminarse.
  {: .lab-note .info .compact}
  > **Importante.** Si llegase a marcar algún error en el borrado, puede deberse a que aun está ocupado en alguna tarea, inténtalo nuevamente. En ocasiones, se borra aunque muestre el mensaje de error.
  {: .lab-note .important .compact}

  ```bash
  docker exec -it cbnode1 couchbase-cli bucket-delete \
    -c 127.0.0.1 -u "$CB_USER" -p "$CB_PASS" \
    --bucket orders
  ```
  ![cbase27]({{ page.images_base | relative_url }}/27.png)

- **Paso 4.** En la terminal, aplica el siguiente comando para detener el nodo.

  > **Nota.** Si es necesario, puedes volver a encender los contenedores con el comando **`docker compose start`**.
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase28]({{ page.images_base | relative_url }}/28.png)

- **Paso 5.** Apaga y elimina el contenedor (se conservan los datos en `./data`).

  > **Nota.** Si es necesario, puedes volver a activar los contenedores con el comando **`docker compose up -d`**.
  {: .lab-note .info .compact}

  ```bash
  docker compose down
  ```
  ![cbase29]({{ page.images_base | relative_url }}/29.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
