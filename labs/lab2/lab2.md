---
layout: lab
title: "Práctica 2: Clúster multinodo en Docker Compose" # CAMBIAR POR CADA PRACTICA
permalink: /lab2/lab2/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab2/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar un clúster de Couchbase de 3 nodos con Docker Compose en una sola máquina, inicializar el clúster desde el nodo 1, unir nodos 2 y 3, rebalancear, crear un bucket de prueba y validar estado/servicios vía CLI, REST y la consola web, usando Visual Studio Code y terminal Git Bash.
prerequisites: # CAMBIAR POR CADA PRACTICA
  - Visual Studio Code con terminal **Git Bash**.
  - 6–8 GB de RAM libres recomendados para 3 nodos (mínimo ~5 GB).  
  - Puertos locales disponibles **8091–8096**, **11210** (se publicarán solo desde el nodo 1).
  - Conectividad a Internet para descargar la imagen.
  - Opcional `jq` para mejorar salidas JSON.
introduction: # CAMBIAR POR CADA PRACTICA
  - Configurarás un clúster Couchbase de 3 nodos (couchbase1, couchbase2, couchbase3) sobre una red de Docker. Publicaremos puertos externos solo en el nodo 1 para usar la Web Console local y APIs; los demás nodos se comunicarán por la red interna de Docker. Automatizarás la creación y el rebalanceo con `couchbase-cli` y verificarás la salud del clúster.
slug: lab2 # CAMBIAR POR CADA PRACTICA
lab_number: 2 # CAMBIAR POR CADA PRACTICA
final_result: > # CAMBIAR POR CADA PRACTICA
  Clúster de Couchbase de **3 nodos** funcionando en Docker Compose, inicializado desde el nodo 1, con nodos 2 y 3 añadidos y rebalanceados, bucket de prueba creado, y validaciones completas por CLI, REST y Web Console.
notes: # CAMBIAR POR CADA PRACTICA
  - Ajusta `CB_RAM`, `CB_INDEX_RAM` y `CB_BUCKET_RAM` según la RAM disponible.  
  - Evita publicar puertos de nodos 2 y 3 para no duplicar puertos en el host.  
  - Si usas Windows con WSL2, mantén el proyecto en el sistema de archivos de Linux (`~`) para mejor rendimiento.  
  - Si el rebalance tarda o falla, revisa RAM y logs en `nodeX/logs/`.  
  - Para Community Edition usa `couchbase/server:community`. Para Enterprise (trial) `couchbase/server:enterprise`.
  - En redes corporativas con proxy, asegúrate que Docker tenga acceso de salida para descargar la imagen.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Instalación Couchbase en Docker
    url: https://docs.couchbase.com/server/current/install/getting-started-docker.html
  - text: REST API de administración
    url: https://docs.couchbase.com/server/current/rest-api/rest-intro.html  
  - text: Puertos de Couchbase
    url: https://docs.couchbase.com/server/current/install/install-ports.html
prev: /lab1/lab1 # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab3/lab3/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---
### Tarea 1. Estructura base del proyecto

Crearás la carpeta de la práctica con subdirectorios por nodo para separar datos, logs y config, y abrirás el proyecto en VS Code.


- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**.

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica2-multinode/{node1,node2,node3}/{data,logs,config}
  mkdir practica2-multinode/scripts
  cd practica2-multinode
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** Ahora en el árbol de directorios lateral derecho verifica los directorios `node1`, `node2`, `node3` existen con `data/`, `logs/`, `config/` dentro y el directorio `scripts`.

  > **Nota.** Mantener datos/losgs por nodo permite borrar o recrear nodos de forma independiente sin perder claridad.
  {: .lab-note .info .compact}

  ![cbase3]({{ page.images_base | relative_url }}/3.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Variables de entorno y Docker Compose (3 nodos)

Definirás variables en `.env` y un `compose.yaml` con 3 servicios. Solo el **nodo 1** expondrá puertos hacia el host. Cada servicio tendrá healthcheck.


- **Paso 1.** Crea el archivo `.env` dentro del directorio **practica2-multinode**.

  > **Importante.** El comando se ejecuta dentro del directorio **practica2-multinode** 
  {: .lab-note .important .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_USER=admin
  CB_PASS='adminlab'
  CB_SERVICES=data,index,query
  CB_RAM=2048
  CB_INDEX_RAM=512
  CB_BUCKET=dev
  CB_BUCKET_RAM=256
  CB_NETWORK=cbnet
  CB_CONTAINER_PREFIX=cbnode
  EOF
  ```

- **Paso 2.** Ahora con mucho cuidado ejecuta este comando que creara el archivo `compose.yaml` donde se definen los 3 nodos de couchbase.:.

  > **Nota.**
  - Una red bridge compartida permite que los nodos se resuelvan por `hostname` (`couchbase1`, `couchbase2`, `couchbase3`).  
  - Publicar puertos solo en el nodo 1 simplifica el acceso desde el host.
  - Stack de 3 servicios listo para ser levantado.
  {: .lab-note .info .compact}

  ```bash
  cat > compose.yaml << 'YAML'
  name: couchbase-multinode
  services:
    couchbase1:
      image: couchbase/server:${CB_TAG}
      container_name: ${CB_CONTAINER_PREFIX}1
      hostname: couchbase1
      domainname: cb.local
      ports:
        - "8091-8096:8091-8096"   # Exponer UI/servicios solo desde el nodo 1
        - "11210:11210"
      environment:
        - CB_USERNAME=${CB_USER}
        - CB_PASSWORD=${CB_PASS}
      volumes:
        - ./node1/data:/opt/couchbase/var
        - ./node1/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./node1/config:/config
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 12
      restart: unless-stopped

    couchbase2:
      image: couchbase/server:${CB_TAG}
      container_name: ${CB_CONTAINER_PREFIX}2
      hostname: couchbase2
      domainname: cb.local
      environment:
        - CB_USERNAME=${CB_USER}
        - CB_PASSWORD=${CB_PASS}
      volumes:
        - ./node2/data:/opt/couchbase/var
        - ./node2/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./node2/config:/config
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 12
      restart: unless-stopped

    couchbase3:
      image: couchbase/server:${CB_TAG}
      container_name: ${CB_CONTAINER_PREFIX}3
      hostname: couchbase3
      domainname: cb.local
      environment:
        - CB_USERNAME=${CB_USER}
        - CB_PASSWORD=${CB_PASS}
      volumes:
        - ./node3/data:/opt/couchbase/var
        - ./node3/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./node3/config:/config
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 12
      restart: unless-stopped

  networks:
    default:
      name: ${CB_NETWORK}
      driver: bridge
  YAML
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

<!-- ESTRUCTURA DE UNA TAREA COPIA Y PEGAR, MODIFICAR CUANTAS TAREAS SEAN NECESARIAS -->
### Tarea 3. Arrancar contenedores y verificar health

Subirás los 3 contenedores y confirmarás que los healthchecks respondan.


- **Paso 1.** Ahora ejecuta el comando para levantar los 3 nodos mediante el docker compose y cargar las variables.

  ```bash
  set -a; source .env; set +a
  docker compose up -d
  ```
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

- **Paso 2.** Espera unos minutos en lo que se levantan los nodos y ejecuta el siguiente comando.

  {%raw%}
  ```bash
  docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
  ```
  {%endraw%}
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

- **Paso 3.** Ahora verifica el status particularmente de cada nodo.
  
  > **Nota.**
  - Los 3 contenedores deben estar `Up`. Health puede tardar ~1–2 min en quedar `healthy` la primera vez.
  - El endpoint `/pools` es una sonda rápida para saber que el proceso de Couchbase está arriba (aunque no inicializado).
  {: .lab-note .info .compact}

  {%raw%}
  ```bash
  docker inspect -f '{{.Name}} -> {{.State.Health.Status}}' ${CB_CONTAINER_PREFIX}1
  docker inspect -f '{{.Name}} -> {{.State.Health.Status}}' ${CB_CONTAINER_PREFIX}2
  docker inspect -f '{{.Name}} -> {{.State.Health.Status}}' ${CB_CONTAINER_PREFIX}3
  ```
  {%endraw%}
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Inicializar clúster, unir nodos y rebalancear

Inicializarás el clúster en el nodo 1, agregarás nodos 2 y 3 con los servicios definidos y lanzarás un rebalanceo.


- **Paso 1.** De manera más eficiente crearas un script para la inicialización de los nodos, copia y pega el siguiente comando.

  > **Nota.** El comando se ejecuta dentro de la carpeta **practica2-multinode**
  {: .lab-note .info .compact}

  {%raw%}
  ```bash
  cat > scripts/cluster-init.sh << 'EOF'
  #!/usr/bin/env bash
  set -euo pipefail

  # ----------------- helpers -----------------
  blue='\033[1;34m'; green='\033[1;32m'; yellow='\033[1;33m'; red='\033[1;31m'; nc='\033[0m'
  info(){ echo -e "${blue}[INFO]${nc} $*"; }
  ok(){   echo -e "${green}[ OK ]${nc} $*"; }
  warn(){ echo -e "${yellow}[WARN]${nc} $*"; }
  err(){  echo -e "${red}[ERR ]${nc} $*" >&2; }

  # ----------------- env -----------------
  [[ -f .env ]] || { err "No existe .env"; exit 1; }
  sed -i 's/\r$//' .env || true
  set -a; source .env; set +a

  : "${CB_CONTAINER_PREFIX:?falta CB_CONTAINER_PREFIX}"
  : "${CB_USER:?falta CB_USER}"
  : "${CB_PASS:?falta CB_PASS}"
  : "${CB_SERVICES:?falta CB_SERVICES}"
  : "${CB_RAM:?falta CB_RAM}"
  : "${CB_INDEX_RAM:?falta CB_INDEX_RAM}"
  : "${CB_BUCKET:?falta CB_BUCKET}"
  : "${CB_BUCKET_RAM:?falta CB_BUCKET_RAM}"

  # Normaliza nombres de servicios a los del CLI
  norm_services(){ echo "$1" | sed -E 's/\bkv\b/data/g; s/\bn1ql\b/query/g; s/\bcbas\b/analytics/g'; }

  SERVICES_NODE1=$(norm_services "${SERVICES_NODE1:-data,index,query}")
  SERVICES_OTHERS=$(norm_services "${SERVICES_OTHERS:-data,query}")   # por defecto index solo en nodo1 (más estable)
  info "Servicios nodo1:   $SERVICES_NODE1"
  info "Servicios otros:   $SERVICES_OTHERS"

  LEADER="${CB_CONTAINER_PREFIX}1"
  C2="${CB_CONTAINER_PREFIX}2"
  C3="${CB_CONTAINER_PREFIX}3"
  DOMAIN="${CB_DOMAIN:-cb.local}"

  # ----------------- prechecks -----------------
  docker ps --format '{{.Names}}' | grep -Eq "^(${LEADER}|${C2}|${C3})$" \
    || { err "Faltan contenedores ${LEADER}/${C2}/${C3} en ejecución"; exit 1; }

  IP1=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "$LEADER")
  IP2=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "$C2")
  IP3=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "$C3")
  info "IPs internas: $LEADER=$IP1  $C2=$IP2  $C3=$IP3"

  FQDN1="couchbase1.${DOMAIN}"
  FQDN2="couchbase2.${DOMAIN}"
  FQDN3="couchbase3.${DOMAIN}"

  # ----------------- utilidades -----------------
  wait_http(){
    local cont="$1" host="$2" tries="${3:-60}" delay="${4:-2}" code=000
    info "Esperando API http://$host:8091/pools en $cont ..."
    for _ in $(seq 1 "$tries"); do
      code=$(docker exec "$cont" sh -lc "curl -s -o /dev/null -w '%{http_code}' http://$host:8091/pools" || true)
      [[ "$code" == "200" ]] || code=$(docker exec "$cont" sh -lc "curl -s -o /dev/null -w '%{http_code}' -u $CB_USER:$CB_PASS http://$host:8091/pools" || true)
      [[ "$code" == "200" || "$code" == "401" ]] && { ok "API (HTTP $code) en $host:8091"; return 0; }
      sleep "$delay"
    done
    err "Timeout esperando $host (último HTTP $code)"; return 1
  }

  append_host_if_missing(){
    # $1=container  $2=IP  $3=FQDN
    docker exec "$1" sh -lc "grep -qE '^[[:space:]]*$2[[:space:]]+$3(\$|[[:space:]])' /etc/hosts || echo '$2 $3' | tee -a /etc/hosts >/dev/null"
  }

  set_hosts_all(){
    info "Actualizando /etc/hosts (append idempotente) en los 3 nodos…"
    for N in "$LEADER" "$C2" "$C3"; do
      append_host_if_missing "$N" "$IP1" "$FQDN1"
      append_host_if_missing "$N" "$IP2" "$FQDN2"
      append_host_if_missing "$N" "$IP3" "$FQDN3"
    done
    ok "/etc/hosts actualizado"
  }

  apply_node_init(){
    # $1=container  $2=fqdn
    local cont="$1" fqdn="$2"
    info "node-init en $cont (hostname=$fqdn)…"
    # 1) sin auth
    if docker exec "$cont" couchbase-cli node-init -c 127.0.0.1 --node-init-hostname "$fqdn" >/dev/null 2>&1; then
      ok "node-init aplicado en $cont (sin auth)"
      return 0
    fi
    # 2) con auth (por si ya está inicializado)
    if docker exec "$cont" couchbase-cli node-init -c 127.0.0.1 -u "$CB_USER" -p "$CB_PASS" --node-init-hostname "$fqdn" >/dev/null 2>&1; then
      ok "node-init aplicado en $cont (con auth)"
      return 0
    fi
    # 3) reset y reintento
    warn "node-init falló en $cont; aplicando node-reset y reintentando…"
    docker exec "$cont" couchbase-cli node-reset -c 127.0.0.1 >/dev/null 2>&1 || true
    docker exec "$cont" couchbase-cli node-init -c 127.0.0.1 --node-init-hostname "$fqdn"
    ok "node-init aplicado en $cont tras reset"
  }

  check_indexer_connectivity(){
    info "Chequeando conectividad indexer (9102) entre nodos…"
    for c in "$LEADER" "$C2" "$C3"; do
      docker exec "$c" sh -lc "for h in $FQDN1 $FQDN2 $FQDN3; do printf '%s -> %s: ' '$c' '\$h'; curl -s -o /dev/null -w '%{http_code}\n' http://\$h:9102 || true; done"
    done
    info "200/401 = OK; 000 = problema de red/DNS."
  }

  # ----------------- 1) Espera API local en cada contenedor -----------------
  wait_http "$LEADER" "127.0.0.1"
  wait_http "$C2"     "127.0.0.1"
  wait_http "$C3"     "127.0.0.1"

  # ----------------- 2) FQDN + /etc/hosts en TODOS -----------------
  apply_node_init "$LEADER" "$FQDN1"
  apply_node_init "$C2"     "$FQDN2"
  apply_node_init "$C3"     "$FQDN3"

  set_hosts_all
  check_indexer_connectivity

  # ----------------- 3) cluster-init en líder (por FQDN) -----------------
  info "cluster-init en $FQDN1 …"
  if docker exec "$LEADER" couchbase-cli cluster-init -c "$FQDN1" \
    --cluster-username "$CB_USER" \
    --cluster-password "$CB_PASS" \
    --services "$SERVICES_NODE1" \
    --cluster-ramsize "$CB_RAM" \
    --cluster-index-ramsize "$CB_INDEX_RAM" \
    --index-storage-setting default; then
    ok "Cluster initialized en $FQDN1"
  else
    warn "Cluster ya inicializado o parámetros ya aplicados; continuamos…"
  fi

  # ----------------- 4) Agregar nodos por FQDN -----------------
  add_node(){
    local fqdn="$1" services="$2" cont="$3"
    info "server-add $fqdn (services=$services)…"
    if docker exec "$LEADER" couchbase-cli server-add -c "$FQDN1" \
        -u "$CB_USER" -p "$CB_PASS" \
        --server-add "$fqdn" \
        --server-add-username "$CB_USER" \
        --server-add-password "$CB_PASS" \
        --services "$services"; then
      ok "Nodo $fqdn agregado"
    else
      warn "Fallo agregando $fqdn. node-reset en $cont y reintento…"
      docker exec "$cont" couchbase-cli node-reset -c 127.0.0.1 >/dev/null 2>&1 || true
      docker exec "$LEADER" couchbase-cli server-add -c "$FQDN1" \
        -u "$CB_USER" -p "$CB_PASS" \
        --server-add "$fqdn" \
        --server-add-username "$CB_USER" \
        --server-add-password "$CB_PASS" \
        --services "$services"
      ok "Nodo $fqdn agregado tras reset"
    fi
  }
  add_node "$FQDN2" "$SERVICES_OTHERS" "$C2"
  add_node "$FQDN3" "$SERVICES_OTHERS" "$C3"

  # ----------------- 5) Rebalance -----------------
  info "Iniciando rebalance…"
  docker exec "$LEADER" couchbase-cli rebalance -c "$FQDN1" -u "$CB_USER" -p "$CB_PASS"
  ok "Rebalance lanzado"

  # ----------------- 6) Verificaciones -----------------
  info "Estado de rebalance:"
  docker exec "$LEADER" couchbase-cli rebalance-status -c "$FQDN1" -u "$CB_USER" -p "$CB_PASS" || true

  info "Server list:"
  docker exec "$LEADER" couchbase-cli server-list -c "$FQDN1" -u "$CB_USER" -p "$CB_PASS" || true

  info "Buckets:"
  if docker exec "$LEADER" couchbase-cli bucket-list -c "$FQDN1" -u "$CB_USER" -p "$CB_PASS" | grep -qx "$CB_BUCKET"; then
    warn "Bucket $CB_BUCKET ya existe"
  else
    docker exec "$LEADER" couchbase-cli bucket-create -c "$FQDN1" -u "$CB_USER" -p "$CB_PASS" \
      --bucket "$CB_BUCKET" --bucket-type couchbase --bucket-ramsize "$CB_BUCKET_RAM" --enable-flush 1
    ok "Bucket $CB_BUCKET creado"
  fi

  check_indexer_connectivity
  ok "Script finalizado."
  EOF
  chmod +x scripts/cluster-init.sh
  ```
  {%endraw%}

- **Paso 2.** Ahora ejecuta el script para levantar los nodos.

  > **Nota.**
  - `server-list` debe reportar **3 nodos** con servicios `data,index,query`.  
  - `bucket-list` debe mostrar el bucket `test`.  
  - `rebalance` no debe devolver errores.
  {: .lab-note .info .compact}

  ```bash
  ./scripts/cluster-init.sh
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 3.** Avanza con la siguiente tarea para realizar las pruebas en lo que termina de ejecutarse el **Rebalance**.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Validaciones por REST y Web Console

Consultarás endpoints REST y validarás desde la consola web que existen 3 nodos, servicios activos y el bucket creado.


- **Paso 1.** Valida los nodos mediante el endpoint REST.

  > **Importante.** Recuerda que la salida de la inforamción es demasiada extensa, puedes analizarla o interrumpirla con **`CTRL + C`**
  {: .lab-note .important .compact}

  ```bash
  set -a; source .env; set +a
  curl -fsS -u "${CB_USER}:${CB_PASS}" http://localhost:8091/pools/default | jq '.'
  ```

- **Paso 2.** Abre la consola web, y verifica que todo este correctamente. Abre la siguiente URL en el navegador **Google Chrome**.

  > **Importante.** Usa los siguientes datos par autenticarte.
  - **Usuario:** `admin`
  - **Contraseña:** `adminlab`
  {: .lab-note .important .compact}

  ```bash
  http://localhost:8091/
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 3.** Entrando al sistema de Couchbase te aparecera la notificación del rebalance que se ha aplicado correctamente.

  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 4.** Da clic en la opción de **Buckets** y verás que aplicó la distribución a los 3 nodos.

  ![cbase11]({{ page.images_base | relative_url }}/11.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Validaciones por REST y Web Console

Crearás un índice primario para ejecutar una consulta rápida sobre el bucket `test`.


- **Paso 1.** Ejecuta el siguiente código para crear un índice primario.

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 cbq -e http://couchbase1:8093 -u "${CB_USER}" -p "${CB_PASS}" -s "CREATE PRIMARY INDEX ON \`${CB_BUCKET}\`;"
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 2.** Ahora realiza la prueba para consultar el índice primario.

  > **Importante.** Para consultas ad-hoc el índice primario es útil. En producción se recomiendan índices específicos.
  {: .lab-note .important .compact}

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 cbq -e http://couchbase1:8093 -u "${CB_USER}" -p "${CB_PASS}" -s "SELECT META().id, * FROM \`${CB_BUCKET}\` LIMIT 5;"
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Limpieza y reinicio (buenas prácticas)

Aprenderás a apagar/encender el servicio y a limpiar volúmenes si necesitas empezar desde cero.


- **Paso 1.** Ahora regresa a la terminal de **Git Bash** para aplicar el siguiente comando.

  > **Nota.** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 2.** Apagar y eliminar contenedor (se conservan los datos en ./data).

  > **Nota.** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}

  ```bash
  docker compose down
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
