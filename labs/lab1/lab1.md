---
layout: lab
title: "Práctica 1: Instalación de un nodo Couchbase en Docker" # CAMBIAR POR CADA PRACTICA
permalink: /lab1/lab1/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab1/img # CAMBIAR POR CADA PRACTICA
duration: "15 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Configurar e inicializar un único nodo de Couchbase Server en Docker (local) usando Visual Studio Code y terminal Git Bash, dejando una estructura de carpetas organizada, el clúster levantado con servicios básicos (Data/Index/Query), un bucket de prueba y validaciones de estado/health para futuras prácticas.
prerequisites: # CAMBIAR POR CADA PRACTICA
  - Docker Desktop instalado y en ejecución (WSL2 recomendado en Windows).
  - Visual Studio Code instalado y terminal Git Bash.
  - 4 GB de RAM libres para el contenedor (recomendado).
  - Puertos locales disponibles **8091–8096** y **11210**.
  - Conectividad a Internet para descargar la imagen.
introduction: # CAMBIAR POR CADA PRACTICA
  - En esta práctica levantarás un nodo de Couchbase Server utilizando Docker y lo inicializarás mediante CLI/REST. Al finalizar, tendrás un contenedor corriendo con servicios Data/Index/Query, un bucket de prueba y verificaciones de salud (health checks).
slug: lab1 # CAMBIAR POR CADA PRACTICA
lab_number: 1 # CAMBIAR POR CADA PRACTICA
final_result: > # CAMBIAR POR CADA PRACTICA
  Un nodo de Couchbase corriendo en Docker local, inicializado con servicios Data/Index/Query, con un bucket de prueba y validaciones por CLI/REST/UI. Estructura de carpetas organizada para reutilización en prácticas posteriores.
notes: # CAMBIAR POR CADA PRACTICA
  - Cambia la contraseña por una segura en entornos no-demo.
  - Si el healthcheck tarda demasiado, revisa RAM disponible y que no haya conflictos de puertos (8091–8096/11210).
  - En Windows con WSL2, usar rutas del proyecto dentro de `~` o del sistema de archivos de Linux (mejor performance).
  - Si `cbq` no está en `PATH` dentro del contenedor, llama con ruta completa `/opt/couchbase/bin/cbq`.
  - Para CE (Community Edition) puedes usar `couchbase/server:community`; para Enterprise (trial), `couchbase/server:enterprise`.
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Documentación oficial Couchbase Server (instalación y Docker)
    url: https://docs.couchbase.com/server/current/install/install-docker.html
  - text: couchbase-cli (referencia de comandos)
    url: https://docs.couchbase.com/server/current/cli/cbcli/couchbase-cli-intro.html
  - text: REST API (administración)
    url: https://docs.couchbase.com/server/current/rest-api/rest-intro.html
  - text: Puertos de Couchbase
    url: https://docs.couchbase.com/server/current/install/install-ports.html
prev: / # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab2/lab2/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---
<!-- EJEMPLO DE UNA TAREA -->
### Tarea 1. Preparar el espacio de trabajo en VS Code

Crearás una carpeta dedicada a la práctica, abrirás el proyecto en VS Code y validarás que Docker Desktop esté operativo con Git Bash como terminal.

#### Tarea 1.1

- **Paso 1.** Crea una carpeta en el escritorio de la **Maquina Virtual** llamada **cs300-labs**.

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 2.** Abre el software de **Visual Studio Code**.

  > **NOTA:** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 3.** Ya que tengas **Visual Studio Code** abierto, ahora da clic en `Open Folder`.

  > **NOTA:** Tambien puedes dar clic en **File** -> **Open Folder**.
  {: .lab-note .info .compact}

  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la ventana del **Explorador de Windows** busca y selecciona la carpeta creada **`cs300-labs`**.

- **Paso 5.** Verifica que haya cargado correctamente el directorio raíz.

  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 6.** Ahora da clic en el icono de la imagen para abrir la terminal, **se encuentra en la parte superior derecha.**

  ![cbase4]({{ page.images_base | relative_url }}/4.png)

- **Paso 7.** En la **barra de herramientas de la terminal** que se encuentra en el lado derecho, **da clic en la pestaña para expandir el menu.**

  ![cbase5]({{ page.images_base | relative_url }}/5.png)

- **Paso 8.** Selecciona del menu la opción **Select Default Profile**, como lo muestra la imagen.

  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 9.** Se abrira una ventana superior central, ahi selecciona la opcion **Git Bash** para la terminal predeterminada.

  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 10.** Ahora cierra la terminal para que tome efecto, da clic en el botón del icono del **bote de basura**.

  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 11.** Vuelve a abrir la terminal como en pasos anteriores.

  ![cbase4]({{ page.images_base | relative_url }}/4.png)

- **Paso 12.** Verifica que aparezca la terminal en **Git Bash** y la ruta del directorio **cs300-labs** como lo muestra la imagen.

  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 13.** En tu carpeta de `cs300-labs`, crea la siguiente estructura base, **copia y pega el siguiente comando en la terminal.**

  ```bash
  mkdir -p ~/Desktop/cs300-labs/practica1-single-node/{data,config,logs,scripts}
  cd ~/Desktop/cs300-labs/practica1-single-node
  ```

- **Paso 14.** Verifica que se muestre correctamente el **directorio de la practica 1 y los sudirectorios.**

  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 15.** Verifica que Docker esté activo y accesible, copi y pega los siguientes comandos en la terminal.

  > **IMPORTANTE:** Si no esta activo, puedes buscar el software de Docker en las aplicaciones del Windows.
  {: .lab-note .important .compact}

  ```bash
  docker version
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Definir variables y Docker Compose

Generarás un archivo `.env` con variables (usuario, contraseña, tag de imagen, memoria) y un `compose.yaml` que mapea puertos, volúmenes y declara la política de healthcheck.

#### Tarea 2.1

- **Paso 16.** En la terminal de **VSC** copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **NOTA:** El archivo `.env` estandariza credenciales y memoria.
  {: .lab-note .info .compact}

  ```bash
  cat > .env << 'EOF'
  CB_TAG=enterprise
  CB_CONTAINER_NAME=cbnode1
  CB_USER=admin
  CB_PASS='adminlab'
  CB_SERVICES=data,index,query
  CB_RAM=2048
  CB_INDEX_RAM=512
  CB_BUCKET=dev
  CB_BUCKET_RAM=256
  EOF
  ```

- **Paso 17.** El comando no dara salida pero debes de ver correctamente la creación del archivo `.env`.

  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 18.** Ahora crea el archivo **Docker Compose** llamado **compose.yaml**. Copia y pega el siguiente codigo en la terminal.

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

- **Paso 19.** Verifica la creación del archivo correctamente.

  ![cbase13]({{ page.images_base | relative_url }}/13.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Descargar imagen y arrancar el contenedor

Levantarás el contenedor con Docker Compose y confirmarás que la imagen se descarga y el servicio inicia.

#### Tarea 3.1

- **Paso 20.** Inicia el servicio, dentro de la terminal ejecuta el siguiente comando.

  > **IMPORTANTE:** Para agilizar los procesos, la imagen ya esta descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  {: .lab-note .important .compact}
  > **IMPORTANTE:** El `docker compose up -d` corre en segundo plano. El healthcheck del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en 8091 antes de continuar.
  {: .lab-note .important .compact}

  ```bash
  docker compose up -d
  ```

  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 21.**  Verifica que se haya levantado correctamente el contenedor, escribe le siguiente comando em la terminal.

  ```bash
  export CB_CONTAINER_NAME=cbnode1
  docker ps --filter "name=${CB_CONTAINER_NAME}"
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 22.** Valida con el comando de **inspect** que en efecto este **Healthy** el contenedor.

  > **NOTA:** El contenedor debe estar `Up` y el healthcheck en `healthy` (puede tardar 30–90s en la primera subida). 
  {: .lab-note .info .compact}

  {% raw %}
  ```bash
  docker inspect -f '{{.State.Health.Status}}' ${CB_CONTAINER_NAME}
  ```
  {% endraw %}
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 23.** Comprueba que el endpoint base este funcionando bien.

  > **NOTA:**
  - El `curl` debe regresar JSON (sin HTML de error).
  - Puedes validar con cualquiera de las 2 opciones **jq** o **python**
  {: .lab-note .info .compact}
  > **IMPORTANTE:** La imagen **representa una parte del resultado** ya que es largo el **json**
  {: .lab-note .important .compact}

  ```bash
  curl -fsS http://localhost:8091/pools | jq .
  ```
  ```bash
  curl -fsS http://localhost:8091/pools | python -m json.tool
  ```

  ![cbase17]({{ page.images_base | relative_url }}/17.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Inicializar clúster y servicios con CLI

Usarás `couchbase-cli` dentro del contenedor para inicializar el clúster, definir cuotas de memoria, habilitar servicios y crear un bucket de prueba.

#### Tarea 3.1

- **Paso 24.** Inicializa el clúster ejecuta el siguiete comando en la terminal.

  > **NOTA:** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM. Habilitar `flush` permite vaciar el bucket desde la UI o CLI.
  {: .lab-note .info .compact}
  > **IMPORTANTE:** El comando se ejecuta desde el directorio de la practica **practica1-single-node**
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
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 25.** Verifica el estado del clúster

  ```bash
  docker exec -it ${CB_CONTAINER_NAME} couchbase-cli server-list \
    -c 127.0.0.1 -u "${CB_USER}" -p "${CB_PASS}"
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 26.** Crea un bucket de prueba de manera directa con el siguiente comando en la terminal.

  ```bash
  docker exec -it ${CB_CONTAINER_NAME} couchbase-cli bucket-create \
    -c 127.0.0.1 -u "${CB_USER}" -p "${CB_PASS}" \
    --bucket "${CB_BUCKET}" \
    --bucket-type couchbase \
    --bucket-ramsize ${CB_BUCKET_RAM} \
    --enable-flush 1
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 27.** Verifica que el bucket se haya creado correctamente, escribe el siguiente comando en la terminal.

  ```bash
  docker exec -it ${CB_CONTAINER_NAME} couchbase-cli bucket-list \
    -c 127.0.0.1 -u "${CB_USER}" -p "${CB_PASS}"
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Validaciones de salud y acceso a la Web Console

Probarás endpoints REST básicos, health del contenedor y accederás a la consola web para verificar servicios y bucket.

#### 5.1

- **Paso 28.** Comprueba la salud del contenedor:

  {% raw %}
  ```bash
  docker inspect -f '{{.State.Health.Status}}' ${CB_CONTAINER_NAME}
  ```
  {% endraw %}
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

- **Paso 29.** Verifica servicios vía REST, escribe el siguiente comando en la terminal.

  > **IMPORTANTE:** La salida de ambos comandos es muy extensa puedes tomarte unos segundos para analizarla.
  {: .lab-note .important .compact}

  ```bash
  curl -fsS -u "${CB_USER}:${CB_PASS}" http://localhost:8091/pools/default | jq '.'
  ```
  ```bash
  curl -fsS -u "${CB_USER}:${CB_PASS}" http://localhost:8091/pools/default/buckets | jq '.'
  ```

- **Paso 30.** Abre la consola web, y verifica que todo este correctamente. Abre la siguiente URL en el navegador **Google Chrome**

  > **IMPORTANTE:** Usa los siguientes datos par autenticarte.
  - **Usuario:** `admin`
  - **Contraseña:** `adminlab`
  {: .lab-note .important .compact}

  ```bash
  http://localhost:8091/
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

- **Paso 31.** Ya dentro de la pagina web de **Couchbase** da clic en **Buckets** del menu lateral izquierdo.

  > **NOTA:** Debes observar el bucket llamado **dev** creado previamente.
  {: .lab-note .info .compact}

  ![cbase24]({{ page.images_base | relative_url }}/24.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Limpieza y reinicio (buenas prácticas)

Aprenderás a apagar/encender el servicio y a limpiar volúmenes si necesitas empezar desde cero.

#### Tarea 6.1

- **Paso 32.** Ahora regresa a la terminal de **Git Bash** para aplicar el siguiente comando.

  > **NOTA:** Si es necesario puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase25]({{ page.images_base | relative_url }}/25.png)

- **Paso 33.** Apagar y eliminar contenedor (se conservan los datos en ./data)

  > **NOTA:** Si es necesario puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}

  ```bash
  docker compose down
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}