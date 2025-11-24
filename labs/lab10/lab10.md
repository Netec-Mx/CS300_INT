---
layout: lab
title: "Práctica 10: Rack Awareness" # CAMBIAR POR CADA PRACTICA
permalink: /lab10/lab10/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab10/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar un clúster multinodo de Couchbase en Docker, **configurar Server Groups (Rack/Zone Awareness)** para aislar réplicas por “rack”, **migrar nodos a grupos**, **rebalancear** y **validar alta disponibilidad** simulando la caída de un rack.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos locales disponibles `18091/28091/38091` (UI/REST), `11210` (solo para depuración si se requiere mapear KV).  
  - Conectividad a Internet para descargar imágenes.  
  - 4-6 GB de RAM libres (3 nodos con servicios `kv,index,query`).
introduction: | # CAMBIAR POR CADA PRACTICA
  **Rack Awareness** en Couchbase se implementa mediante **Server Groups**, lo que permite que **réplicas** y **particiones** (vBuckets) se distribuyan entre grupos lógicos (racks/zonas). Así, la falla total de un “rack” no causa pérdida de datos ni indisponibilidad, siempre que existan suficientes réplicas distribuidas en otros grupos. En esta práctica crearás tres nodos, formarás un clúster, definirás de 2 a 3 grupos (p. ej., `RackA`, `RackB`, `RackC`), moverás nodos y **rebalancearás** para garantizar la colocación cruzada de réplicas.
slug: lab10 # CAMBIAR POR CADA PRACTICA
lab_number: 10 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Levantaste un clúster de 3 nodos, creaste **Server Groups (Rack Awareness)**, distribuíste nodos entre racks y realizaste un **rebalance**. Luego configuraste un **bucket con réplicas=2**, insertaste y consultaste datos, simulaste la **caída de un rack** y comprobaste la **continuidad del servicio**, consolidando prácticas de alta disponibilidad con Couchbase.
notes: | # CAMBIAR POR CADA PRACTICA
  - **Réplicas**: configura `--replica` acorde al número de racks (p. ej., dos réplicas en tres nodos/racks).  
  - **Rebalance**: siempre que cambies grupos o agregues y elimines nodos.  
  - **Durabilidad**: en producción usa **`DurabilityLevel`** para confirmar la persistencia o replicación de escrituras.  
  - **Auto-Failover**: ajusta timeouts de `Cluster Settings` según tu SLO.  
  - **Topologías**: en `clouds`, mapea `Server Groups` a **`zones/AZ`** para resiliencia regional.  
  - **Seguridad**: considera TLS (ver Práctica 6) y RBAC mínima necesaria.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Server Groups (Rack/Zone Awareness)
    url: https://docs.couchbase.com/server/current/manage/manage-groups/manage-groups.html  
  - text: Rebalance y operaciones de clúster
    url: https://docs.couchbase.com/server/current/learn/clusters-and-availability/rebalance.html  
  - text: Puertos y redes de Couchbase
    url: https://docs.couchbase.com/server/current/install/install-ports.html
    
prev: /lab9/lab9/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab11/lab11/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1. Estructura base del proyecto

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica10-rack/
  mkdir -p practica10-rack/{node1,node2,node3}/{data,logs,config}
  mkdir -p practica10-rack/scripts
  cd practica10-rack
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** Ahora, en el árbol de directorios lateral derecho verifica los directorios `node1`, `node2`, `node3` existen con `data/`, `logs/`, `config/` dentro y el directorio `scripts`.

  > **Nota.** Mantener `datos/logs` por nodo permite borrar o recrear nodos de forma independiente sin perder claridad.
  {: .lab-note .info .compact}

  ![cbase3]({{ page.images_base | relative_url }}/3.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Variables de entorno y Docker Compose (3 nodos)

Definirás variables en `.env` y un `compose.yaml` con 3 servicios. Solo el **nodo 1** expondrá puertos hacia el host. Cada servicio tendrá healthcheck.


- **Paso 1.** Crea el archivo `.env` dentro del directorio **`practica10-rack`**.

  > **Importante.** El comando se ejecuta dentro del directorio **`practica10-rack`**. 
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

- **Paso 2.** Ahora, con mucho cuidado, ejecuta este comando que creará el archivo `compose.yaml` donde se definen los tres nodos de Couchbase.

  > **Notas**
  - Una red bridge compartida permite que los nodos se resuelvan por `hostname` (`couchbase1`, `couchbase2`, `couchbase3`).  
  - Publicar puertos solo en el nodo 1 simplifica el acceso desde el `host`.
  - El stack de 3 servicios está listo para levantarse.
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


- **Paso 1.** Ahora, ejecuta el comando para levantar los 3 nodos mediante el docker `compose` y cargar las variables.

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

- **Paso 3.** Ahora, verifica el status particularmente de cada nodo.
  
  > **Nota.**
  - Los 3 contenedores deben estar `Up`. `Health` puede tardar ~1–2 min en quedar `healthy` la primera vez.
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


- **Paso 1.** De manera más eficiente, crea un script para la inicializcion de los nodos. Copia y pega el siguiente comando.

  > **Nota.** El comando se ejecuta dentro de la carpeta **`practica10-rack`**.
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
    warn "Clúster ya inicializado o parámetros ya aplicados; continuamos…"
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

- **Paso 2.** Ahora, ejecuta el script para levantar los nodos.

  > **Notas**
  - `server-list` debe reportar **3 nodos** con servicios `data,index,query`.  
  - `bucket-list` debe mostrar el bucket `test`.  
  - `rebalance` no debe devolver errores.
  {: .lab-note .info .compact}

  ```bash
  ./scripts/cluster-init.sh
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Crear Server Groups (Rack Awareness) y mover los nodos

Crearás grupos lógicos (racks) y moverás nodos entre ellos. Usarás **`server-group-manage`** para crear, listar y mover.

- **Paso 1.** Lista los grupos existentes con el siguiente comando.

  > **Nota.** Por defecto, existe un **`Group 1`**.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 couchbase-cli group-manage \
    -c couchbase1 -u "${CB_USER}" -p "${CB_PASS}" --list
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 2.** Crea el primer grupo llamado **`RackA`**.

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 couchbase-cli group-manage \
    -c couchbase1 -u "${CB_USER}" -p "${CB_PASS}" \
    --create --group-name "RackA"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 3.** Crea el segundo grupo llamado **RackB**.

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 couchbase-cli group-manage \
    -c couchbase1 -u "${CB_USER}" -p "${CB_PASS}" \
    --create --group-name "RackB"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 4.** Mueve el nodo **`couchbase2`** al grupo **`RackA`**.

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 couchbase-cli group-manage \
    -c couchbase1.cb.local:8091 -u "${CB_USER}" -p "${CB_PASS}" \
    --move-servers "couchbase2.cb.local:8091" \
    --from-group "Group 1" \
    --to-group "RackA"
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 5.** Ahora, mueve el nodo **`couchbase3`** al grupo **`RackB`**.

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 couchbase-cli group-manage \
    -c couchbase1 -u "${CB_USER}" -p "${CB_PASS}" \
    --move-servers "couchbase3.cb.local:8091" \
    --from-group "Group 1" \
    --to-group "RackB"
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 6.** Confirma que los nodos estén correctamente en los grupos.

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 couchbase-cli group-manage \
    -c couchbase1 -u "${CB_USER}" -p "${CB_PASS}" --list
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 7.** Ahora, ejecuta el **`rebalance`** para aplicar cambios de colocación. 

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 couchbase-cli rebalance \
    -c couchbase1 -u "${CB_USER}" -p "${CB_PASS}"
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Crear bucket con réplicas y validar distribución

Crearás un bucket con réplicas para permitir disponibilidad ante la caída de un rack. Luego probarás inserciones y lectura.

- **Paso 1.** Ajusta el bucket creado con **`réplicas=2`**.

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 couchbase-cli bucket-edit \
    -c couchbase1 -u "${CB_USER}" -p "${CB_PASS}" \
    --bucket "${CB_BUCKET}" \
    --bucket-replica 2 \
    --enable-flush 1
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 2.** Aplica `Rebalance` para el nuevo factor de réplica.

  > **Nota.** Puede tardar 1 minuto en promedio.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 couchbase-cli rebalance \
    -c couchbase1 -u "${CB_USER}" -p "${CB_PASS}"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 3.** Crea el índice primario y, luego, inserta el documento para después consultarlo. Ejecuta los dos comandos uno por uno.

  > **Nota.** La salida es extensa, la imagen representa una parte de la salida.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 cbq \
    -e http://couchbase1:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s "CREATE PRIMARY INDEX IF NOT EXISTS ON \`${CB_BUCKET}\`;"
  ```
  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 cbq \
    -e http://couchbase1:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s "INSERT INTO \`${CB_BUCKET}\` (KEY,VALUE) VALUES ('doc::1', {\"type\":\"demo\",\"rack\":\"A\"});" \
    -s "UPSERT INTO \`${CB_BUCKET}\` (KEY,VALUE) VALUES ('doc::2', {\"type\":\"demo\",\"rack\":\"B\"});" \
    -s "SELECT META().id AS id, rack FROM \`${CB_BUCKET}\` WHERE type='demo' ORDER BY META().id;"
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 4.** Con el siguiente comando, verifica la distribución.

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 couchbase-cli bucket-list \
    -c couchbase1 -u "${CB_USER}" -p "${CB_PASS}"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Simular la caída de un rack y validar HA

Detendrás uno de los nodos de un rack y validarás que los datos siguen accesibles (failover/rebalance, según sea necesario).

- **Paso 1.** Primero, detendrás temporalmente el **`n3`** para simular la caída del rack.

  {%raw%}
  ```bash
  docker stop ${CB_CONTAINER_PREFIX}3
  sleep 5
  docker ps --format "table {{.Names}}\t{{.Status}}"
  ```
  {%endraw%}
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 2.** Ahora, realiza la lectura de datos desde `n1` (consulta N1QL).

  > **Nota.** Si el comando se queda pegado, espera unos minutos a que termine. Probablemente, dará error por que no encuentra como rutear la solicitud. Vuelve a ejecutar el comando, el nodo debería responder .
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${CB_CONTAINER_PREFIX}1 cbq \
    -e http://couchbase1:8093 -u "${CB_USER}" -p "${CB_PASS}" \
    -s "SELECT COUNT(*) AS n FROM \`${CB_BUCKET}\`;" \
    -s "SELECT META().id AS id, rack FROM \`${CB_BUCKET}\` WHERE type='demo' ORDER BY META().id;"
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 3.** Ahora, recupera el `rack/nodo` caído.

  ```bash
  docker start ${CB_CONTAINER_PREFIX}3
  sleep 5
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 8. Limpieza

Borrar datos en el entorno para repetir pruebas.

- **Paso 1.** En la terminal, aplica el siguiente comando para detener el nodo.

  > **Nota.** Si es necesario, puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  

- **Paso 2.** Apaga y elimina el contenedor (se conservan los datos en `./data`).

  > **Nota.** Si es necesario, puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **Importante.** Es normal el mensaje del objeto de red **`No resource found to remove`**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[7] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
