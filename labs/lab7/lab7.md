---
layout: lab
title: "Práctica 7: Conexión de un SDK a Couchbase" # CAMBIAR POR CADA PRACTICA
permalink: /lab7/lab7/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab7/img # CAMBIAR POR CADA PRACTICA
duration: "20 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Levantar un nodo de Couchbase en Docker (local), crear un bucket/colecciones y un usuario de aplicación con permisos mínimos, e implementar una app **Node.js** (SDK oficial de Couchbase) que realice operaciones básicas *upsert/get*, *query N1QL* y *subdocumentos*.
prerequisites: # CAMBIAR POR CADA PRACTICA
  - Visual Studio Code con terminal **Git Bash**.
  - 6–8 GB de RAM libres recomendados para 3 nodos (mínimo ~5 GB).  
  - Puertos locales disponibles **`8091–8096`**, **`11210`** (se publicarán solo desde el nodo 1).
  - Node.js 18+ (recomendado) y **npm** disponibles en el host.
  - Conectividad a Internet para descargar la imagen.
  - Opcional `jq` para mejorar salidas JSON.
introduction: | # CAMBIAR POR CADA PRACTICA
  En esta práctica, crearás una carpeta nueva, un `compose.yaml` mínimo para Couchbase, inicializarás el clúster y recursos de datos, y escribirás una aplicación Node.js que se conecte mediante el SDK oficial. Validarás la conectividad ejecutando operaciones KV y consultas N1QL. Además, incluirás validaciones y troubleshooting para entornos reales.
slug: lab7 # CAMBIAR POR CADA PRACTICA
lab_number: 7 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Un entorno con Couchbase en Docker, bucket/colección y un **usuario RBAC mínimo**; además, una **app Node.js** conectándose con el **SDK** para realizar operaciones KV, subdocumentos y N1QL con validaciones y pruebas de error.
notes: | # CAMBIAR POR CADA PRACTICA
  - **TLS:** para producción usa `couchbases://` y el puerto 11207, instalando certificados de confianza (ver Práctica 6).  
  - **Índices:** evita el índice primario en producción; define índices secundarios adecuados.  
  - **Permisos:** aplica el principio de **menor privilegio** y separa *usuarios por servicio*.  
  - **Versiones:** si usas proxies o firewalls locales, asegúrate de permitir 8091–8096 y 11210.  
  - **Node.js:** si trabajas con **nvm**, ejecuta `nvm use 18` o superior antes de `npm install`.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: SDK Node.js (conceptos y API)
    url: https://docs.couchbase.com/nodejs-sdk/current/hello-world/start-using-sdk.html  
  - text: N1QL (lenguaje de consultas)
    url: https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/index.html  
  - text: Puertos de Couchbase
    url: https://docs.couchbase.com/server/current/install/install-ports.html
    
prev: /lab6/lab6/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab8/lab8/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1. Preparar la estructura de trabajo

Organizarás carpetas para aislar datos, configuración, logs y el código de la app; abrirás VS Code y validarás Docker.

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**.
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando, debes estar en el directorio **`cs300-labs`**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica7-sdk/
  mkdir -p practica7-sdk/couchbase
  mkdir -p practica7-sdk/couchbase/{data,logs,config}
  mkdir -p practica7-sdk/scripts
  mkdir -p practica7-sdk/app
  cd practica7-sdk
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC**, copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **Nota.** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-sdk-node1
  CB_HOST=127.0.0.1
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'
  CB_SERVICES=data,index,query
  CB_RAM=2048
  CB_INDEX_RAM=512

  CB_BUCKET=appbucket
  CB_BUCKET_RAM=256
  CB_SCOPE=appscope
  CB_COLLECTION=widgets

  APP_USER=app_user
  APP_PASS='labpassapp'
  EOF
  ```

- **Paso 5.** Ahora, crea el archivo **Docker Compose** llamado **`compose.yaml`**. Copia y pega el siguiente código en la terminal.

  > **Nota**
  - El archivo `compose.yaml` mapea puertos `8091–8096` para la consola web y `11210` para `clientes`.
  - El healthcheck consulta el endpoint `/pools` que responde cuando el servicio está arriba (aunque aún no inicializado).
  {: .lab-note .info .compact}

  ```bash
  cat > compose.yaml << 'YAML'
  services:
    couchbase:
      image: couchbase/server:${CB_TAG}
      container_name: ${CB_CONTAINER}
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

- **Paso 6.** Inicia el servicio. Dentro de la terminal, ejecuta el siguiente comando.

  > **Importante**
  > 
  > Para agilizar los procesos, la imagen ya está descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  > El `docker compose up -d` corre en segundo plano. El healthcheck del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en `8091` antes de continuar.
  {: .lab-note .important .compact}


  ```bash
  docker compose up -d
  ```
  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 7.** Inicializa el clúster y ejecuta el siguiete comando en la terminal.

  > **Nota.** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para `Index` es razonable; ajusta según tu RAM. Habilitar `flush` permite vaciar el bucket desde la UI o CLI.
  {: .lab-note .info .compact}
  > **Importante.** El comando se ejecuta desde el directorio **`practica7-sdk`**.
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
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

- **Paso 8.** Verifica que el clúster esté **healthy** y que se muestre el json con las propiedades del nodo.

  > **Nota.**
  - Contenedor `cb-sdk-node1` aparece **`Up`**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-sdk-node1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear un bucket, una colección y un usuario de app


- **Paso 1.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${CB_BUCKET}" \
    --bucket-type couchbase \
    --bucket-ramsize ${CB_BUCKET_RAM} \
    --enable-flush 1
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 2.** Ahora, crea el `Scope`.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" \
    -d "name=${CB_SCOPE}"
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 3.** Con este comando, crea el `Collection`.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLLECTION}"
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 4.**  Crea el índice primario para N1QL.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq \
    -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s 'CREATE PRIMARY INDEX `ix_primary_'"${CB_COLLECTION}"'` ON `'"${CB_BUCKET}"'`.`'"${CB_SCOPE}"'`.`'"${CB_COLLECTION}"'`;'
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)
 
- **Paso 5.** Crea el usuario de aplicación con permisos mínimos sobre la colección.

  ```bash
  docker exec -it "${CB_CONTAINER}" couchbase-cli user-manage \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --set --rbac-username "${APP_USER}" --rbac-password "${APP_PASS}" \
    --auth-domain local \
    --roles "data_reader[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],data_writer[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],query_select[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],query_insert[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],query_update[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}]"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 6.** Verifica que el bucket esté creado correctamente.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-list \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 7.** Verifica que el `Scope` sea correcto.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" | head -c 400; echo
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Inicializar proyecto Node.js e instalar SDK

- **Paso 1.** Crea el proyecto e instala las dependencias del SDK Node.js.

  > **Importante.** Es posible que tarde un par de minutos, espera a que termine antes de avanzar.
  {: .lab-note .important .compact}

  ```bash
  cd app
  npm init -y
  npm install couchbase dotenv
  ```

- **Paso 2.** Ahora, crea el archivo `.env` de la app (en `app/.env`):

  > **Nota.** Este archivo se crea en el directorio **app**.
  {: .lab-note .info .compact}

    ```bash
  cat > .env << 'EOF'
  CB_CONNSTR=couchbase://127.0.0.1
  CB_USERNAME=app_user
  CB_PASSWORD=labpassapp
  CB_BUCKET=appbucket
  CB_SCOPE=appscope
  CB_COLLECTION=widgets
  EOF
  ```

- **Paso 3.** Crea el archivo `index.js` de ejemplo.

  > **Nota.** Este archivo se crea en el directorio **`app`**.
  {: .lab-note .info .compact}

  ```bash
  cat > index.js << 'EOF'
  require('dotenv').config();
  const couchbase = require('couchbase');

  (async () => {
    try {
      const {
        CB_CONNSTR, CB_USERNAME, CB_PASSWORD,
        CB_BUCKET, CB_SCOPE, CB_COLLECTION,
      } = process.env;

      // Conexión (no TLS). Para TLS usa couchbases:// y certificados.
      const cluster = await couchbase.connect(CB_CONNSTR, {
        username: CB_USERNAME,
        password: CB_PASSWORD,
      });

      // Referencias a bucket/scope/collection
      const bucket = cluster.bucket(CB_BUCKET);
      const scope = bucket.scope(CB_SCOPE);
      const coll = scope.collection(CB_COLLECTION);

      // 1) KV: upsert + get
      const docId = 'widget::1001';
      const payload = { id: docId, name: 'gadget', qty: 5, ts: Date.now() };
      await coll.upsert(docId, payload);
      const got = await coll.get(docId);
      console.log('GET payload:', got.content);

      // 2) Subdocumento: mutateIn + lookupIn
      await coll.mutateIn(docId, [
        couchbase.MutateInSpec.upsert('qty', 6),
        couchbase.MutateInSpec.upsert('tags', ['lab', 'sdk']),
      ]);

      const lookup = await coll.lookupIn(docId, [
        couchbase.LookupInSpec.get('qty'),
        couchbase.LookupInSpec.exists('name'),
      ]);

      const qty = lookup.content[0].value;   // valor de 'qty'
      const nameExists = lookup.content[1].exists; // booleano
      console.log('Subdoc qty=', qty, 'nameExists=', nameExists);

      // 3) N1QL con parámetros (keyspace con backticks)
      const q = 'SELECT META().id, name, qty FROM `' + CB_BUCKET + '`.`' + CB_SCOPE + '`.`' + CB_COLLECTION + '` WHERE qty >= $min ORDER BY qty DESC LIMIT 5;';
      const result = await cluster.query(q, { parameters: { min: 1 } });
      console.log('Query rows:', result.rows);

      await cluster.close();
      console.log('OK: SDK operando.');
    } catch (err) {
      console.error('ERROR SDK:', err);
      process.exit(1);
    }
  })();
  EOF
  ```
- **Paso 4.** Ejecuta la aplicación.

  > **Nota.** El comando se ejecuta dentro del directorio **`app`**
  {: .lab-note .info .compact}

  ```bash
  node index.js
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Pruebas adicionales y troubleshooting

- **Paso 1.**  Realiza una prueba con las credenciales erróneas (se espera error de `auth`).

  ```bash
  CB_PASSWORD='Wrong!2025' node index.js || echo "Fallo esperado de autenticación."
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 2.** Comprueba los puertos y la salud del contenedor de Couchbase.

  {%raw%}
  ```bash
  set -a; source ../.env; set +a
  docker ps --filter "name=${CB_CONTAINER}"
  docker inspect -f '{{.State.Health.Status}}' ${CB_CONTAINER}
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  {%endraw%}
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 3.** Revisa los logs del contenedor como parte de los pasos de investigación.

  ```bash
  docker logs --tail=100 ${CB_CONTAINER}
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Limpieza

Opcionalmente, puedes bajar el servicio y borrar datos para repetir la práctica desde cero.

- **Paso 1.** En la terminal, aplica el siguiente comando para detener el nodo.

  > **Nota.** Si es necesario, puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 2.** Apaga y elimina el contenedor (se conservan los datos en `./data`).

  > **Nota.** Si es necesario, puedes volver a activar los contenedores con el comando **`docker compose up -d`**.
  {: .lab-note .info .compact}
  > **Importante.** Es normal el mensaje del objeto de red **`No resource found to remove`**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
