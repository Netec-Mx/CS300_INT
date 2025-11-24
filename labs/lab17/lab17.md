---
layout: lab
title: "Práctica 17: Replicación bidireccional" # CAMBIAR POR CADA PRACTICA
permalink: /lab17/lab17/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab17/img # CAMBIAR POR CADA PRACTICA
duration: "35 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar **dos clústeres** Couchbase en Docker y configurar **XDCR bidireccional** (Origen ↔ Destino) entre buckets y colecciones equivalentes. Probarás la replicación en ambos sentidos, validarás estados, **resolverás conflictos** usando *Last-Write-Wins (LWW)* y comprobarás que los cambios convergen de forma consistente.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 6–8 GB de RAM libres (2×2 nodos con `data,index,query`).  
introduction: | # CAMBIAR POR CADA PRACTICA
  **XDCR** permite replicar datos entre clústeres. En modo **bidireccional**, cada clúster envía y recibe cambios del otro. Para evitar bucles y resolver diferencias, Couchbase aplica **resolución de conflictos** (por *seqno* o por **LWW**). En esta práctica definiremos el bucket con **LWW** para que el documento con **timestamp mayor** prevalezca cuando se actualice la **misma clave** en ambos clústeres.
slug: lab17 # CAMBIAR POR CADA PRACTICA
lab_number: 17 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Has levantado **dos clústeres** y configurado **XDCR bidireccional**. Probaste replicación en ambos sentidos, ejecutaste un **conflicto controlado** y confirmaste la **resolución LWW** y la **convergencia**
notes: | # CAMBIAR POR CADA PRACTICA
  - En producción, habilita **TLS**, **certificados** y **Alternate Addresses** cuando uses LB o redes mixtas.  
  - Ajusta parámetros de XDCR (tamaño de lote, *checkpoint intervals*, compresión) según latencia/throughput.  
  - Si tu modelo de datos lo permite, usa **filtros** o **mapeos de colecciones** para reducir volumen y evitar ruido.  
  - Para *disaster recovery*, prueba *promotion* y *fallback* de roles con pruebas de *failover* y *backfill* controlado.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Visión general de XDCR
    url: https://docs.couchbase.com/server/current/learn/clusters-and-availability/xdcr-overview.html 
  - text: CLI xdcr-setup / xdcr-replicate
    url: https://docs.couchbase.com/server/current/cli/cbcli/couchbase-cli-xdcr-setup.html
  - text: Resolución de conflictos (LWW vs seqno)
    url: https://docs.couchbase.com/server/current/learn/clusters-and-availability/xdcr-conflict-resolution.html
  - text: Puertos de Couchbase (útil para mapeos)
    url: https://docs.couchbase.com/server/current/install/install-ports.html
prev: /lab16/lab16/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab18/lab18/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Carpeta de práctica y variables

Crearás una carpeta aislada con un `.env` que define puertos, nombres de contenedores y keyspaces.

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica17-xdcr-bidi/
  mkdir -p practica17-xdcr-bidi/src/{data,logs,config}
  mkdir -p practica17-xdcr-bidi/dst/{data,logs,config}
  mkdir -p practica17-xdcr-bidi/scripts
  cd practica17-xdcr-bidi
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **Nota.** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise

  # --- Cluster ORIGEN (Source) ---
  SRC_NODE=cb-src1
  SRC_HOST=couchbases1
  SRC_UI_RANGE=18091-18096     # mapea a 8091-8096 internos
  SRC_KV_PORT=11211            # (opcional) expone 11210 interno

  # --- Cluster DESTINO (Target) ---
  DST_NODE=cb-dst1
  DST_HOST=couchbased1
  DST_UI_RANGE=28091-28096
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
  CB_BUCKET=shop
  CB_BUCKET_RAM=512
  CB_SCOPE=inv
  CB_COLLECTION=products

  # Tipo de resolución de conflictos del bucket
  #   lww = Last-Write-Wins (por timestamp)
  #   seqno = por secuencia de mutación
  CB_CONFLICT=timestamp
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Docker Compose con **dos clústeres (1 nodo c/u)**

Definirás un `compose.yaml` que levanta dos nodos Couchbase (Origen y Destino) con *healthcheck* y puertos diferenciados.

- **Paso 1.** Ahora crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente codigo en la terminal.

  > **Nota.**
  - El archivo `compose.yaml` mapea puertos 8091–8096 para la consola web y 11210 para clientes.
  - El healthcheck consulta el endpoint `/pools` que responde cuando el servicio está arriba (aunque aún no inicializado).
  {: .lab-note .info .compact}

  ```bash
  cat > compose.yaml << 'YAML'
  name: couchbase-xdcr-bidi-lab
  services:
    # --------- CLUSTER ORIGEN ----------
    src:
      image: couchbase/server:${CB_TAG}
      container_name: ${SRC_NODE}
      hostname: ${SRC_HOST}
      ports:
        - "${SRC_UI_RANGE}:8091-8096"
        - "${SRC_KV_PORT:-11210}:11210"
      environment:
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      volumes:
        - ./src/data1:/opt/couchbase/var
        - ./src/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./src/config:/config
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 20

    # --------- CLUSTER DESTINO ----------
    dst:
      image: couchbase/server:${CB_TAG}
      container_name: ${DST_NODE}
      hostname: ${DST_HOST}
      ports:
        - "${DST_UI_RANGE}:8091-8096"
        - "${DST_KV_PORT:-11210}:11210"
      environment:
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      volumes:
        - ./dst/data1:/opt/couchbase/var
        - ./dst/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./dst/config:/config
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

- **Paso 2.** Inicia el servicio, dentro de la terminal ejecuta el siguiente comando.

  > **Importante.** Para agilizar los procesos, la imagen ya esta descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  {: .lab-note .important .compact}
  > **Importante.** El `docker compose up -d` corre en segundo plano. El healthcheck del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en 8091 antes de continuar.
  {: .lab-note .important .compact}

  ```bash
  docker compose up -d
  ```
  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 3.** Verifica la salud de los contenedores que se hayan creado correctamente.

  {%raw%}
  ```bash
  docker inspect -f '{{.State.Health.Status}}' cb-src1
  docker inspect -f '{{.State.Health.Status}}' cb-dst1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Inicializar los clústeres y crear buckets/colecciones (LWW)

Inicializarás cada clúster y crearás el **mismo bucket/scope/collection** en ambos con **LWW** para facilitar la prueba de conflicto controlado.

#### Tarea 3.1

- **Paso 1.** Inicializa el clúster con el nodo **cb-src1**, ejecuta el siguiete comando en la terminal.

  > **Nota.** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **Importante.** El comando se ejecuta desde el directorio de la practica **practica17-xdcr-bidi**. Puede tardar unos segundos en inicializar.
  {: .lab-note .important .compact}

  ```bash
  set -a; source .env; set +a
  docker exec -it ${SRC_NODE} couchbase-cli cluster-init \
    --cluster ${SRC_HOST} \
    --cluster-username "${CB_ADMIN}" \
    --cluster-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}" \
    --cluster-ramsize ${SRC_CLUSTER_RAM} \
    --cluster-index-ramsize ${SRC_INDEX_RAM} \
    --index-storage-setting default
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

- **Paso 2.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **Nota.**
  - Contenedor `cb-src1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-src1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:18091/pools | head -c 200; echo
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 3.** Ahora ejecuta el siguiente comando para la creación del bucket con **LWW**

  > **Nota.** **LWW (timestamp)** hace que el documento con **timestamp mayor** gane cuando hay conflicto por la misma clave.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${SRC_NODE} couchbase-cli bucket-create \
    -c ${SRC_HOST} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${CB_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize ${CB_BUCKET_RAM} --enable-flush 1 \
    --conflict-resolution ${CB_CONFLICT}
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 4.** Ahora crea el *Scope* **inv**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:18091/pools/default/buckets/${CB_BUCKET}/scopes" \
    -d "name=${CB_SCOPE}"
  ``` 
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 5.** Con este comando crea el *Collection* **products**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:18091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLLECTION}"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

#### Tarea 3.2.

- **Paso 1.** Inicializa el clúster con el nodo **cb-dst1**, ejecuta el siguiete comando en la terminal.

  ```bash
  set -a; source .env; set +a
  docker exec -it ${DST_NODE} couchbase-cli cluster-init \
    --cluster ${DST_HOST} \
    --cluster-username "${CB_ADMIN}" \
    --cluster-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}" \
    --cluster-ramsize ${DST_CLUSTER_RAM} \
    --cluster-index-ramsize ${DST_INDEX_RAM} \
    --index-storage-setting default
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 2.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  ```bash
  docker ps --filter "name=cb-dst1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:28091/pools | head -c 200; echo
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 3.** Ahora ejecuta el siguiente comando para la creación del bucket con **LWW**

  > **Nota.** **LWW (timestamp)** hace que el documento con **timestamp mayor** gane cuando hay conflicto por la misma clave.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${DST_NODE} couchbase-cli bucket-create \
    -c ${DST_HOST} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${CB_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize ${CB_BUCKET_RAM} --enable-flush 1 \
    --conflict-resolution ${CB_CONFLICT}
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 4.** Ahora crea el *Scope* **inv**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:28091/pools/default/buckets/${CB_BUCKET}/scopes" \
    -d "name=${CB_SCOPE}"
  ``` 
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 5.** Con este comando crea el *Collection* **products**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:28091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLLECTION}"
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Configurar **XDCR bidireccional** (referencias + replicaciones).

Crearás una **referencia remota** en cada clúster apuntando al otro y dos **replicaciones**: Origen→Destino y Destino→Origen.

#### Tarea 4.1.

- **Paso 1.** Crea la **referencia origen** en el nodo Origen.

  ```bash
  docker exec -it ${SRC_NODE} couchbase-cli xdcr-setup \
    -c ${SRC_HOST} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --create \
    --xdcr-cluster-name dst-cluster \
    --xdcr-hostname ${DST_HOST} \
    --xdcr-username "${CB_ADMIN}" \
    --xdcr-password "${CB_ADMIN_PASS}"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 2.** Ahora crea la **referencia destino** en el nodo Destino.

  ```bash
  docker exec -it ${DST_NODE} couchbase-cli xdcr-setup \
    -c ${DST_HOST} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --create \
    --xdcr-cluster-name src-cluster \
    --xdcr-hostname ${SRC_HOST} \
    --xdcr-username "${CB_ADMIN}" \
    --xdcr-password "${CB_ADMIN_PASS}"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

#### Tarea 4.2.

- **Paso 1.** Crea la **replicación** Origen -> Destino.

  ```bash
  docker exec -it ${SRC_NODE} couchbase-cli xdcr-replicate \
    -c ${SRC_HOST} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --create \
    --xdcr-cluster-name dst-cluster \
    --xdcr-from-bucket ${CB_BUCKET} \
    --xdcr-to-bucket ${CB_BUCKET} \
    --enable-compression 1
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 2.** Ahora crea la **replicación** Destino -> Origen.

  ```bash
  docker exec -it ${DST_NODE} couchbase-cli xdcr-replicate \
    -c ${DST_HOST} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --create \
    --xdcr-cluster-name src-cluster \
    --xdcr-from-bucket ${CB_BUCKET} \
    --xdcr-to-bucket ${CB_BUCKET} \
    --enable-compression 1
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 3.** Verifica que la replicación en el Origen este correcta.

  ```bash
  docker exec -it ${SRC_NODE} couchbase-cli xdcr-replicate -c ${SRC_HOST} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" --list
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 4.** Verifica que la replicación en el Destino este correcta.

  ```bash
  docker exec -it ${DST_NODE} couchbase-cli xdcr-replicate -c ${DST_HOST} -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" --list
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Pruebas de ida y vuelta entre Origen/Destino.

Insertarás documentos en **Origen** y **Destino** y validarás su llegada al opuesto.

- **Paso 1.** Inserta los siguientes datos desde el **Origen**.

  ```bash
  docker exec -it "${SRC_NODE}" cbq -e "http://${SRC_HOST}:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "UPSERT INTO \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` (KEY,VALUE) VALUES
        ('prod::101', {\"type\":\"product\", \"name\":\"XDCR Test Lamp\", \"category\":\"lighting\", \"price\":29.99, \"ts\":NOW_MILLIS()}),
        ('prod::102', {\"type\":\"product\", \"name\":\"XDCR Cable\",     \"category\":\"hardware\", \"price\": 4.99, \"ts\":NOW_MILLIS()});"
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

- **Paso 2.** Ahora, consulta los datos en el **Destino**.

  > **Nota.** La replicación puede tardar unos momentos.
  {: .lab-note .info .compact}

  ```bash
  sleep 5
  docker exec -it "${DST_NODE}" cbq -e "http://${DST_HOST}:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT META().id, name, category, price, ts
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`
        WHERE META().id IN ['prod::101','prod::102'];"
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

- **Paso 3.** Inserta los siguientes datos desde el **Destino**.

  ```bash
  docker exec -it "${DST_NODE}" cbq -e "http://${DST_HOST}:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "UPSERT INTO \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` (KEY,VALUE) VALUES
        ('prod::103', {\"type\":\"product\", \"name\":\"XDCR DST Lamp\", \"category\":\"lighting\", \"price\":49.99, \"ts\":NOW_MILLIS()}),
        ('prod::104', {\"type\":\"product\", \"name\":\"XDCR DST Cable\",     \"category\":\"hardware\", \"price\": 25.99, \"ts\":NOW_MILLIS()});"
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

- **Paso 4.** Ahora, consulta los datos en el **Origen**.

  > **Nota.** La replicación puede tardar unos momentos.
  {: .lab-note .info .compact}

  ```bash
  sleep 5
  docker exec -it "${DST_NODE}" cbq -e "http://${DST_HOST}:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT META().id, name, category, price, ts
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`
        WHERE META().id IN ['prod::103','prod::104'];"
  ```
  ![cbase24]({{ page.images_base | relative_url }}/24.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. **Conflicto controlado** y resolución **LWW**.

Actualizarás **la misma clave** en ambos clústeres con *timestamps distintos*, para que LWW resuelva a favor del **mayor** `ts`.

- **Paso 1.** Crea el siguiente documento base en ambos (id `bidi::conflict`).

  ```bash
  docker exec -it "${SRC_NODE}" cbq -e "http://${SRC_HOST}:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "UPSERT INTO \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` (KEY,VALUE)
        VALUES (\"bidi::conflict\", {\"type\":\"product\", \"name\":\"Item C\", \"price\":10.00, \"ts\":NOW_MILLIS()});"
  ```
  ![cbase25]({{ page.images_base | relative_url }}/25.png)

- **Paso 2.** Confirma que se encuentra en el **Destino**.

  ```bash
  docker exec -it "${DST_NODE}" cbq -e "http://${DST_HOST}:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT META().id, name, price, ts
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`
        WHERE META().id='bidi::conflict';"
  ```
  ![cbase26]({{ page.images_base | relative_url }}/26.png)

- **Paso 3.** Ahora, realiza una actualizaciones divergente (con `ts` diferente). Ejecuta todo el bloque junto para simular la concurrencia.

  ```bash
  # Update en ORIGEN (ts visible en el doc, solo para inspección)
  docker exec -it "${SRC_NODE}" cbq -e "http://${SRC_HOST}:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "UPSERT INTO \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` (KEY,VALUE)
        VALUES ('bidi::conflict', {\"type\":\"product\", \"name\":\"Item C - SRC\", \"price\":11.00, \"ts\":NOW_MILLIS()});"

  # Pequeño delay para asegurar mutación posterior en DESTINO
  sleep 1

  # Update en DESTINO (esta debería ganar en LWW)
  docker exec -it "${DST_NODE}" cbq -e "http://${DST_HOST}:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "UPSERT INTO \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` (KEY,VALUE)
        VALUES ('bidi::conflict', {\"type\":\"product\", \"name\":\"Item C - DST\", \"price\":12.00, \"ts\":NOW_MILLIS()});"
  ```
  ![cbase27]({{ page.images_base | relative_url }}/27.png)

- **Paso 4.** Valida el valor final convergente en ambos lados.

  ```bash
  # Origen
  docker exec -it "${SRC_NODE}" cbq -e "http://${SRC_HOST}:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT META().id, name, price, ts
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`
        WHERE META().id='bidi::conflict';"

  # Destino
  docker exec -it "${DST_NODE}" cbq -e "http://${DST_HOST}:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT META().id, name, price, ts
        FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`
        WHERE META().id='bidi::conflict';"
  ```
  ![cbase28]({{ page.images_base | relative_url }}/28.png)

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
  ![cbase29]({{ page.images_base | relative_url }}/29.png)

- **Paso 2.** Apagar y eliminar contenedor (se conservan los datos en ./data).

  > **Nota.** Si es necesario, puedes volver a activar los contenedores con el comando **`docker compose up -d`**.
  {: .lab-note .info .compact}
  > **Importante.** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase30]({{ page.images_base | relative_url }}/30.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
