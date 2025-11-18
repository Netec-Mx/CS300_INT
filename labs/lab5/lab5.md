---
layout: lab
title: "Práctica 5: Auditoría de accesos" # CAMBIAR POR CADA PRACTICA
permalink: /lab5/lab5/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab5/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Habilitar y configurar la auditoría de accesos en Couchbase (eventos de autenticación, cambios RBAC, operaciones de buckets/colecciones y consultas), generar eventos reales con usuarios de prueba, y validar los registros mediante CLI/REST y revisión de archivos de log.
prerequisites: # CAMBIAR POR CADA PRACTICA
  - Visual Studio Code con terminal **Git Bash**.
  - 6–8 GB de RAM libres recomendados para 3 nodos (mínimo ~5 GB).  
  - Puertos locales disponibles **8091–8096**, **11210** (se publicarán solo desde el nodo 1).
  - Conectividad a Internet para descargar la imagen.
  - Opcional `jq` para mejorar salidas JSON.
introduction: > # CAMBIAR POR CADA PRACTICA
  La auditoría (Audit Logging) registra “quién, qué, cuándo y desde dónde” ante eventos de seguridad y administración: inicios de sesión, cambios de roles, creación de buckets/colecciones, modificaciones de índices y ejecución de consultas, entre otros. En esta práctica activarás la auditoría, generarás eventos intencionales (éxitos y fallos) y validarás la información almacenada en los archivos de log y endpoints REST.
slug: lab5 # CAMBIAR POR CADA PRACTICA
lab_number: 5 # CAMBIAR POR CADA PRACTICA
final_result: > # CAMBIAR POR CADA PRACTICA
  Auditoría de Couchbase **activada y validada** autenticación (éxitos/fallos), cambios RBAC, operaciones administrativas e interacciones N1QL. Aplicaste filtros por usuario y verificaste contenidos en `audit.log`.
notes: # CAMBIAR POR CADA PRACTICA
  - El volumen de auditoría puede crecer rápidamente planifica rotación y retención.  
  - Si `jq` no está disponible, usa `grep`/`less` para inspeccionar JSON line-by-line.  
  - En despliegues multi-nodo, habilita la auditoría de manera consistente en todos los nodos.  
  - Redaction `partial` oculta claves/valores sensibles; `full` amplía la protección (más costo de soporte).  
  - Revisa permisos del sistema host para que el contenedor pueda escribir en `${AUDIT_DIR}` si lo montas como volumen.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Auditoría en Couchbase
    url: https://docs.couchbase.com/server/current/manage/manage-security/manage-auditing.html  
  - text: couchbase-cli setting-audit
    url: https://docs.couchbase.com/server/current/cli/cbcli/couchbase-cli-setting-audit.html
  - text: Log Redaction
    url: https://docs.couchbase.com/server/current/manage/manage-logging/manage-logging.html
  - text: CBCollect Info
    url: https://docs.couchbase.com/server/current/cli/cbcollect-info-tool.html
prev: /lab4/lab4/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab6/lab6/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Preparación del entorno y variables

Organizarás la carpeta de esta práctica, definirás variables de entorno y verificarás el contenedor/servicio antes de configurar la auditoría.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica5-auditoria/
  mkdir -p practica5-auditoria/scripts
  cd practica5-auditoria
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
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

  # Usuarios para la practica
  RBAC_ANALYST=analyst_ro
  RBAC_ANALYST_PASS='Ana!2025'

  RBAC_WRITER=writer_app
  RBAC_WRITER_PASS='Writ3r!2025'

  RBAC_DBA=dba_scope
  RBAC_DBA_PASS='DbA!2025'

  # Rutas de logs dentro del contenedor
  AUDIT_DIR=/opt/couchbase/var/lib/couchbase/logs/audit
  AUDIT_FILE=audit.log
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

  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 7.** Inicializa el clúster, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM. Habilitar `flush` permite vaciar el bucket desde la UI o CLI.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica5-auditoria**
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

- **Paso 8.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **NOTA:**
  - Contenedor `cbnode1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
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

### Tarea 2: Habilitar auditoría mediante CLI/REST

Activarás la auditoría, configurarás la ruta de logs, el tamaño de rotación y revisarás la configuración vía REST.

#### Tarea 2.1

- **Paso 9.** Primero crea el directorio de **auditoria** dentro del contenedor de couchbase.

  ```bash
  set -a; source .env; set +a
  export MSYS_NO_PATHCONV=1
  export MSYS2_ARG_CONV_EXCL="*"
  for N in cbnode1; do
    docker exec "$N" sh -lc '
      set -e
      mkdir -p /opt/couchbase/var/lib/couchbase/logs/audit
      chown -R couchbase:couchbase /opt/couchbase/var/lib/couchbase/logs/audit
      ls -ld /opt/couchbase/var/lib/couchbase/logs/audit
    '
  done
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 10.** Ejecuta el siguiente comando para la configuracion de la auditoría con `couchbase-cli setting-audit`:

  > **NOTA:** La propiedad `rotate-interval 0` = rotación por tamaño; `rotate-size` ~20MB.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it cbnode1 couchbase-cli setting-audit \
    -c "${CB_HOST}" -u "${CB_USER}" -p "${CB_PASS}" \
    --set \
    --audit-enabled 1 \
    --audit-log-path "${AUDIT_DIR}" \
    --audit-log-rotate-interval 0 \
    --audit-log-rotate-size 20971520
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 11.** Ahora realiza la verificación de la activación de la auditoria.

  ```bash
  docker exec cbnode1 sh -lc 'curl -fsS -u "$CB_USERNAME:$CB_PASSWORD" http://127.0.0.1:8091/settings/audit'
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 12.** Verifica tambien los archivos creados con el siguiente comando.

  ```bash
  docker exec cbnode1 sh -lc "ls -l /opt/couchbase/var/lib/couchbase/logs/audit"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Bucket/scope/collections

Crearás el bucket `orders`, el *scope* `sales` y la *collections* `orders`.

#### Tarea 3.1

- **Paso 13.** Crear el bucket `orders` con **256MB** y *flush* habilitado.

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
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 14.** Crea el *scope* **sales** vía REST, copia y pega el comando en la terminal.

  ```bash
  # Scope
  curl -fsS -u "${CB_USER}:${CB_PASS}" -X POST \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" \
    -d "name=${CB_SCOPE}" || true
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 15.** Crea el *collections* **orders** vía REST, copia y pega el comando en la terminal.

  ```bash
  # Collection: orders
  curl -fsS -u "${CB_USER}:${CB_PASS}" -X POST \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes/${CB_SCOPE}/collections" \
    -d "name=${CB_COLL_ORDERS}" || true
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 16.** Crea el índice primario para **orders** y realizar las pruebas de N1QL.

  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON \`${CB_BUCKET}\`.${CB_SCOPE}.${CB_COLL_ORDERS};"
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 17.** Ejectua el siguiente comando para listar **sales** con **orders** correctamente.

  > **NOTA:** Configuraste el *keyspace* `orders.sales.orders` para probar permisos granulares por *collection*.
  {: .lab-note .info .compact}

  ```bash
  curl -fsS -u "${CB_USER}:${CB_PASS}" \
    "http://localhost:8091/pools/default/buckets/${CB_BUCKET}/scopes" | jq '.'
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 18.** Ahora crea el usuario **analyst_ro** (solo lectura en `orders.sales.orders`), copia y pega el siguiente comando en la terminal.

  ```bash
  docker exec -it cbnode1 couchbase-cli user-manage \
    -c ${CB_HOST} -u "${CB_USER}" -p "${CB_PASS}" \
    --set --rbac-username "${RBAC_ANALYST}" --rbac-password "${RBAC_ANALYST_PASS}" \
    --auth-domain local \
    --roles 'data_reader[orders:sales:orders],query_select[orders:sales:orders]'
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Generar eventos de autenticación (éxitos y fallos)

Provocarás inicios de sesión correctos y fallidos para observar eventos de auditoría de autenticación.

#### Tarea 4.1

- **Paso 19.** Realiza la prueba con el usuario **analyst_ro** para generar logs

  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 -u "${RBAC_ANALYST}" -p ${RBAC_ANALYST_PASS} -s "SELECT 1;"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 14.** Ahora intenta el inicio de sesion para simular una falla.

  ```bash
  docker exec -it cbnode1 cbq -e http://${CB_HOST}:8093 -u "${RBAC_ANALYST}" -p 'Wrong!2025' -s "SELECT 1;" || true
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 15.** Ahora revisa los logs de auditoria generados por los eventos de inicio de sesión.

  > **NOTA:**
  - Debes observar entradas recientes indicando éxito y/o fallo de autenticación con timestamps.  
  - La línea contiene el usuario y el origen **(host/endpoint).**
  - Los eventos de autenticación son clave para **trazabilidad de accesos** y detección de intentos fallidos.
  {: .lab-note .info .compact}

  ```bash
  docker exec cbnode1 sh -lc '
    for d in /opt/couchbase/var/lib/couchbase/logs/audit /opt/couchbase/var/lib/couchbase/logs; do
      f="$d/current-audit.log"
      [ -e "$f" ] && { echo "== $f =="; grep -Ei "auth|Authentication|Authorization|n1ql|statement|query" "$f" | tail -n 50; exit 0; }
    done
    echo "No hay current-audit.log todavía"; exit 1
  '
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Generar eventos RBAC y de administración

Crearás y eliminarás temporalmente un usuario para registrar eventos de cambio RBAC.

#### Tarea 4.1

- **Paso 16.** Crea un usuario temporal `temp_audit` con rol mínimo.

  ```bash
  docker exec -it cbnode1 couchbase-cli user-manage \
    -c ${CB_HOST} -u "${CB_USER}" -p "${CB_PASS}" \
    --set --rbac-username temp_audit --rbac-password 'Temp!2025' \
    --auth-domain local \
    --roles "ro_admin"
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 17.** Ahora lista el usuario para confirmar que se haya creado correctamente.

  ```bash
  docker exec -it cbnode1 couchbase-cli user-manage \
    -c ${CB_HOST} -u "${CB_USER}" -p "${CB_PASS}" --list | grep temp_audit
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 18.** Ahora elimina el usuario `temp_audit` para registrar el evento de borrado:

  ```bash
  docker exec -it cbnode1 couchbase-cli user-manage \
    -c ${CB_HOST} -u "${CB_USER}" -p "${CB_PASS}" \
    --delete --rbac-username temp_audit --auth-domain local
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

- **Paso 19.** Revisar los eventos del uusuario creado y eliminado:

  > **NOTA:** Entradas en `audit.log` con acciones de creación/borrado de usuario.  
  {: .lab-note .info .compact}

  ```bash
  docker exec cbnode1 sh -lc '
  LOG=/opt/couchbase/var/lib/couchbase/logs/audit/current-audit.log
  [ -e "$LOG" ] || LOG=/opt/couchbase/var/lib/couchbase/logs/current-audit.log
  echo "== Eventos de USUARIOS =="
  # Coincide con las llamadas de RBAC por REST y con nombres/descr de eventos
  grep -Ei "/settings/rbac/users/|user-manage|rbac|set user|delete user" "$LOG" | tail -n 50 || true
  '
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Auditoría de operaciones de datos / consultas

Ejecutarás operaciones de lectura/escritura con distintos usuarios para generar eventos de acc
eso a datos y Query (N1QL).

#### Tarea 5.1

- **Paso 20.** Abre la consola web, y verifica que todo este correctamente. Abre la siguiente URL en el navegador **Google Chrome**

  > **IMPORTANTE:** Usa los siguientes datos par autenticarte.
  - **Usuario:** `admin`
  - **Contraseña:** `adminlab`
  {: .lab-note .important .compact}

  ```bash
  http://localhost:8091/
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

- **Paso 21.** En la interfaz da clic en la opción lateral izquierda **Security** expante la propiedad **Query and Index Service** y activala.

  ![cbase24]({{ page.images_base | relative_url }}/24.png)

- **Paso 22.** Da clic en la opción **Save**

- **Paso 23.** Regresa a la terminal de **VSC** 

- **Paso 24.** Crea el usuario **writer_app** (escritura en `orders.sales.orders`, sin permisos de lectura/SELECT):

  ```bash
  docker exec -it cbnode1 couchbase-cli user-manage \
    -c "$CB_HOST" -u "$CB_USER" -p "$CB_PASS" \
    --set --rbac-username "$RBAC_WRITER" --rbac-password "$RBAC_WRITER_PASS" \
    --auth-domain local \
    --roles "data_writer[${CB_BUCKET}:${CB_SCOPE}:${CB_COLL_ORDERS}],query_insert[${CB_BUCKET}:${CB_SCOPE}:${CB_COLL_ORDERS}],query_update[${CB_BUCKET}:${CB_SCOPE}:${CB_COLL_ORDERS}]"
  ```
  ![cbase25]({{ page.images_base | relative_url }}/25.png)

- **Paso 25.** Inserta con el usuario **writer_app** (evento de mutación) los siguientes datos:

  ```bash
  docker exec -it cbnode1 cbq -e http://$CB_HOST:8093 \
    -u "$RBAC_WRITER" -p "$RBAC_WRITER_PASS" \
    -s "INSERT INTO \`$CB_BUCKET\`.$CB_SCOPE.$CB_COLL_ORDERS (KEY, VALUE) VALUES
        ('o::2001', {\"orderId\":2001, \"customerId\":\"c::010\", \"total\":120.5,  \"status\":\"NEW\",  \"ts\":NOW_MILLIS()}),
        ('o::2002', {\"orderId\":2002, \"customerId\":\"c::011\", \"total\":75.0,   \"status\":\"PAID\", \"ts\":NOW_MILLIS()});"
  ```
  ![cbase26]({{ page.images_base | relative_url }}/26.png)

- **Paso 26.** Ahora realiza la consulta con el usuario `analyst_ro` **SELECT**.

  ```bash
  docker exec -it cbnode1 cbq -e http://$CB_HOST:8093 \
    -u "$RBAC_ANALYST" -p "$RBAC_ANALYST_PASS" \
    -s "SELECT orderId,total,status FROM \`orders\`.sales.orders USE KEYS ['o::2001','o::2002'];"
  ```
  ![cbase27]({{ page.images_base | relative_url }}/27.png)

- **Paso 27.** Ejecuta el siguiente comando para revisar la auditoria y verifica las líneas que mencionen `n1ql`, `query` o mutaciones **SELECT** o **INSERT**.

  > **NOTA:** La salida del comando es muy extensa dependiendo de cuantas pruebas hayas realizado, la imagen solo demuestra parte de la salida.
  {: .lab-note .info .compact}

  ```bash
  docker exec cbnode1 sh -lc '
  LOG=/opt/couchbase/var/lib/couchbase/logs/audit/current-audit.log
  [ -e "$LOG" ] || LOG=/opt/couchbase/var/lib/couchbase/logs/current-audit.log
  grep -Ei "n1ql|statement|insert|o::2001|o::2002" "$LOG" | tail -n 20 || true
  '
  ```
  ![cbase28]({{ page.images_base | relative_url }}/28.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Limpieza

Revertirás exclusiones y dejarás la auditoría activada, o la desactivarás según necesidades del entorno de laboratorio.

#### Tarea 6.1

- **Paso 28.** Borrar los usuarios RBAC crados para la practica.

  ```bash
  for U in "${RBAC_ANALYST}" "${RBAC_WRITER}"; do
    docker exec -it cbnode1 couchbase-cli user-manage \
      -c ${CB_HOST} -u "${CB_USER}" -p "${CB_PASS}" \
      --delete --rbac-username "$U" --auth-domain local || true
  done
  ```
  ![cbase29]({{ page.images_base | relative_url }}/29.png)

 **Paso 29.** Ahora elimina el bucket con el siguiente comando.

  > **NOTA:** Espera unos segundos, es normal que tarde en eliminarse.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Si te llegase a marcar algun error en el borrado, pude deberse a que aun esta ocupado en alguna tarea, intentalo nuevamente. En ocasiones se borra aunque mande el mensaje de error.
  {: .lab-note .important .compact}

  ```bash
  docker exec -it cbnode1 couchbase-cli bucket-delete \
    -c 127.0.0.1 -u "$CB_USER" -p "$CB_PASS" \
    --bucket orders
  ```

- **Paso 30.** Ahora en la terminal aplica el siguiente comando para detener el nodo.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase30]({{ page.images_base | relative_url }}/30.png)

- **Paso 31.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}

  ```bash
  docker compose down
  ```
  ![cbase31]({{ page.images_base | relative_url }}/31.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}