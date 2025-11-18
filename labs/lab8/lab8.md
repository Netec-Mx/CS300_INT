---
layout: lab
title: "Práctica 8: Inserción masiva de documentos" # CAMBIAR POR CADA PRACTICA
permalink: /lab8/lab8/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab8/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Realizar inserciones masivas de documentos en Couchbase de manera reproducible y segura. Levantarás un nodo en Docker, crearás bucket/scope/collection, cargarás datos en formato NDJSON con `cbimport` (alta velocidad) y, en paralelo, harás un cargado masivo con el SDK de Node.js (control fino de lotes, concurrencia y reintentos). Validarás conteos, muestreos y tiempos
prerequisites: # CAMBIAR POR CADA PRACTICA
  - Visual Studio Code con terminal **Git Bash**.
  - 6–8 GB de RAM libres recomendados para 3 nodos (mínimo ~5 GB).  
  - Puertos locales disponibles **8091–8096**, **11210** (se publicarán solo desde el nodo 1).
  - Node.js 18+ (recomendado) y **npm** disponibles en el host.
  - Conectividad a Internet para descargar la imagen.
  - Opcional `jq` para mejorar salidas JSON.
introduction: | # CAMBIAR POR CADA PRACTICA
  Couchbase facilita el “bulk load” de datos por varias vías. En esta práctica verás dos enfoques complementarios:  

  1. **`cbimport`** (dentro del contenedor): muy rápido y simple para NDJSON.  
  2. **SDK Node.js**: más control sobre claves, concurrencia, reintentos, durabilidad y validaciones en línea.
slug: lab8 # CAMBIAR POR CADA PRACTICA
lab_number: 8 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Has ejecutado cargas masivas por dos rutas: **`cbimport`** (alta velocidad) y **SDK Node.js** (control de flujo). Levantaste un entorno desde cero, generaste un dataset reproducible, ajustaste parámetros de rendimiento y realizaste validaciones de conteo y muestreo, dejando una guía replicable para futuras pruebas de ingestión.
notes: | # CAMBIAR POR CADA PRACTICA
  - **Claves**: al reimportar con las mismas claves (`o::0000001`…), `upsert` del SDK sobrescribe; `cbimport` generará error o actualizará según flags/versión. Cambia `_key` si deseas duplicar sin sobrescribir.  
  - **Índices**: el `COUNT(*)` puede ser costoso en datos grandes. Considera índices secundarios y filtros (`WHERE`) para validaciones más rápidas.  
  - **Durabilidad**: para ingestión más segura (más lenta), usa opciones de **durability** en el SDK (p. ej., persistTo/replicateTo o DurabilityLevel en clústeres con réplicas).  
  - **TLS**: para producción, usa `couchbases://` (puerto 11207) y certificados confiables. 
  - **Recursos**: si ves errores de cuota de memoria, reduce `CB_BUCKET_RAM`/`DOC_COUNT` o cierra apps que consuman RAM/CPU.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: cbimport JSON
    url: https://docs.couchbase.com/server/current/tools/cbimport-json.html 
  - text: SDK Node.js (cargas y ejemplos)
    url: https://docs.couchbase.com/nodejs-sdk/current/hello-world/start-using-sdk.html  
  - text: N1QL (lenguaje de consultas)
    url: https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/index.html
  - text: CLI de administración
    url: https://docs.couchbase.com/cloud/reference/command-line-tools.html
    
prev: /lab7/lab7/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab9/lab9/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Preparar la estructura de trabajo

Crearás una estructura independiente con carpetas para datos, configuración y scripts; abrirás el proyecto en VS Code y dejarás variables en `.env`.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica8-bulk/
  mkdir -p practica8-bulk/couchbase/{data,logs,config}
  mkdir -p practica8-bulk/scripts
  mkdir -p practica8-bulk/app
  mkdir -p practica8-bulk/datasets
  cd practica8-bulk
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-bulk-node1
  CB_HOST=127.0.0.1
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'
  CB_SERVICES=data,index,query
  CB_RAM=2048
  CB_INDEX_RAM=512

  CB_BUCKET=bulkdata
  CB_BUCKET_RAM=256
  CB_SCOPE=sales
  CB_COLLECTION=orders

  # Tamaño dataset para pruebas (puedes incrementar)
  DOC_COUNT=50000

  APP_USER=bulk_user
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

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM. Habilitar `flush` permite vaciar el bucket desde la UI o CLI.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica8-bulk**
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
  - Contenedor `cb-bulk-node1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-bulk-node1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Crear bucket/scope/collection/

Se creará el bucket de ingestión y se preparará el keyspace con índices mínimos para validaciones.

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

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Generar dataset NDJSON (controlable por tamaño)

Generarás un dataset reproducible en formato NDJSON (una línea = un documento) para cargarlo con `cbimport`.

#### Tarea 3.1

- **Paso 15.** Define el codigo para el generador de registros en Node.js.

  > **IMPORTANTE:** Verifica que el archivo se haya creado en el directorio **datasets**
  {: .lab-note .important .compact}

  ```bash
  cd datasets
  cat > gen-orders.js << 'EOF'
  // node gen-orders.js [count] --out orders.ndjson
  const fs = require('fs');

  const countArg = process.argv.find(a => /^\d+$/.test(a));
  const outIdx = process.argv.indexOf('--out');
  const outPath = outIdx >= 0 ? process.argv[outIdx + 1] : null;

  let n = parseInt(countArg ?? process.env.DOC_COUNT ?? '10000', 10);
  if (!Number.isFinite(n) || n <= 0) n = 10000;

  function rand(min, max) { return Math.floor(Math.random() * (max - min + 1)) + min; }

  const now = Date.now();
  const stream = outPath ? fs.createWriteStream(outPath, { encoding: 'utf8', flags: 'w' }) : process.stdout;

  for (let i = 1; i <= n; i++) {
    const id = `o::${String(i).padStart(7, '0')}`;
    const doc = {
      _key: id,
      type: "order",
      orderId: i,
      customerId: `c::${rand(1, 5000)}`,
      items: rand(1, 6),
      total: +(Math.random() * 500).toFixed(2),
      status: ["NEW", "PAID", "CANCELLED", "SHIPPED"][rand(0, 3)],
      ts: now - rand(0, 1000 * 60 * 60 * 24 * 30)
    };
    stream.write(JSON.stringify(doc) + '\n');
  }

  if (outPath) {
    stream.end(() => console.error(`OK: generado ${n} docs en ${outPath}`));
  }
  EOF
  ```

- **Paso 16.** Ahora ejecuta el archivo Node.js para la creacion del documento **orders.ndjson**

  ```bash
  set -a; source ../.env; set +a
  echo "DOC_COUNT=${DOC_COUNT}"
  node gen-orders.js --out orders.ndjson
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 17.** Verifica que se haya creado correctamente el archivo con los datos, ejecuta el siguiente comando.


  ```bash
  wc -l orders.ndjson
  head -n 2 orders.ndjson
  tail -n 2 orders.ndjson
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Inserción masiva con `cbimport` (alta velocidad)

Cargarás el NDJSON directamente en la collection con claves generadas desde `_key` y múltiples hilos.

#### Tarea 4.1

- **Paso 18.** Primero copia el archivo **NDJSON** al contenedor porque lo tenemos en el host, escribe el siguiente comando

  > **NOTA:** Recuerda que el comando se ejecuta dentro del directorio **datasets**
  {: .lab-note .info .compact}

  ```bash
  docker cp orders.ndjson "${CB_CONTAINER}":/tmp/orders.ndjson
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 19.** Carga el dataset al contenedor con la propiedad `cbimport`.

  > **NOTA:** Solo como **referencia** el dataset tambien se puede montaral contenedor: `docker cp orders.ndjson ${CB_CONTAINER}:/tmp/orders.ndjson`
  {: .lab-note .info .compact}

  ```bash
  docker exec -it "${CB_CONTAINER}" sh -lc '
  cbimport json \
    -c 127.0.0.1 \
    -u "'"${CB_ADMIN}"'" -p "'"${CB_ADMIN_PASS}"'" \
    -b "'"${CB_BUCKET}"'" \
    --scope-collection-exp "'"${CB_SCOPE}.${CB_COLLECTION}"'" \
    -f lines \
    -g "%_key%" \
    -t 4 \
    -d /tmp/orders.ndjson \
    -e /tmp/import.err -v
  '
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 20.** Ahora ejecuta el siguiente comando para la validación de la carga de documento.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq \
  -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
  -s "SELECT COUNT(*) AS cnt FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\`;"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 21.** Ahora muestra 3 documentos para re-validar que todo este cargado correctamente.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq \
    -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "SELECT META().id, total, status FROM \`${CB_BUCKET}\`.\`${CB_SCOPE}\`.\`${CB_COLLECTION}\` LIMIT 3;"
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Usuario de aplicación y ruta de inserción con SDK

Crearás un usuario limitado para la app y correrás un cargador en Node.js con control de lotes/concurrencia y reintentos.

#### Tarea 5.1

- **Paso 22.** Crea el usuario con el siguiente comando, ejecutalo en la terminal.

  > **NOTA:** Ahora debemos regresar un directorio a la raíz **practica8-bulk** para ejecutar el siguiente comando.
  {: .lab-note .info .compact}

  ```bash
  cd ..
  docker exec -it ${CB_CONTAINER} couchbase-cli user-manage \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --set --rbac-username "${APP_USER}" --rbac-password ${APP_PASS} \
    --auth-domain local \
    --roles "data_writer[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],data_reader[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}],query_select[${CB_BUCKET}:${CB_SCOPE}:${CB_COLLECTION}]"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 23.** Ahora dentro del directorio **app** se creara la variables de entorno que usara el usuario.

  ```bash
  cd app
  cat > .env << 'EOF'
  CB_CONNSTR=couchbase://127.0.0.1
  CB_USERNAME=bulk_user
  CB_PASSWORD=labpassapp
  CB_BUCKET=bulkdata
  CB_SCOPE=sales
  CB_COLLECTION=orders

  # Parámetros de carga
  LOAD_FILE=../datasets/orders.ndjson
  BATCH_SIZE=500
  CONCURRENCY=8
  EOF
  ```

- **Paso 24.** Ahora inicializa el proyecto de Node.js con los siguientes comandos.

  > **NOTA:** Estos comandos se ejecutan en el directorio **app** puede tardar unos minutos. Espera
  {: .lab-note .info .compact}

  ```bash
  npm init -y
  npm install couchbase dotenv
  ```

- **Paso 25.** Crea el archivo load.js que definira la logica para la carga masiva de los dockumentos.

  > **IMPORTANTE:** El Script es muy grande, en caso de que el espacio en la terminal sea poco. Crea el archivo manualmente y copia el contenido a partir de la linea que dice **require('dotenv').config();** hacia abajo.
  {: .lab-note .important .compact}

  ```bash
  cat > load.js << 'EOF'
  // load.js
  require('dotenv').config();
  const fs = require('fs');
  const readline = require('readline');
  const couchbase = require('couchbase');

  (async function main() {
    // === 1) Validación de variables de entorno ===
    const required = [
      'CB_CONNSTR','CB_USERNAME','CB_PASSWORD',
      'CB_BUCKET','CB_SCOPE','CB_COLLECTION','LOAD_FILE'
    ];
    const missing = required.filter(k => !process.env[k]);
    if (missing.length) {
      console.error('[CONFIG] Faltan variables .env:', missing.join(', '));
      process.exit(1);
    }

    const {
      CB_CONNSTR, CB_USERNAME, CB_PASSWORD,
      CB_BUCKET, CB_SCOPE, CB_COLLECTION,
      LOAD_FILE, BATCH_SIZE, CONCURRENCY, CB_DURABILITY
    } = process.env;

    const batchSize   = parseInt(BATCH_SIZE || '500', 10);
    const concurrency = parseInt(CONCURRENCY || '8', 10);
    const progressEvery = Math.max(1000, Math.floor(batchSize)); // imprime progreso cada N
    const durabilityMap = {
      none: null,
      majority: couchbase.DurabilityLevel.Majority,
      persistToMajority: couchbase.DurabilityLevel.PersistToMajority,
      majorityAndPersistOnMaster: couchbase.DurabilityLevel.MajorityAndPersistOnMaster
    };
    const durabilityLevel = durabilityMap[(CB_DURABILITY || 'none').toLowerCase()] ?? null;

    console.log('────────────────────────────────────────────────────────');
    console.log('[INICIO] Loader Couchbase (Node.js SDK)');
    console.log('[CONFIG] Archivo:', LOAD_FILE);
    console.log('[CONFIG] Bucket/Scope/Collection:', `${CB_BUCKET}.${CB_SCOPE}.${CB_COLLECTION}`);
    console.log('[CONFIG] batchSize:', batchSize, 'concurrency:', concurrency);
    console.log('[CONFIG] durabilityLevel:', durabilityLevel ? CB_DURABILITY : 'none');
    console.log('────────────────────────────────────────────────────────');

    // === 2) Conexión a Couchbase ===
    console.time('[TIEMPO] Conexión cluster');
    const cluster = await couchbase.connect(CB_CONNSTR, {
      username: CB_USERNAME,
      password: CB_PASSWORD,
    });
    const coll = cluster.bucket(CB_BUCKET).scope(CB_SCOPE).collection(CB_COLLECTION);
    console.timeEnd('[TIEMPO] Conexión cluster');

    // === 3) Estado y métricas ===
    let buffer = [];
    let total = 0;
    let parsed = 0;
    let invalidLines = 0;
    let failed = 0;
    let retried = 0;
    const started = Date.now();

    function hr() {
      const s = (Date.now() - started) / 1000;
      return `${s.toFixed(1)}s`;
    }
    function mem() {
      const mb = process.memoryUsage().rss / (1024 * 1024);
      return `${mb.toFixed(1)}MB`;
      }
    function eta(done, rate) {
      if (!rate) return 'N/A';
      const remaining = Math.max(0, estimatedTotal - done);
      const sec = remaining / rate;
      return `${Math.floor(sec/60)}m ${Math.round(sec%60)}s`;
    }

    // Estimación de total si el archivo tiene 1 JSON por línea (puede no ser exacto)
    const estimatedTotal = await new Promise((resolve) => {
      let lines = 0;
      const s = fs.createReadStream(LOAD_FILE);
      s.on('data', (chunk) => { for (let i=0;i<chunk.length;i++) if (chunk[i] === 10) lines++; });
      s.on('end', () => resolve(lines));
      s.on('error', () => resolve(0));
    });
    if (estimatedTotal) {
      console.log(`[INFO] Estimación de líneas (docs): ~${estimatedTotal.toLocaleString()}`);
    } else {
      console.log('[INFO] No se pudo estimar el total de líneas (no bloqueante).');
    }

    // === 4) Procesamiento por lotes con límite de concurrencia ===
    async function flush() {
      if (buffer.length === 0) return;
      const chunk = buffer;
      buffer = [];

      let idx = 0;
      async function worker() {
        while (idx < chunk.length) {
          const i = idx++;
          const doc = chunk[i];
          const key = doc._key ?? doc.id ?? doc.key;
          if (!key) {
            failed++;
            continue;
          }
          try {
            await coll.upsert(key, doc, durabilityLevel ? { durabilityLevel } : undefined);
          } catch (e) {
            // Reintento una vez
            try {
              retried++;
              await coll.upsert(key, doc, durabilityLevel ? { durabilityLevel } : undefined);
            } catch (e2) {
              failed++;
              console.error(`[ERROR] Fallo definitivo en ${key}: ${e2?.message || e2}`);
            }
          } finally {
            total++;
            if (total % progressEvery === 0) {
              const elapsed = (Date.now() - started) / 1000;
              const rate = total / Math.max(1, elapsed);
              const pct = estimatedTotal ? ((total / estimatedTotal) * 100).toFixed(1) : 'N/A';
              console.log(`[PROGRESO] ${total.toLocaleString()} docs • ${rate.toFixed(0)} docs/s • tiempo ${hr()} • mem ${mem()} • avance ${pct}% • ETA ${eta(total, rate)}`);
            }
          }
        }
      }
      const workers = Array.from({ length: concurrency }, () => worker());
      await Promise.all(workers);
    }

    // === 5) Lector de líneas con pausa/resume para controlar memoria ===
    const rl = readline.createInterface({
      input: fs.createReadStream(LOAD_FILE, { encoding: 'utf8' })
    });

    // Reporte periódico (cada 5s)
    const ticker = setInterval(() => {
      const elapsed = (Date.now() - started) / 1000;
      const rate = total / Math.max(1, elapsed);
      console.log(`[TICK] parsed=${parsed.toLocaleString()} total=${total.toLocaleString()} failed=${failed} retried=${retried} invalidLines=${invalidLines} • ${rate.toFixed(0)} docs/s • mem ${mem()} • t=${hr()}`);
    }, 5000);

    // Manejo de Ctrl+C para cierre elegante
    let shuttingDown = false;
    async function shutdown(code = 0) {
      if (shuttingDown) return;
      shuttingDown = true;
      clearInterval(ticker);
      console.log('\n[SHUTDOWN] Vaciando buffer y cerrando cluster...');
      try { await flush(); } catch {}
      try { await cluster.close(); } catch {}
      const elapsed = (Date.now() - started) / 1000;
      const rate = total / Math.max(1, elapsed);
      console.log('────────────────────────────────────────────────────────');
      console.log('[RESUMEN]');
      console.log('   docs insertados:', total.toLocaleString());
      console.log('   reintentos:', retried);
      console.log('   fallos definitivos:', failed);
      console.log('   líneas inválidas:', invalidLines);
      console.log('   tiempo total:', `${elapsed.toFixed(1)}s`);
      console.log('   rendimiento medio:', `${rate.toFixed(0)} docs/s`);
      console.log('   memoria RSS final:', mem());
      console.log('────────────────────────────────────────────────────────');
      process.exit(code);
    }
    process.on('SIGINT', () => { console.warn('\n[SEÑAL] SIGINT recibida (Ctrl+C).'); shutdown(0); });
    process.on('SIGTERM', () => { console.warn('\n[SEÑAL] SIGTERM recibida.'); shutdown(0); });

    console.time('[TIEMPO] Carga total');
    try {
      for await (const line of rl) {
        if (!line.trim()) continue;
        try {
          buffer.push(JSON.parse(line));
          parsed++;
        } catch (e) {
          invalidLines++;
          console.error('[WARN] Línea inválida, se omite:', line.slice(0, 120));
        }

        if (buffer.length >= batchSize) {
          rl.pause();           // ✦ evita que crezca el buffer mientras flusheamos
          await flush();
          rl.resume();
        }
      }
      // Flush final
      await flush();
      console.timeEnd('[TIEMPO] Carga total');
    } catch (e) {
      console.error('[ERROR loader]:', e);
      await shutdown(1);
      return;
    }

    clearInterval(ticker);
    await shutdown(0);
  })();
  ```

- **Paso 26.** Ejecuta el script para realizar la carga.

  ```bash
  node load.js
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Usuario de aplicación y ruta de inserción con SDK

Correrás consultas para verificar estructura, rangos y distribución simple del dataset.

#### Tarea 6.1

- **Paso 27.** Verifica las estadisticas sobre los registros insertados.

  ```bash
  docker exec -it ${CB_CONTAINER} cbq -e http://127.0.0.1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "SELECT status, COUNT(*) AS n FROM ${CB_BUCKET}.${CB_SCOPE}.${CB_COLLECTION} GROUP BY status;"
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 28.** Verificación del rango de fecha de los documentos insertados.

  > **IMPORTANTE:** En algunos campos se muestra en **milisegundos** la consulta se adapto para que se mostraran los dias o fechas en UTC.
  {: .lab-note .important .compact}

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq -e http://127.0.0.1:8093 \
    -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "SELECT
          COUNT(1) AS docs,
          MIN(s.t) AS min_ts_ms,
          MAX(s.t) AS max_ts_ms,
          MILLIS_TO_STR(MIN(s.t)) AS min_ts_iso_utc,
          MILLIS_TO_STR(MAX(s.t)) AS max_ts_iso_utc,
          DATE_DIFF_STR(MILLIS_TO_STR(MAX(s.t)), MILLIS_TO_STR(MIN(s.t)), \"day\") AS rango_dias
        FROM (
          SELECT CASE
            WHEN ISNUMBER(ts) THEN ts
            WHEN ISSTRING(ts) THEN TO_NUMBER(ts)
            ELSE MISSING
          END AS t
          FROM ${CB_BUCKET}.${CB_SCOPE}.${CB_COLLECTION}
        ) AS s
        WHERE s.t IS NOT MISSING;"
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Limpieza

Borrar datos en el entorno para repetir pruebas.

#### Tarea 7.1

- **Paso 29.** Realiza el vaciado del bucket eliminara todos los documentos.

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
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

- **Paso 30.** En la terminal aplica el siguiente comando para detener el nodo.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

- **Paso 31.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase24]({{ page.images_base | relative_url }}/24.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}