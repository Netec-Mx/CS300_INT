---
layout: lab
title: "Práctica 24: Paquete de soporte con cbcollect_info" # CAMBIAR POR CADA PRACTICA
permalink: /lab24/lab24/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab24/img # CAMBIAR POR CADA PRACTICA
duration: "20 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Generar un **paquete de soporte** de Couchbase con `cbcollect_info`, practicando opciones clave, validando el contenido (manifiesto, logs, diagnósticos) y dejando evidencia reutilizable para escalar a soporte. Ejecutarás la recolección dentro del contenedor Docker y guardarás el ZIP en tu host.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 3–4 GB de RAM libres (servicios `kv,index,query`).
  - Utilidades del host `unzip`, `jq` (opcional para inspección). 
introduction: | # CAMBIAR POR CADA PRACTICA
  La propiedad `cbcollect_info` empaqueta **logs**, **configuración**, **estadísticas** y **diagnósticos** del nodo para facilitar troubleshooting. Couchbase permite **Log Redaction** (None / Partial / Full) para ocultar datos sensibles (por ejemplo, valores de documentos) dentro del archivo resultante. En esta práctica crearás **dos** paquetes: uno **sin redaction** y otro **con redaction parcial** y revisarás sus diferencias.
slug: lab24 # CAMBIAR POR CADA PRACTICA
lab_number: 24 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Generaste dos paquetes de soporte con `cbcollect_info`: uno **sin** redaction y otro **con** redaction **partial**. Validaste su contenido, comprobaste los marcadores de redacción y entendiste cómo entregar de forma **segura** la evidencia para troubleshooting o escalamiento a soporte.
notes: | # CAMBIAR POR CADA PRACTICA
  - Para **entornos regulados**, usa **redaction full** y valida antes de compartir externamente.  
  - Evita publicar paquetes en repositorios públicos; contienen configuración sensible.  
  - En clústeres grandes, considera `--threads` y ventanas de menor carga para minimizar el impacto.  
  - Con **TLS** habilitado, la herramienta recolecta certificados y configuraciones relacionadas para diagnóstico.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Log Redaction (niveles y configuración)
    url: https://www.couchbase.com/blog/log-redaction-in-couchbase-server-5-5/
  - text: La propiedad collect-logs-start/stop (CLI multinodo)
    url: https://docs.couchbase.com/server/current/cli/cbcli/couchbase-cli-collect-logs-start.html

prev: /lab23/lab23/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab1/lab1/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1. Carpeta de la práctica y variables.

Crearás una carpeta aislada con subdirectorios y un `.env` con variables consistentes para el laboratorio.

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**.
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, da clic en el ícono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha**.

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando, debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica24-cbcollect/
  mkdir -p practica24-cbcollect/couchbase/{data,logs,config}
  mkdir -p practica24-cbcollect/scripts
  cd practica24-cbcollect
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC**, copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **Nota.** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER=cb-collect-n1
  CB_HOST=couchbase1

  # Credenciales admin
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'

  # Servicios
  CB_SERVICES=data,index,query
  CLUSTER_RAM=3072
  INDEX_RAM=768


  # Datos de ejemplo
  APP_BUCKET=supportlab
  APP_SCOPE=shop
  APP_COLLECTION=events
  APP_BUCKET_RAM=256

  # Carpeta de salida (montada)
  COLLECT_OUT=/collects
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Docker Compose y salud del nodo.

Definirás un `compose.yaml` con un único nodo Couchbase y validarás que la API responde.

- **Paso 1.** Ahora, crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente código en la terminal.

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
      hostname: ${CB_HOST}
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

- **Paso 2.** Inicia el servicio, dentro de la terminal, ejecuta el siguiente comando.

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
  docker ps --filter "name=cb-collect-n1"
  docker inspect -f '{{.State.Health.Status}}' cb-collect-n1
  ```
  {%endraw%}
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Inicializar el clúster y crear un bucket “estrecho”.

Inicializarás el nodo, crearás un bucket/colección y registrarás algo de actividad para que el paquete contenga eventos reales.

- **Paso 1.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **Nota.** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **Importante.** El comando se ejecuta desde el directorio de la práctica **practica24-cbcollect**. Puede tardar unos segundos en inicializar.
  {: .lab-note .important .compact}

  ```bash
  set -a; source .env; set +a
  docker exec -it "$CB_CONTAINER" couchbase-cli cluster-init \
    --cluster 127.0.0.1 \
    --cluster-username "$CB_ADMIN" \
    --cluster-password "$CB_ADMIN_PASS" \
    --services "${SERVICES_NORM:-$CB_SERVICES}" \
    --cluster-ramsize "$CLUSTER_RAM" \
    --cluster-index-ramsize "$INDEX_RAM" \
    --index-storage-setting default
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

- **Paso 2.** Verifica que el clúster esté **healthy** y que se muestre el json con las propiedades del nodo.

  > **Nota.**
  - El contenedor `cb-collect-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexión es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-collect-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:8091/pools | head -c 200; echo
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 3.** Ejecuta el siguiente comando para la creación del bucket.

  ```bash
  docker exec -it ${CB_CONTAINER} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${APP_BUCKET}" --bucket-type couchbase \
    --bucket-ramsize ${APP_BUCKET_RAM} --enable-flush 1
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 4.** Ahora, crea el *Scope* **shop**.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes" \
    -d "name=${APP_SCOPE}"
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 5.** Con este comando crea el *Collection* **events**.

  ```bash
  curl -fsS -u "${CB_ADMIN}:${CB_ADMIN_PASS}" \
    -X POST "http://localhost:8091/pools/default/buckets/${APP_BUCKET}/scopes/${APP_SCOPE}/collections" \
    -d "name=${APP_COLLECTION}"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 6.** Crea el índice primario temporal para validaciones.

  ```bash
  docker exec -it "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=false \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON ${APP_BUCKET}.${APP_SCOPE}.${APP_COLLECTION};"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 7.** Carga 30 documentos (loop con N1QL).

  > **Importante.** Es normal que la terminal se quede en espera, ya que está insertando los 30 registros. El proceso finalizará solo. Epera unos minutos.
  {: .lab-note .important .compact}

  ```bash
  set -euo pipefail
  KEYSPACE="\`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`"
  for i in $(seq 1 30); do
    name="Item-$i"

    # Genera precio entre 1.00 y 9.99
    cents=$((RANDOM%900 + 100))             # 100..999  => 1.00..9.99
    price=$(printf "%d.%02d" $((cents/100)) $((cents%100)))

    # Arma la sentencia N1QL con claves JSON
    n1ql=$(printf 'UPSERT INTO %s (KEY,VALUE) VALUES ("product::%d", {"type":"product","name":"%s","price":%s,"ts":NOW_MILLIS()});' \
                  "$KEYSPACE" "$i" "$name" "$price")

    docker exec -i "${CB_CONTAINER}" cbq \
      -e "http://127.0.0.1:8093" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" -q=true \
      -s "$n1ql" >/dev/null
  done
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 8.** Verifica que se hayan cargado correctamente, realiza la consulta de conteo.

  ```bash
  docker exec -i "${CB_CONTAINER}" cbq -e "http://127.0.0.1:8093" -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" -q=false \
    -s "SELECT COUNT(*) AS n
        FROM \`${APP_BUCKET}\`.\`${APP_SCOPE}\`.\`${APP_COLLECTION}\`
        WHERE META().id LIKE 'product::%';"
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Recolección **sin** log redaction.

Ejecutarás `cbcollect_info` con configuración por defecto (sin redaction) y validarás el archivo resultante.

- **Paso 1.** Ejecuta `cbcollect_info` dentro del contenedor.

  > **Nota.** Si detectas algunos codigos de error, es normal, ya que no están muchos paquetes instalados. El proceso puede tardar de **1 a 5 minutos**, termina solo. 
  {: .lab-note .info .compact}

  {%raw%}
  ```bash
  # === Config ===
  CB_CONTAINER="${CB_CONTAINER:-cb-collect-n1}"
  OUT_IN="/opt/couchbase/var/lib/couchbase/logs"

  # Verifica que el contenedor exista
  docker ps --format '{{.Names}}' | grep -Fx "$CB_CONTAINER" >/dev/null || {
    echo "ERROR: No existe el contenedor '$CB_CONTAINER'"; exit 1; }

  # === Genera el cbcollect (SIN redacción) ===
  docker exec -u 0 -i "$CB_CONTAINER" bash -lc '
    set -e
    OUT_IN="/opt/couchbase/var/lib/couchbase/logs"
    mkdir -p "$OUT_IN"

    TS="$(date +%Y%m%d-%H%M%S)"
    OUT_FILE="cbcollect_no_redaction_${TS}.zip"

    echo ">> Ejecutando cbcollect_info (sin redaction). Esto puede tardar..."
    /opt/couchbase/bin/cbcollect_info "$OUT_IN/$OUT_FILE"

    echo ">> Últimos ZIPs en $OUT_IN:"
    ls -lt "$OUT_IN"/*.zip 2>/dev/null | head -n5
  '
  ```
  {%endraw%}
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 2.** Verifica el archivo y cópialo al host.

  ```bash
  LATEST_NR=$(docker exec -i "$CB_CONTAINER" bash -lc \
    'ls -1t /opt/couchbase/var/lib/couchbase/logs/*.zip 2>/dev/null | grep -i "cbcollect_no_redaction_" | head -n1')

  [ -n "$LATEST_NR" ] || { echo "ERROR: No se encontró ZIP sin redaction."; exit 1; }

  mkdir -p collects
  docker cp "$CB_CONTAINER:$LATEST_NR" collects/ || { echo "ERROR: docker cp falló"; exit 1; }

  BASENAME_NR="$(basename "$LATEST_NR")"
  echo ">> Archivo copiado a: collects/$BASENAME_NR"

  # Validaciones rápidas
  ls -lh collects | grep cbcollect_no_redaction_ || true

  # (Opcional) Listar contenido del ZIP sin extraer
  if command -v unzip >/dev/null 2>&1; then
    unzip -l "collects/$BASENAME_NR"
  elif command -v jar >/dev/null 2>&1; then
    (cd collects && jar tf "$BASENAME_NR")
  elif command -v 7z >/dev/null 2>&1; then
    7z l "collects/$BASENAME_NR"
  else
    echo "Nota: instala 'unzip', o usa 'jar tf' / '7z l' para listar."
  fi
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Recolección **con** log redaction (nivel **partial**).

Activarás redaction **partial** para ocultar datos sensibles de usuario y generarás un segundo paquete, comparándolo con el primero.

- **Paso 1.** Inicia la recolección redacted con CLI.

  > **Nota.** En clústeres multinodo, `collect-logs-start/stop` recolecta desde *todos* los nodos. Aquí tenemos un solo nodo.
  {: .lab-note .info .compact}

  ```bash
  # === Vars ===
  CB_CONTAINER="${CB_CONTAINER:-cb-collect-n1}"
  CB_ADMIN="${CB_ADMIN:-admin}"
  CB_ADMIN_PASS="${CB_ADMIN_PASS:-adminlab}"
  OUT_IN="/opt/couchbase/var/lib/couchbase/logs"

  # Inicia recolección con redacción 'partial' en TODOS los nodos (sirve también si solo hay 1)
  docker exec "$CB_CONTAINER" couchbase-cli collect-logs-start \
    -c 127.0.0.1:8091 -u "$CB_ADMIN" -p "$CB_ADMIN_PASS" \
    --all-nodes \
    --redaction-level partial

  # Espera unos segundos para que junte material y detén para consolidar ZIP
  sleep 12
  docker exec "$CB_CONTAINER" couchbase-cli collect-logs-stop \
    -c 127.0.0.1:8091 -u "$CB_ADMIN" -p "$CB_ADMIN_PASS"

  # Revisa los últimos ZIPs generados dentro del contenedor
  docker exec "$CB_CONTAINER" bash -lc "ls -lt $OUT_IN/*.zip 2>/dev/null | head -n5"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 2.** Copia el archivo al host y valida el ZIP redacted.

  ```bash
  # === Detecta el ZIP más reciente y cópialo al host ===
  CB_CONTAINER="${CB_CONTAINER:-cb-collect-n1}"
  OUT_IN="/opt/couchbase/var/lib/couchbase/logs"

  # ZIP más reciente dentro del contenedor
  LATEST=$(docker exec "$CB_CONTAINER" bash -lc "ls -1t $OUT_IN/*.zip 2>/dev/null | head -n1")
  [ -n "$LATEST" ] || { echo "ERROR: No se encontró ningún ZIP en $OUT_IN"; exit 1; }
  echo "ZIP en contenedor: $LATEST"

  # Copia a carpeta local ./collects
  mkdir -p collects
  docker cp "$CB_CONTAINER:$LATEST" collects/ || { echo "ERROR: docker cp falló"; exit 1; }
  ZIP_LOCAL="collects/$(basename "$LATEST")"
  echo ">> Archivo copiado a: $ZIP_LOCAL"

  # Validaciones (tamaño y contenido)
  ls -lh "$ZIP_LOCAL"
  unzip -l "$ZIP_LOCAL" | head -n40

  # (Opcional) Busca tags de redacción <ud>…</ud> en una muestra
  unzip -p "$ZIP_LOCAL" | grep -m1 -o "<ud>.*</ud>" || echo "No se detectaron tags <ud> en la muestra (puede depender del contenido recolectado)."
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Comparativa y verificación de redaction.

Comprobarás que en el paquete redacted aparecen **marcadores** de redacción y que el manifiesto registra el nivel usado.

- **Paso 1.** Extrae ambos ZIP y busca los marcadores `<ud>`.

  ```bash
  # === Limpieza suave (no aborta si no existen) ===
  rm -rf collects/no_redaction collects/redacted collects/_tmp1 collects/_tmp2 >/dev/null 2>&1 || true
  mkdir -p collects/no_redaction collects/redacted collects/_tmp1 collects/_tmp2

  # === Toma los 2 ZIPs más recientes (si sólo hay uno, avisa y sigue) ===
  CAND1=$(ls -1t collects/*.zip 2>/dev/null | sed -n '1p')
  CAND2=$(ls -1t collects/*.zip 2>/dev/null | sed -n '2p')

  echo "ZIP #1 (más reciente): ${CAND1:-<no encontrado>}"
  echo "ZIP #2               : ${CAND2:-<no encontrado>}"

  if [ -z "$CAND1" ] || [ -z "$CAND2" ]; then
    echo "WARN: Necesitas al menos 2 ZIPs (uno sin redaction y otro con partial)."
  fi

  # === Extrae sin romper la terminal si falla unzip ===
  [ -n "$CAND1" ] && unzip -q "$CAND1" -d collects/_tmp1 || echo "WARN: No pude extraer $CAND1"
  [ -n "$CAND2" ] && unzip -q "$CAND2" -d collects/_tmp2 || echo "WARN: No pude extraer $CAND2"

  # === Heurística principal: buscar marcadores <ud> (redacted) ===
  HAS_UD_1=$(grep -R -l '<ud>' collects/_tmp1 2>/dev/null | head -n1 || true)
  HAS_UD_2=$(grep -R -l '<ud>' collects/_tmp2 2>/dev/null | head -n1 || true)

  RED_ZIP=""; NO_ZIP=""; RED_DIR=""; NO_DIR=""

  if [ -n "$HAS_UD_1" ] && [ -z "$HAS_UD_2" ]; then
    RED_ZIP="$CAND1"; NO_ZIP="$CAND2"; RED_DIR="collects/_tmp1"; NO_DIR="collects/_tmp2"
  elif [ -z "$HAS_UD_1" ] && [ -n "$HAS_UD_2" ]; then
    RED_ZIP="$CAND2"; NO_ZIP="$CAND1"; RED_DIR="collects/_tmp2"; NO_DIR="collects/_tmp1"
  else
    # === Fallback: pistas en metadatos/manifiestos ===
    META1=$(grep -R -i -E 'redact|redaction' collects/_tmp1 2>/dev/null | head -n1 || true)
    META2=$(grep -R -i -E 'redact|redaction' collects/_tmp2 2>/dev/null | head -n1 || true)
    if [ -n "$META1" ] && [ -z "$META2" ]; then
      RED_ZIP="$CAND1"; NO_ZIP="$CAND2"; RED_DIR="collects/_tmp1"; NO_DIR="collects/_tmp2"
    elif [ -z "$META1" ] && [ -n "$META2" ]; then
      RED_ZIP="$CAND2"; NO_ZIP="$CAND1"; RED_DIR="collects/_tmp2"; NO_DIR="collects/_tmp1"
    else
      # === Último recurso: asume "el más reciente es redacted" (pero sin romper nada)
      [ -n "$CAND1" ] && RED_ZIP="$CAND1" && RED_DIR="collects/_tmp1"
      [ -n "$CAND2" ] && NO_ZIP="$CAND2" && NO_DIR="collects/_tmp2"
      echo "INFO: No pude distinguir por contenido; asumo más reciente = REDACTED."
    fi
  fi

  echo ">> Clasificación:"
  echo "   REDACTED    : ${RED_ZIP:-<indefinido>}"
  echo "   NO REDACTION: ${NO_ZIP:-<indefinido>}"

  # === Mueve a carpetas finales si existen ===
  [ -n "$RED_DIR" ] && [ -d "$RED_DIR" ] && rm -rf collects/redacted && mv "$RED_DIR" collects/redacted || true
  [ -n "$NO_DIR"  ] && [ -d "$NO_DIR"  ] && rm -rf collects/no_redaction && mv "$NO_DIR"  collects/no_redaction || true

  # === Evidencia de marcadores <ud> en REDACTED ===
  echo ">> Primeras coincidencias de <ud> en REDACTED (si existen):"
  grep -R --line-number --color=auto '<ud>' collects/redacted 2>/dev/null | head || \
    echo "INFO: No se encontraron <ud> (puede depender de los datos recolectados)."
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 2.** Revisar los manifiestos y metadatos de los archivos.

  ```bash
  # === Pistas de manifiesto/metadatos en REDACTED ===
  echo ">> Pistas de redaction/manifest en REDACTED:"
  grep -R --line-number --color=auto -i -E 'redact|manifest|redaction' collects/redacted 2>/dev/null | head || \
    echo "INFO: No se hallaron metadatos explícitos (varía por versión)."
  ```
  ```bash
  # === Tamaños y fechas de los ZIP originales ===
  [ -n "$RED_ZIP" ] && [ -f "$RED_ZIP" ] && ls -lh "$RED_ZIP" || true
  [ -n "$NO_ZIP"  ] && [ -f "$NO_ZIP"  ] && ls -lh "$NO_ZIP"  || true
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

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
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 2.** Apagar y eliminar el contenedor (se conservan los datos en ./data).

  > **Nota.** Si es necesario, puedes volver a activar los contenedores con el comando **`docker compose up -d`**.
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
