---
layout: lab
title: "Práctica 15: Eventing simple y consulta en Analytics" # CAMBIAR POR CADA PRACTICA
permalink: /lab15/lab15/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab15/img # CAMBIAR POR CADA PRACTICA
duration: "30 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar Couchbase y configurar **Eventing** (función simple que audita mutaciones a otra colección) y **Analytics** (dataset y consulta SQL++ for Analytics) en un solo nodo. Validarás el flujo extremo a extremo **mutación → función Eventing → documento auditado → consulta en Analytics**.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 4–6 GB de RAM libres (servicios `kv,index,query`).
introduction: | # CAMBIAR POR CADA PRACTICA
  **Eventing** permite reaccionar a mutaciones (insert/update/delete) en tiempo casi real para ejecutar lógica (JavaScript) y escribir resultados en otros keyspaces. **Analytics** ejecuta SQL++ paralelo sobre datasets con ingesta continua desde buckets/colecciones, ideal para agregaciones sin impactar el plano transaccional. Aquí crearás una función Eventing que copia y enriquece documentos de `app.products` hacia `audit.products_audit`, y luego consultarás los auditados desde Analytics. (La API REST de Analytics usa el puerto `8095`; la de Eventing expone endpoints en `/api/v1` y puede accederse directo en `8096` o vía proxy en `8091`.)
slug: lab15 # CAMBIAR POR CADA PRACTICA
lab_number: 15 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Has creado una **función Eventing** que audita mutaciones desde `app.shop.products` a `audit.audit.products_audit` y consultaste esos datos con **Analytics** vía REST, validando ingesta y agregaciones. Dominas ahora el flujo básico **operacional → Eventing → analítico** en un solo nodo.
notes: | # CAMBIAR POR CADA PRACTICA
  - En producción, **Eventing** suele escalar con múltiples *workers* y requiere plan de **metadata bucket** dedicado y backups del mismo. 
  - Puedes consumir Analytics también con **SDKs** usando el endpoint de Query de Analytics.  
  - Para procesar históricos además de `from_now`, usa `dcp_stream_boundary="everything"` (procesa backlog). 
  - La **API de Eventing** y la **API de Analytics** están documentadas oficialmente y son estables para automatización.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Eventing REST API (overview y endpoints)
    url: https://docs.couchbase.com/server/current/eventing-rest-api/index.html
  - text: Básicos de Eventing y ejemplos de código
    url: https://docs.couchbase.com/server/current/eventing/eventing-examples.html
  - text: SQL++ for Analytics por REST (/analytics/service)
    url: https://docs.couchbase.com/server/current/analytics-rest-service/index.html
  - text: Detalles de la API de Analytics (puertos, enlaces)
    url: https://docs.couchbase.com/server/current/analytics/rest-analytics.html
  - text: Terminología/metadata de Eventing
    url: https://docs.couchbase.com/server/current/eventing/eventing-Terminologies.html
prev: /lab14/lab14/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab16/lab16/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Carpeta de práctica y variables

Crearás una carpeta aislada y un `.env` con variables para servicios y keyspaces.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica15-eventing-analytics/
  mkdir -p practica15-eventing-analytics/couchbase/{data,logs,config}
  mkdir -p practica15-eventing-analytics/scripts
  cd practica15-eventing-analytics
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-evan-n1
  CB_HOST=127.0.0.1
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'
  CB_SERVICES=data,index,query,eventing,analytics
  CB_RAM=2048
  CB_INDEX_RAM=512
  CB_EVENTING_RAM=256
  CB_ANALYTICS_RAM=1024

  # Buckets
  APP_BUCKET=app
  AUDIT_BUCKET=audit
  META_BUCKET=meta    # Eventing metadata

  # Scope/collections de trabajo
  APP_SCOPE=shop
  APP_COLL=products

  AUDIT_SCOPE=audit
  AUDIT_COLL=products_audit

  # Nombre de la función Eventing
  EV_FUNC=fn_audit_products
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Docker Compose y salud del nodo

Definirás un `compose.yaml` con un nodo único que expone `data,index,query,eventing,analytics` y healthcheck básico.

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
  docker ps --filter "name=cb-evan-n1"
  docker inspect -f '{{.State.Health.Status}}' cb-evan-n1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Inicializar clúster, buckets y colecciones

Inicializarás el clúster, crearás los buckets `app`, `audit` y `meta` (metadata de Eventing) y los keyspaces necesarios.

#### Tarea 3.1

- **Paso 8.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica15-eventing-analytics**. Puede tardar unos segundos en inicializar.
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
    --cluster-eventing-ramsize "${CB_EVENTING_RAM}" \
    --cluster-analytics-ramsize "${CB_ANALYTICS_RAM}" \
    --index-storage-setting default
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

- **Paso 9.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **NOTA:**
  - Contenedor `cb-evan-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-evan-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 10.** Ejecuta el siguiente comando para la creación de los bucket.

  ```bash
  for B in "${APP_BUCKET}" "${AUDIT_BUCKET}" "${META_BUCKET}"; do
    docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
      -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
      --bucket "${B}" --bucket-type couchbase --bucket-ramsize 512 --enable-flush 1
  done
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 11.** Ahora crea el *Scope* **app.shop.products**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes" \
    -d "name=${APP_SCOPE}"
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 12.** Con este comando crea el *Collection* **app.shop.products**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes/${APP_SCOPE}/collections" \
    -d "name=${APP_COLL}"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 13.** Ahora crea el *Scope* **audit.audit.products_audit**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${AUDIT_BUCKET}/scopes" \
    -d "name=${AUDIT_SCOPE}"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 14.** Con este comando crea el *Collection* **audit.audit.products_audit**

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${AUDIT_BUCKET}/scopes/${AUDIT_SCOPE}/collections" \
    -d "name=${AUDIT_COLL}"
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Carga de datos de ejemplo en `app.shop.products`

Insertarás documentos con campos básicos que activarán la función Eventing.

#### Tarea 4.1

- **Paso 15.** Crea 8 documentos de prueba (UPSERT vía N1QL)

  > **IMPORTANTE:** El resultado es iterativo para los 8 documentos la salida es muy grande, la imagen representa la sección de la inserción.
  {: .lab-note .important .compact}

  ```bash
  for i in 1 2 3 4 5 6 7 8; do
    case $i in
      1)  name="Desk Lamp";    catg="lighting"; price=19.99;  stock=20 ;;
      2)  name="LED Strip";    catg="lighting"; price=12.50;  stock=50 ;;
      3)  name="USB-C Cable";  catg="hardware"; price=5.99;   stock=200 ;;
      4)  name="Charger";      catg="hardware"; price=24.90;  stock=80 ;;
      5)  name="Ergo Chair";   catg="office";   price=159.00; stock=15 ;;
      6)  name="Desk";         catg="office";   price=199.00; stock=10 ;;
      7)  name="Smart Bulb";   catg="lighting"; price=9.99;   stock=300 ;;
      8)  name="Floor Lamp";   catg="lighting"; price=34.99;  stock=12 ;;
    esac

    json_payload=$(printf '{"type":"product","name":"%s","category":"%s","price":%s,"stock":%s,"ts_now":NOW_MILLIS()}' \
                            "$name" "$catg" "$price" "$stock")

    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://${CB_HOST}:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "UPSERT INTO \`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLL}\` (KEY,VALUE)
          VALUES (\"prod::${i}\", ${json_payload});"
  done
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 16.** Verificar conteo (índice primario temporal)

  ```bash
  docker exec -i "${CB_CONTAINER}" cbq \
    -e "http://${CB_HOST}:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "SELECT COUNT(*) AS n
        FROM \`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLL}\`
        WHERE META().id LIKE 'prod::%';"
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Crear y **desplegar** función Eventing (audit)

Definirás una función Eventing que se activa en mutaciones de `app.shop.products`, enriquece y **escribe** a `audit.audit.products_audit`. La crearás y desplegarás por REST.

#### Tarea 5.1

- **Paso 17.** Definir el código de la función (JavaScript Eventing) crea el handler OnUpdate/OnDelete que construye un documento de auditoría y lo escribe en el binding dst.

  ```bash
  cat > scripts/${EV_FUNC}.js << 'JS'
  function OnUpdate(doc, meta) {
    if (!doc.type || doc.type !== "product") return;
    var audit = {
      src_id: meta.id,
      name: doc.name,
      category: doc.category,
      price: doc.price,
      stock: doc.stock,
      ts_src: doc.ts_now || Date.now(),
      ts_audit: Date.now(),
      action: "upsert"
    };
    dst[meta.id] = audit; // binding "dst" -> audit.audit.products_audit
  }

  function OnDelete(meta, options) {
    var audit = { src_id: meta.id, ts_audit: Date.now(), action: "delete" };
    dst["del::" + meta.id] = audit;
  }
  JS
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 18.** Plantilla JSON del **Function Definition** con bindings que define el keyspace de origen, el bucket de metadata para Eventing y el binding dst con acceso rw.

  > **NOTA:** Este Python arma el JSON final con el formato correcto (source_keyspace, metadata_keyspace, buckets) y mete el JS en appcode.
  {: .lab-note .info .compact}

  ```bash
  python - <<'PY'
  import os, json
  e=os.environ
  cfg={
    "appname": e["EV_FUNC"],
    "depcfg": {
      "source_bucket": e["APP_BUCKET"],
      "source_scope": e["APP_SCOPE"],
      "source_collection": e["APP_COLL"],
      "metadata_bucket": e["META_BUCKET"],
      "metadata_scope": "_default",
      "metadata_collection": "_default",
      "bucket_bindings": [
        {
          "alias": "dst",
          "bucket_name": e["AUDIT_BUCKET"],
          "scope_name": e["AUDIT_SCOPE"],
          "collection_name": e["AUDIT_COLL"],
          "access": "rw"
        }
      ]
    },
    "settings": {
      "dcp_stream_boundary": "from_now",
      "log_level": "INFO",
      "deployment_status": True,
      "processing_status": True
    },
    "function_scope": { "bucket": e["APP_BUCKET"], "scope": e["APP_SCOPE"] },
    "appcode": open(f'scripts/{e["EV_FUNC"]}.js','r',encoding='utf-8').read()
  }
  out=f'scripts/{e["EV_FUNC"]}.deploy.json'
  json.dump(cfg, open(out,'w',encoding='utf-8'), ensure_ascii=False, indent=2)
  print(out)
  PY

  # Valida JSON
  jq -e . "scripts/${EV_FUNC}.deploy.json" >/dev/null && echo "JSON OK"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 19.** Publicar la función (un solo POST). Usa el proxy de 8091 que respondera “Successfully stored”.

  > **IMPORTANTE:** Si algo ocurre puedes borrar la funcion con este comando: `curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" -X DELETE "$EVENT_API/functions/${EV_FUNC}" >/dev/null 2>&1 || true`
  {: .lab-note .important .compact}

  ```bash
  EVENT_API="http://${CB_HOST}:8091/_p/event/api/v1"
  echo "EVENT_API=$EVENT_API"
  curl -i -s -u "$CB_ADMIN:$CB_ADMIN_PASS" \
    -H "Content-Type: application/json" \
    -X POST "$EVENT_API/functions/${EV_FUNC}" \
    --data-binary @scripts/${EV_FUNC}.deploy.json
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Crear **dataset de Analytics** y consultar

Configurarás un dataset en Analytics sobre la colección de auditoría y ejecutarás SQL++ for Analytics por REST (`8095`).

#### Tarea 6.1

- **Paso 20.** Probar que Analytics responde

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    --data-urlencode 'statement=SELECT 1 AS ok;' \
    "http://${CB_HOST}:8095/analytics/service"
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 21.**Crear dataverse (idempotente)

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    --data-urlencode 'statement=CREATE DATAVERSE dv_audit IF NOT EXISTS;' \
    "http://${CB_HOST}:8095/analytics/service"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 22.** Validar que existe en metadatos

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    --data-urlencode 'statement=SELECT DataverseName
                      FROM Metadata.`Dataverse`
                      WHERE DataverseName = "dv_audit";' \
    "http://${CB_HOST}:8095/analytics/service"
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 23.** Crear dataset sobre la colección de auditoría que define el dataset ds_audit que apunta a `${AUDIT_BUCKET}`.`${AUDIT_SCOPE}`.`${AUDIT_COLL}`.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    --data-urlencode "statement=USE dv_audit;
                      CREATE DATASET ds_audit
                      ON \`${AUDIT_BUCKET}\`.\`${AUDIT_SCOPE}\`.\`${AUDIT_COLL}\`;" \
    "http://${CB_HOST}:8095/analytics/service"
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 24.** Valida que se liste el dataset recién creado

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    --data-urlencode 'statement=SELECT DataverseName, DatasetName
                      FROM Metadata.`Dataset`
                      WHERE DataverseName = "dv_audit" AND DatasetName = "ds_audit";' \
    "http://${CB_HOST}:8095/analytics/service"
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

- **Paso 25.** Conecta el link Local (inicia ingesta) e inicia la ingesta continua desde data hacia Analytics.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    --data-urlencode 'statement=USE dv_audit; CONNECT LINK Local;' \
    "http://${CB_HOST}:8095/analytics/service"
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

- **Paso 26.** Verifica el estado del link

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    --data-urlencode 'statement=USE dv_audit; CONNECT LINK Local;' \
    "http://${CB_HOST}:8095/analytics/service"
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Limpieza

Borrar datos en el entorno para repetir pruebas.

#### Tarea 7.1

- **Paso 27.** En la terminal aplica el siguiente comando para detener el nodo.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase24]({{ page.images_base | relative_url }}/24.png)

- **Paso 28.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase25]({{ page.images_base | relative_url }}/25.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}