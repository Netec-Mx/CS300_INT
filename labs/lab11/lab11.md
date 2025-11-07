---
layout: lab
title: "Práctica 11: Failover manual y automatico" # CAMBIAR POR CADA PRACTICA
permalink: /lab11/lab11/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab11/img # CAMBIAR POR CADA PRACTICA
duration: "30 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Desplegar un clúster de 3 nodos de Couchbase en Docker y practicar **failover manual (graceful/force)** y **auto-failover**, incluyendo **configuración de políticas**, **simulación de fallas**, **recuperación (delta/full)** y **rebalance**; todo en **Visual Studio Code** usando **Git Bash**, cuidando escapes en comandos y validando la continuidad de servicio y la integridad de los datos con consultas N1QL.
prerequisites:  # CAMBIAR POR CADA PRACTICA
  - Software **Docker Desktop** en ejecución.  
  - Software **Visual Studio Code** con terminal **Git Bash**.  
  - Puertos libres para UI/REST por nodo `18091/28091/38091` (mapeados a `8091`).
  - Conectividad a Internet para descargar imágenes.  
  - 4–6 GB de RAM libres (3 nodos con servicios `kv,index,query`).
introduction: | # CAMBIAR POR CADA PRACTICA
  Couchbase ofrece **Auto-Failover** para sacar de rotación nodos que no responden durante un tiempo configurable, y **Failover manual** (graceful/force) gestionado por el operador. Tras el failover, puedes **recuperar** el nodo con **delta** (reutiliza datos existentes) o **full** (descarta datos) y luego ejecutar un **rebalance** para restablecer la distribución de particiones (vBuckets) y réplicas.
slug: lab11 # CAMBIAR POR CADA PRACTICA
lab_number: 11 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Formaste un clúster de 3 nodos, creaste un bucket con réplicas y configuraste **Auto-Failover**. Simulaste una falla para observar el auto-failover, ejecutaste **failover manual** (graceful/force) y llevaste a cabo **recovery (delta/full)** seguido de **rebalance**, validando que el servicio y los datos se mantienen disponibles y consistentes.
notes: # CAMBIAR POR CADA PRACTICA
  - Tiempos de auto-failover balancea `AFO_TIMEOUT` con tus SLO; valores muy bajos pueden causar failovers innecesarios.  
  - Réplicas con 3 nodos, `--replica 2` maximiza resiliencia.  
  - Durabilidad considera `DurabilityLevel` en escrituras críticas.  
  - Observabilidad revisa *Logs* y *Events* en la UI para ver entradas de failover/recovery.  
  - Red/Firewall pruebas de “falla parcial” pueden simularse con reglas de red (`docker network disconnect`) para observar comportamientos.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Failover manual (graceful/force)
    url: https://docs.couchbase.com/server/current/learn/clusters-and-availability/failover.html  
  - text: Recovery (delta/full) y rebalance
    url: https://docs.couchbase.com/server/current/learn/clusters-and-availability/rebalance.html
    
prev: /lab10/lab10/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab12/lab12/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1: Estructura base del proyecto

Crearás la carpeta independiente de la práctica y un archivo `.env` con variables estándar para 3 nodos.

#### Tarea 1.1

- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando debes estar en el directorio **cs300-labs**, puedes guiarte de la imagen.

  ```bash
  mkdir -p practica11-failover/
  mkdir -p practica11-failover/couchbase/{data1,data2,data3,logs,config}
  mkdir -p practica11-failover/couchbase/scripts
  cd practica11-failover
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  # Nombres de contenedor
  NODE1=cb-fo-n1
  NODE2=cb-fo-n2
  NODE3=cb-fo-n3

  # Admin
  CB_ADMIN=admin
  CB_ADMIN_PASS='adminlab'

  # Servicios y RAM del clúster
  CB_SERVICES=data,index,query
  CB_RAM=2048
  CB_INDEX_RAM=512
  CB_NETWORK=cbnet

  # Bucket para probar HA/Failover
  CB_BUCKET=habucket
  CB_BUCKET_RAM=512
  CB_REPLICAS=2   # recomendado con 3 nodos para resiliencia

  # Auto-failover (segundos)
  AFO_TIMEOUT=30
  AFO_MAXFAILS=1
  EOF
  ```

- **Paso 5.** Ahora crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente codigo en la terminal.

  > **NOTA:**
  - El archivo `compose.yaml` mapea puertos 8091–8096 para la consola web y 11210 para clientes.
  - El healthcheck consulta el endpoint `/pools` que responde cuando el servicio está arriba (aunque aún no inicializado).
  {: .lab-note .info .compact}

  ```bash
  cat > compose.yaml << 'YAML'
  name: couchbase-rack-awareness
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

  networks:
    default:
      name: ${CB_NETWORK}
      driver: bridge
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

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2: Crear el clúster y unir nodos

Inicializarás el clúster en `n1` y unirás `n2` y `n3`, finalizando con un rebalance.

#### Tarea 2.1

- **Paso 7.** Inicializa el clúster en **n1**, ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica11-failover**
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

- **Paso 8.** Verifica que el cluster este **healthy** y que se muestre el json con las propiedades del nodo.

  > **NOTA:**
  - Contenedor `cb-fo-n1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
  {: .lab-note .info .compact}

  ```bash
  docker ps --filter "name=cb-fo-n1"
  curl -fsS -u "$CB_ADMIN:$CB_ADMIN_PASS" http://localhost:18091/pools | head -c 200; echo
  ```
  ![cbase5]({{ page.images_base | relative_url }}/5.png)

- **Paso 9.** Agregar el nodo `n2` al clúster (desde `n1`)

  {%raw%}
  ```bash
  CB2_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-fo-n2)
  docker exec -it ${NODE1} couchbase-cli server-add \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-add "${CB2_IP}" --server-add-username "${CB_ADMIN}" --server-add-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}"
  ```
  {%endraw%}
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 10.** Agregar el nodo `n3` al clúster (desde `n1`)
  
  {%raw%}
  ```bash
  CB3_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-fo-n3)
  docker exec -it ${NODE1} couchbase-cli server-add \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-add "${CB3_IP}" --server-add-username "${CB_ADMIN}" --server-add-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}"
  ```
  {%endraw%}
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 11.** Ahora realiza el **Rebalance** entre los nodos.

  {%raw%}
  ```bash
  CB1_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cb-fo-n1)
  docker exec -it ${NODE1} couchbase-cli rebalance \
    -c "${CB1_IP}" -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  {%endraw%}
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3: Crear bucket con réplicas y datos de prueba

Crearás un bucket con réplicas y cargarás algunos documentos para evaluar continuidad ante failover.

#### Tarea 3.1

- **Paso 11.** Crear el bucket con **réplicas=2**.

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

- **Paso 12.** Ahora inserta y lee documentos de prueba (N1QL)

  > **NOTA:** La imagen es representativa, el resultado es mas extenso.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${NODE1} cbq -e http://couchbase1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "INSERT INTO ${CB_BUCKET} (KEY,VALUE) VALUES ('doc::1', {type:'demo', rack:'A'});" \
    -s "UPSERT INTO ${CB_BUCKET} (KEY,VALUE) VALUES ('doc::2', {type:'demo', rack:'B'});" \
    -s "SELECT META().id, rack FROM ${CB_BUCKET} WHERE type='demo' ORDER BY META().id;"
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4: Configurar Auto-Failover

Habilitarás auto-failover con un timeout razonable y límites de activación.

#### Tarea 4.1

- **Paso 12.** Ejecuta el siguiente comando en la terminal para ajustar el auto-failover (CLI)

  ```bash
  docker exec -it ${NODE1} couchbase-cli setting-autofailover \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --enable-auto-failover 1 \
    --auto-failover-timeout ${AFO_TIMEOUT} \
    --max-failovers ${AFO_MAXFAILS} \
    --can-abort-rebalance 1
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 13.** Verifica que la configuración se haya aplicado correctamente.

  ```bash
  docker exec -it ${NODE1} sh -lc "curl -u '${CB_ADMIN}:${CB_ADMIN_PASS}' http://127.0.0.1:8091/settings/autoFailover"
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5: Simular falla y observar Auto-Failover

Detendrás un nodo para disparar el auto-failover y validarás que el clúster sigue sirviendo datos.

#### Tarea 5.1

- **Paso 14.** Ahora deten el nodo `n3` para simular la caida.

  ```bash
  docker stop ${NODE3}
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

- **Paso 15.** Esperar al menos **30 segundos** y consulta los datos. Ejecuta el comando, ya espera los 30 segundos.

  ```bash
  sleep $((AFO_TIMEOUT+5))
  docker exec -it ${NODE1} cbq -e http://couchbase1:8093 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    -s "SELECT COUNT(*) AS n FROM ${CB_BUCKET};"
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 16.** Revisar el estado de los nodos con el siguiente comando.

  ```bash
  docker exec -it ${NODE1} couchbase-cli server-list \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6: Failover manual (graceful vs force)

Reiniciarás `n3`, lo volverás a unir (si auto-failover ocurrió) y practicarás **failover manual** en `n2` para observar diferencias entre **graceful** y **force**.


#### Tarea 6.1

- **Paso 17.** Levanta el nodo `n3` y unelo de nuevo. Ejecuta uno por uno los comandos.

  > **IMPORTANTE:** Primero dale **2 minutos** al segundo rebalance. Si el segundo rebalance se atora ejecuta **`CTRL + C`** y vuelve a intentarlo.
  {: .lab-note .important .compact}

  ```bash
  docker start ${NODE3}
  sleep 90
  ```
  ```bash
  docker exec -it ${NODE1} couchbase-cli rebalance \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  ```bash
  docker exec -it ${NODE1} couchbase-cli server-add \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-add "${CB3_IP}" --server-add-username "${CB_ADMIN}" --server-add-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}"
  ```
  ```bash
  docker exec -it ${NODE1} couchbase-cli rebalance \
    -c couchbase1.cb.local:8091 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 18.** Aplica ahora **Graceful failover** en `n2` (retira ordenadamente)

  > **NOTA:** **Graceful** intenta mover datos/servicios de forma ordenada
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${NODE1} couchbase-cli failover \
    -c couchbase1.cb.local:8091 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-failover "${CB2_IP}:8091"
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 19.** Ahora para probar **force** primero debes rebalancear al nodo 2 y luego agregarlo. Ejecuta esta serie de comandos.

  > **IMPORTANTE:** Primero dale **2 minutos** al segundo rebalance. Si el segundo rebalance se atora ejecuta **`CTRL + C`** y vuelve a intentarlo. **Si te marca error al final se debe a recursos/latencia vuelve a intentar el rebalance**
  {: .lab-note .important .compact}

  ```bash
  docker exec -it ${NODE1} couchbase-cli rebalance \
    -c couchbase1.cb.local:8091 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  ```bash
  docker exec -it ${NODE1} couchbase-cli server-add \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server-add "${CB2_IP}" --server-add-username "${CB_ADMIN}" --server-add-password "${CB_ADMIN_PASS}" \
    --services "${CB_SERVICES}"
  ```
  ```bash
  docker exec -it ${NODE1} couchbase-cli rebalance \
    -c couchbase1.cb.local:8091 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}"
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 20.** Ahora si aplica **force** para comparar el funcionamiento.

  > **NOTA:** **force/hard** asume falla irrecuperable inmediata.
  {: .lab-note .info .compact}

  ```bash
  docker exec -it ${NODE1} couchbase-cli failover \
    -c couchbase1 -u "${CB_ADMIN}" -p "${CB_ADMIN_PASS}" \
    --server couchbase2 --force
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7: Limpieza

Borrar datos en el entorno para repetir pruebas.

#### Tarea 7.1

- **Paso 21.** En la terminal aplica el siguiente comando para detener el nodo.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 22.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **IMPORTANTE:** Es normal el mensaje del objeto de red **No resource found to remove**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}