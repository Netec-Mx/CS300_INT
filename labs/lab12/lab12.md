---
layout: lab
title: "Práctica 12: Rebalance tras cambio de nodos" # CAMBIAR POR CADA práctica
permalink: /lab12/lab12/ # CAMBIAR POR CADA práctica
images_base: /labs/lab12/img # CAMBIAR POR CADA práctica
duration: "25 minutos" # CAMBIAR POR CADA práctica
objective: # CAMBIAR POR CADA práctica
  - Desplegar un clúster de Couchbase para **rebalance** después de **cambios de topología** *scale out* (agregar nodos) y *scale in* (retirar nodos). Validarás continuidad del servicio, redistribución de particiones (vBuckets) e integridad de datos
prerequisites:  # CAMBIAR POR CADA práctica
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 4–6 GB de RAM libres (3 nodos con servicios `kv,index,query`).
introduction: | # CAMBIAR POR CADA práctica
  En Couchbase, el **rebalance** redistribuye particiones (vBuckets), réplicas e índices cuando cambias la **topología** del clúster (agregar/quitar nodos, mover grupos, etc.). Un rebalance correcto mantiene disponibilidad y rendimiento al equilibrar carga y almacenamiento. En está práctica crearás un clúster de 3 nodos, cargarás datos, harás **scale out** a 4 nodos y después **scale in** de vuelta a 3, verificando que los datos siguen accesibles y que la distribución se estabiliza.
slug: lab12 # CAMBIAR POR CADA práctica
lab_number: 12 # CAMBIAR POR CADA práctica
final_result: | # CAMBIAR POR CADA práctica
  Creaste un clúster de 3 nodos, cargaste datos, realizaste **scale out** a 4 nodos y luego **scale in** a 3, ejecutando **rebalance** en cada cambio y verificando continuidad e integridad. Ahora dominas el ciclo típico de **cambio de capacidad** en Couchbase y su efecto en la distribución de vBuckets e índices.
notes: | # CAMBIAR POR CADA práctica
  - **Index Service**: si usas índices secundarios, considera *index rebalancing* y la opción de *moving indexes* (tiempos mayores).  
  - **Server Groups**: en entornos productivos, combina cambios de nodos con **Rack/Zone Awareness** (ver Práctica 10).  
  - **Durabilidad**: para escrituras críticas durante rebalances, considera `DurabilityLevel`.  
  - **Monitoreo**: observa *Tasks* en la UI y métricas de *Rebalance Progress*.  
  - **Versiones**: flags de CLI pueden variar levemente entre versiones de Couchbase Server.
references: # CAMBIAR POR CADA práctica LINKS ADICIONALES DE DOCUMENTACION
  - text: Rebalance (conceptos y flujos)
    url: https://docs.couchbase.com/server/current/learn/clusters-and-availability/rebalance.html 
  - text: Puertos
    url: https://docs.couchbase.com/server/current/install/install-ports.html
    
prev: /lab11/lab11/ # CAMBIAR POR CADA práctica MENU DE NAVEGACION HACIA ATRAS        
next: /lab13/lab13/ # CAMBIAR POR CADA práctica MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Estructura base y variables

Crearás una carpeta aislada para la práctica y un `.env` con variables para 4 nodos potenciales.



- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**.

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica12-rebalance/
  mkdir -p practica12-rebalance/couchbase/{data1,data2,data3,data4,logs,config}
  mkdir -p practica12-rebalance/couchbase/scripts
  cd practica12-rebalance
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **Nota.** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  # Nodos
  NODE1=cb-rb-n1
  NODE2=cb-rb-n2
  NODE3=cb-rb-n3
  NODE4=cb-rb-n4   # se usará para scale out

  # Admin
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'

  # Servicios y RAM del clúster
  CB_SERVICES=data,index,query
  CB_RAM=2048
  CB_INDEX_RAM=512
  CB_NETWORK=cbnet

  # Bucket para probar HA/Failover
  CB_BUCKET=rbucket
  CB_BUCKET_RAM=512
  CB_REPLICAS=1   # con 3-4 nodos basta para resiliencia básica
  EOF
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Estructura base y variables

Definirás un `compose.yaml` con 3 nodos iniciales y un 4º nodo listo para *scale out* (se puede levantar después).



- **Paso 1.** Ahora crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente código en la terminal.

  > **Nota.**
  - El archivo `compose.yaml` mapea puertos 8091–8096 para la consola web y 11210 para clientes.
  - El healthcheck consulta el endpoint `/pools` que responde cuando el servicio está arriba (aunque aún no inicializado).
  {: .lab-note .info .compact}

  ```bash
  cat > compose.yaml << 'YAML'
  name: couchbase-rebalance-lab
  services:
    n1:
      image: couchbase/server:${CB_TAG}
      container_name: ${NODE1}
      hostname: couchbase1
      domainname: cb.local
      ports:
        - "18091-18096:8091-8096"   # UI/REST para nodo1
      volumes:
        - ./couchbase/data1:/opt/couchbase/var
        - ./couchbase/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./couchbase/config:/config
      environment:
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 20

    n2:
      image: couchbase/server:${CB_TAG}
      container_name: ${NODE2}
      hostname: couchbase2
      domainname: cb.local
      ports:
        - "28091-28096:8091-8096"   # UI/REST para nodo2
      volumes:
        - ./couchbase/data2:/opt/couchbase/var
        - ./couchbase/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./couchbase/config:/config
      environment:
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 20

    n3:
      image: couchbase/server:${CB_TAG}
      container_name: ${NODE3}
      hostname: couchbase3
      domainname: cb.local
      ports:
        - "38091-38096:8091-8096"   # UI/REST para nodo3
      volumes:
        - ./couchbase/data3:/opt/couchbase/var
        - ./couchbase/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./couchbase/config:/config
      environment:
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 20

    # Nodo para scale out
    n4:
      image: couchbase/server:${CB_TAG}
      container_name: ${NODE4}
      hostname: couchbase4
      domainname: cb.local
      ports:
        - "48091-48096:8091-8096"
      volumes:
        - ./couchbase/data4:/opt/couchbase/var
        - ./couchbase/logs:/opt/couchbase/var/lib/couchbase/logs
        - ./couchbase/config:/config
      environment:
        - CB_USERNAME=${CB_ADMIN}
        - CB_PASSWORD=${CB_ADMIN_PASS}
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 20
      deploy:
        replicas: 0  # lo levantaremos manualmente cuando sea necesario

  networks:
    default:
      name: ${CB_NETWORK}
      driver: bridge
  YAML
  ```

- **Paso 2.** Inicia el servicio, dentro de la terminal ejecuta el siguiente comando.

  > **Importante.** Para agilizar los procesos, la imagen ya está descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  {: .lab-note .important .compact}
  > **Importante.** El `docker compose up -d` corre en segundo plano. El healthcheck del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en 8091 antes de continuar.
  {: .lab-note .important .compact}

  ```bash
  docker compose up -d
  ```
  ![cbase3]({{ page.images_base | relative_url }}/3.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Crear el clúster (3 nodos) y cargar datos

Inicializarás el clúster en `n1`, unirás `n2` y `n3`, crearás el bucket y cargarás datos de prueba.



- **Paso 1.** Inicializa el clúster en **n1**, ejecuta el siguiete comando en la terminal.

  > **Nota.** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **Importante.** El comando se ejecuta desde el directorio de la práctica **practica12-rebalance**
  {: .lab-note .important .compact}

  ```bash
  set -a; source .env; set +a
  docker exec -it ${NODE1} couchbase-cli cluster-init \
    --cluster 127.0.0.1 \
    --cluster-username "$CB_ADMIN" \
    --cluster-password "$CB_ADMIN_PASS" \
    --services "${SERVICES_NORM:-$CB_SERVICES}" \
    --cluster-ramsize "$CB_RAM" \
    --cluster-index-ramsize "$CB_INDEX_RAM" \
    --index-storage-setting default
  ```
  ![cbase4]({{ page.images_base | relative_url }}/4.png)

- **Paso 2.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **Nota.**
  - Contenedor `cb-rb-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - está conexión es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-rb-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:18091/pools | head -c 200; echo
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

- **Paso 3.** Agregar el nodo `n2` al clúster (desde `n1`).

  {%raw%}
  ```bash
  CB2_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-rb-n2)
  docker exec -it ${NODE1} couchbase-cli server-add \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-add "${CB2_IP}" --server-add-username "${CB_ADMIN}" --server-add-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}"
  ```
  {%endraw%}
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 4.** Agregar el nodo `n3` al clúster (desde `n1`).

  {%raw%}
  ```bash
  CB3_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-rb-n3)
  docker exec -it ${NODE1} couchbase-cli server-add \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-add "${CB3_IP}" --server-add-username "${CB_ADMIN}" --server-add-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}"
  ```
  {%endraw%}
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 5.** Ahora realiza el **Rebalance** entre los nodos.

  {%raw%}
  ```bash
  CB1_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-rb-n1)
  docker exec -it ${NODE1} couchbase-cli rebalance \
    -c "${CB1_IP}" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  {%endraw%}
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 6.** Crear el bucket con **réplicas=1**.

  ```bash
  docker exec -it ${NODE1} couchbase-cli bucket-create \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --bucket "${CB_BUCKET}" \
    --bucket-type couchbase \
    --bucket-ramsize ${CB_BUCKET_RAM} \
    --enable-flush 1 \
    --bucket-replica ${CB_REPLICAS}
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 7.** Ahora inserta y lee documentos de prueba (N1QL).

  > **Nota.** La imagen es representativa, el resultado es más extenso.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${NODE1} cbq -e http://couchbase1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "INSERT INTO \`${CB_BUCKET}\` (KEY,VALUE) VALUES ('doc::1', {\"type\":\"demo\",\"rack\":\"A\"});" \
    -s "UPSERT INTO \`${CB_BUCKET}\` (KEY,VALUE) VALUES ('doc::2', {\"type\":\"demo\",\"rack\":\"B\"});" \
    -s "SELECT META().id AS id, rack FROM \`${CB_BUCKET}\` WHERE \`type\` = \"demo\" ORDER BY META().id;"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: **Scale Out** (agregar nodo `n4`) y Rebalance

Levantarás `n4`, lo agregarás al clúster y ejecutarás **rebalance** para redistribuir carga y datos.



- **Paso 1.** Levanta el nodo `n4`.

  ```bash
  docker compose up -d --scale n4=1 n4
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 2.** Verifica la salud hasta que este **healthy**.

  {%raw%}
  ```bash
  docker inspect -f '{{.State.Health.Status}}' ${NODE4}
  ```
  {%endraw%}
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 3.** Agregar el nodo `n4` al clúster.

  {%raw%}
  ```bash
  CB4_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-rb-n4)
  docker exec -it ${NODE1} couchbase-cli server-add \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-add "${CB4_IP}" --server-add-username "${CB_ADMIN}" --server-add-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}"
  ```
  {%endraw%}
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 4.** Rebalancea la infraestructura con el nodo 4.

  ```bash
  docker exec -it ${NODE1} couchbase-cli rebalance \
    -c "${CB1_IP}" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: **Scale In** (retirar `n2`) y Rebalance

Retirarás un nodo (`n2`) del clúster de 4 nodos y ejecutarás **rebalance** para reubicar particiones e índices.



- **Paso 1.** Expulsar `n2` con **graceful failover** y rebalance.

  ```bash
  docker exec -it ${NODE1} couchbase-cli failover \
    -c couchbase1.cb.local:8091 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-failover "${CB2_IP}:8091"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 2.** Revisar el estado de los nodos con el siguiente comando.

  ```bash
  docker exec -it ${NODE1} couchbase-cli server-list \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 3.** Detener y quitar contenedor `n2`.

  ```bash
  docker stop ${NODE2}
  docker rm ${NODE2}
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Limpieza

Borrar datos en el entorno para repetir pruebas.



- **Paso 1.** En la terminal aplica el siguiente comando para detener el nodo.

  > **Nota.** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 2.** Apagar y eliminar contenedor (se conservan los datos en ./data).

  > **Nota.** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **Importante.** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
