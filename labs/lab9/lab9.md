---
layout: lab
title: "Práctica 9: Operaciones CRUD con SDK" # CAMBIAR POR CADA PRACTICA
permalink: /lab9/lab9/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab9/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Crear un bucket/colección y un usuario de aplicación con privilegios mínimos, e implementar una app **Node.js** que ejecute operaciones **CRUD completas** (Create/Read/Update/Delete) con el **SDK oficial de Couchbase**, incluyendo **control de errores**, **optimistic locking con CAS**, **subdocumentos** y **consultas N1QL**.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Lenguaje **Node.js 18+** y **npm** instalados.  
  - Los **Puertos libres:** 8091–8096 (UI/REST), 11210 (KV no TLS).  
  - Internet para descargar la imagen y paquetes npm.
introduction: | # CAMBIAR POR CADA PRACTICA
  Vas a crear el entorno y código desde cero. Practicarás CRUD con características clave del SDK: **upsert/insert/get**, **replace/remove** con **CAS**, **mutateIn/lookupIn** (subdocumentos), y **N1QL** con parámetros. Verás validaciones y mensajes de error comunes (documento existente, no encontrado, CAS inválido).
slug: lab9 # CAMBIAR POR CADA PRACTICA
lab_number: 9 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Has desplegado Couchbase y desarrollaste un script **CRUD** completo con **CAS** y **subdocumentos**, además de consultas **N1QL**, validaciones y manejo de errores comunes. Esto sienta las bases para aplicaciones reales que requieren consistencia y rendimiento.
notes: | # CAMBIAR POR CADA PRACTICA
  - **TLS**: para producción usa `couchbases://` (11207) con certificados de confianza.  
  - **Índices**: evita el índice primario en producción; crea índices adecuados a tus consultas.  
  - **CAS**: úsalo en updates críticos para evitar sobrescrituras perdidas.  
  - **Durabilidad**: en clústeres con réplicas, considera `DurabilityLevel` para garantizar escritura replicada/persistida.  
  - **Namespacing**: prefiere claves con prefijos (`prod::`) para facilitar particionamiento y mantenimiento.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Operaciones KV y CAS
    url: https://docs.couchbase.com/nodejs-sdk/current/howtos/kv-operations.html
  - text: SDK Node.js (cargas y ejemplos)
    url: https://docs.couchbase.com/nodejs-sdk/current/hello-world/start-using-sdk.html  
  - text: N1QL (lenguaje de consultas)
    url: https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/index.html
  - text: Subdocumentos
    url: https://docs.couchbase.com/nodejs-sdk/current/howtos/subdocument-operations.html 
    
prev: /lab8/lab8/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab10/lab10/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Estructura base del proyecto

Crearás una carpeta aislada para esta práctica, con directorios para datos del contenedor, configuración y la app.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica9-crud/
  mkdir -p practica9-crud/couchbase/{data,logs,config}
  mkdir -p practica9-crud/scripts
  mkdir -p practica9-crud/app
  cd practica9-crud
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-crud-node1
  CB_HOST=127.0.0.1
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'
  CB_SERVICES=data,index,query
  CB_RAM=2048
  CB_INDEX_RAM=512

  CB_BUCKET=appbucket
  CB_BUCKET_RAM=256
  CB_SCOPE=appscope
  CB_COLLECTION=products

  APP_USER=crud_user
  APP_PASS='labpassapp'
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

- **Paso 6.** Inicia el servicio, dentro de la terminal ejecuta el siguiente comando.

  > **IMPORTANTE:** Para agilizar los procesos, la imagen ya esta descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  {: .lab-note .important .compact}
  > **IMPORTANTE:** El `docker compose up -d` corre en segundo plano. El healthcheck del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en 8091 antes de continuar.
  {: .lab-note .important .compact}

  ```bash
  docker compose up -d
  ```
  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 7.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica9-crud**. Puede tardar unos segundos en inicializar.
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

- **Paso 8.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **NOTA:**
  - Contenedor `cb-crud-node1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-crud-node1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Crear bucket/scope/collection/

Crearás bucket/scope/collection, un índice primario para pruebas y un usuario de app con privilegios mínimos.

#### Tarea 2.1

- **Paso 9.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${CB_BUCKET}" \
    --bucket-type couchbase \
    --bucket-ramsize ${CB_BUCKET_RAM} \
    --enable-flush 1
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 10.** Ahora crea el *Scope*.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" \
    -d "name=${CB_SCOPE}"
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 11.** Con este comando crea el *Collection*

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLLECTION}"
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 12.**  Crea los índices primarios para N1QL.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq \
    -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s 'CREATE PRIMARY INDEX `ix_primary_'"${CB_COLLECTION}"'` ON `'"${CB_BUCKET}"'`.`'"${CB_SCOPE}"'`.`'"${CB_COLLECTION}"'`;'
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 13.** Verifica que el bucket este creado correctamente.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-list \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 14.** Verifica que el *Scope* este correctamente.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" | head -c 400; echo
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 15.** Ahora crea el usuario de **app** (tendra menor privilegio: lectura/escritura/queries sobre la colección):

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli user-manage \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --set --rbac-username "${APP_USER}" --rbac-password ${APP_PASS} \
    --auth-domain local \
    --roles "data_reader[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],data_writer[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],query_select[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],query_insert[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],query_update[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],query_delete[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}]"
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Crear la app Node.js (CRUD completo)

Implementarás un script `crud.js` que conecte, haga **insert**, **get** (con proyección), **replace** con **CAS**, **mutateIn/lookupIn** y **remove**; además, **N1QL** con parámetros.

#### Tarea 3.1

- **Paso 16.** Ahora dentro del directorio **app** se creara la variables de entorno que usara el usuario.

  ```bash
  cd app
  cat > .env << 'EOF'
  CB_CONNSTR=couchbase://127.0.0.1
  CB_USERNAME=crud_user
  CB_PASSWORD=labpassapp
  CB_BUCKET=appbucket
  CB_SCOPE=appscope
  CB_COLLECTION=products
  EOF
  ```

- **Paso 17.** Ahora inicializa el proyecto de Node.js con los siguientes comandos.

  > **NOTA:** Estos comandos se ejecutan en el directorio **app** puede tardar unos minutos. Espera
  {: .lab-note .info .compact}

  ```bash
  npm init -y
  npm install couchbase dotenv
  ```

- **Paso 18.** Crea el archivo `crud.js` y la logica para las operaciones CRUD.

  ```bash
  cat > crud.js << 'EOF'
  require('dotenv').config();
  const couchbase = require('couchbase');

  async function main() {
    const {
      CB_CONNSTR, CB_USERNAME, CB_PASSWORD,
      CB_BUCKET, CB_SCOPE, CB_COLLECTION,
    } = process.env;

    console.log('➡️ Conectando a Couchbase…', CB_CONNSTR);
    const cluster = await couchbase.connect(CB_CONNSTR, {
      username: CB_USERNAME,
      password: CB_PASSWORD,
    });

    const bucket = cluster.bucket(CB_BUCKET);
    const scope  = bucket.scope(CB_SCOPE);
    const coll   = scope.collection(CB_COLLECTION);
    console.log(`✅ Conectado. Bucket=${CB_BUCKET} Scope=${CB_SCOPE} Collection=${CB_COLLECTION}`);

    const docId = 'prod::1001';
    const initial = { id: docId, sku: 'SKU-1001', name: 'Widget', price: 19.99, stock: 10, tags: ['new'] };

    // 1) CREATE
    console.log('\n=== CREATE (insert) ===');
    try {
      const ins = await coll.insert(docId, initial);
      console.log('INSERT OK, CAS=', ins.cas.toString('hex'));
    } catch (e) {
      if (e instanceof couchbase.DocumentExistsError) {
        console.log('INSERT omitido: el documento ya existía');
      } else {
        throw e;
      }
    }

    // 2) READ (proyección)
    console.log('\n=== READ (get con project) ===');
    const got = await coll.get(docId, { project: ['name', 'price'] });
    console.log('GET →', got.content, 'CAS=', got.cas.toString('hex'));

    // 3) REPLACE con CAS (optimistic locking)
    console.log('\n=== REPLACE (CAS) ===');
    const full = await coll.get(docId); // documento completo para modificar
    const current = full.content;
    current.price = 24.99;
    const rep = await coll.replace(docId, current, { cas: full.cas });
    console.log('REPLACE OK, nuevo CAS=', rep.cas.toString('hex'));

    // 4) SUB-DOCUMENT: lookupIn (lectura parcial)
    console.log('\n=== SUB-DOC lookupIn ===');
    const sub = await coll.lookupIn(docId, [
      couchbase.LookupInSpec.get('stock'),
      couchbase.LookupInSpec.get('tags[0]'),
    ]);
    const stock = sub.content[0].value;   // ✅ acceso correcto por índice
    const firstTag = sub.content[1].value;
    console.log('lookupIn → stock =', stock, ', firstTag =', firstTag);

    // 5) SUB-DOCUMENT: mutateIn (incrementar stock y push a tags)
    console.log('\n=== SUB-DOC mutateIn ===');
    const mut = await coll.mutateIn(docId, [
      couchbase.MutateInSpec.increment('stock', 5),
      couchbase.MutateInSpec.arrayPush('tags', 'promo'),
    ], { storeSemantics: couchbase.StoreSemantics.Replace });
    console.log('mutateIn OK. Nuevos valores subdoc (por operación):', mut.content);

    // 6) QUERY (SQL++) sobre la colección
    console.log('\n=== QUERY (SQL++) ===');
    const min = 10;
    const statement = `SELECT id, name, price
                      FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`
                      WHERE price >= $min
                      ORDER BY price DESC
                      LIMIT 3;`;
    const qres = await cluster.query(statement, { parameters: { min } });
    console.log('Top 3 por precio >=', min, '→ filas:', qres.rows);

    await cluster.close();
    console.log('\n✅ CRUD finalizada.');
  }

  main().catch(e => { console.error('ERROR CRUD:', e); process.exit(1); });
  EOF
  ```

- **Paso 19.** Ejecuta el script que realizara las operaciones.

  > **IMPORTANTE:**
  - **Crear:** Inserta un documento nuevo con clave única; falla si existe (usa upsert para reemplazar).
  - **Leer:** Recupera por clave; admite proyección de campos y control de concurrencia con CAS.
  - **Actualizar:** Modifica campos del documento; usa replace/upsert o mutateIn; valida versiones con CAS.
  - **Borrar:** Elimina el documento por clave; puede verificar existencia y manejar error si no está.
  {: .lab-note .important .compact}

  ```bash
  node crud.js
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Comprobaciones con N1QL y errores comunes

Ejecutarás consultas de verificación y forzarás errores típicos para entender su diagnóstico.

#### Tarea 4.1

- **Paso 20.** Verificar el conteo con el siguiente comando y muestreo. Ejecuta uno por uno los comandos.

  ```bash
  docker exec -it ${CB_CONTAINER} cbq -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "SELECT COUNT(*) AS n FROM ${CB_BUCKET}.${CB_SCOPE}.${CB_COLLECTION};"
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)
  ```bash
  docker exec -it ${CB_CONTAINER} cbq -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "SELECT META().id, name, price FROM ${CB_BUCKET}.${CB_SCOPE}.${CB_COLLECTION} LIMIT 5;"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 21.** Forzar **DocumentNotFoundError** (buscar una clave inexistente).

  ```bash
  node -e "require('dotenv').config();const couchbase=require('couchbase');(async()=>{const c=await couchbase.connect(process.env.CB_CONNSTR,{username:process.env.CB_USERNAME,password:process.env.CB_PASSWORD});const coll=c.bucket(process.env.CB_BUCKET).scope(process.env.CB_SCOPE).collection(process.env.CB_COLLECTION);try{await coll.get('noexiste::1');}catch(e){console.log('Esperado:',e.constructor.name)}await c.close()})()"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 22.** Forzar **DocumentExistsError** (insert de una clave existente).

  ```bash
  node -e "require('dotenv').config();const couchbase=require('couchbase');(async()=>{const c=await couchbase.connect(process.env.CB_CONNSTR,{username:process.env.CB_USERNAME,password:process.env.CB_PASSWORD});const coll=c.bucket(process.env.CB_BUCKET).scope(process.env.CB_SCOPE).collection(process.env.CB_COLLECTION);await coll.upsert('dup::1',{a:1});try{await coll.insert('dup::1',{a:2})}catch(e){console.log('Esperado:',e.constructor.name)}await c.close()})()"
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Limpieza

Borrar datos en el entorno para repetir pruebas.

#### Tarea 5.1

- **Paso 23.** Realiza el vaciado del bucket eliminara todos los documentos.

  > **NOTA:** Es normal que el comando mande un resultado vacio.
  {: .lab-note .info .compact}
  > **IMPORTANTE:**
  - **OPCIONAL** Si quieres verificar puedes escribir este comando.
  - `curl -s -u "${CB_ADMIN}:${CB_ADMIN_PASS}" "http://localhost:8091/pools/default/buckets/${CB_BUCKET}" | jq '.basicStats.itemCount'`
  {: .lab-note .important .compact}

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" -X POST \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/controller/doFlush"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 30.** En la terminal aplica el siguiente comando para detener el nodo.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 31.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}