---
layout: lab
title: "Práctica 6: TLS con certificados autofirmados" # CAMBIAR POR CADA PRACTICA
permalink: /lab6/lab6/ # CAMBIAR POR CADA PRACTICA
images_base: /labs/lab6/img # CAMBIAR POR CADA PRACTICA
duration: "25 minutos" # CAMBIAR POR CADA PRACTICA
objective: # CAMBIAR POR CADA PRACTICA
  - Implementar y configurar TLS (Transport Layer Security) utilizando certificados autofirmados en un entorno Docker con Couchbase, proporcionando una comprensión práctica de la seguridad en comunicaciones y la configuración de certificados SSL/TLS en contenedores.
prerequisites: # CAMBIAR POR CADA PRACTICA
  - Visual Studio Code con terminal **Git Bash**.
  - 6–8 GB de RAM libres recomendados para 3 nodos (mínimo ~5 GB).  
  - Puertos locales disponibles **8091–8096**, **11210** (se publicarán solo desde el nodo 1).
  - Conectividad a Internet para descargar la imagen.
  - Opcional `jq` para mejorar salidas JSON.
introduction: | # CAMBIAR POR CADA PRACTICA
  Transport Layer Security (TLS) es un protocolo criptográfico que proporciona comunicaciones seguras a través de una red. Es el sucesor de Secure Sockets Layer (SSL) y se utiliza ampliamente para asegurar conexiones web, correo electrónico y otras comunicaciones.

  En está práctica, implementarás un entorno completo de TLS usando:
  - **Docker**: para containerizar nuestros servicios.
  - **Couchbase**: como base de datos NoSQL que soporta TLS.
  - **OpenSSL**: para generar nuestros certificados.
  - **Certificados autofirmados**: para establecer conexiones seguras.
slug: lab6 # CAMBIAR POR CADA PRACTICA
lab_number: 6 # CAMBIAR POR CADA PRACTICA
final_result: | # CAMBIAR POR CADA PRACTICA
  Al completar está práctica, habrás logrado:

  1. **Configuración completa de TLS**: Un entorno Docker con Couchbase ejecutándose con certificados TLS autofirmados
  2. **Comprensión práctica**: Experiencia hands-on con generación de certificados, configuración de servicios y validación de TLS
  3. **Habilidades transferibles**: Conocimientos aplicables a otros servicios y entornos de producción
  4. **Entorno funcional**: Un sistema completamente operativo con conexiones seguras verificadas
notes: | # CAMBIAR POR CADA PRACTICA
  Seguridad
  - **Certificados autofirmados**: Solo para desarrollo y aprendizaje
  - **Contraseñas**: Cambiar credenciales por defecto en entornos reales
  - **Permisos**: Los archivos de claves privadas deben tener permisos restrictivos

  Limitaciones
  - Los navegadores mostrarán advertencias de seguridad
  - Los certificados autofirmados no son válidos para producción
  - La configuración es específica para entornos de desarrollo local
references: # CAMBIAR POR CADA PRACTICA LINKS ADICIONALES DE DOCUMENTACION
  - text: Documentación oficial de Couchbase TLS
    url: https://docs.couchbase.com/server/current/manage/manage-security/configure-server-certificates.html
  - text: Guía de OpenSSL
    url: https://www.openssl.org/docs/man1.1.1/man1/openssl.html
  - text: Docker Compose Reference
    url: https://docs.docker.com/compose/compose-file/
  - text: TLS Best Practices
    url: https://wiki.mozilla.org/Security/Server_Side_TLS
prev: /lab5/lab5/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ATRAS        
next: /lab7/lab7/ # CAMBIAR POR CADA PRACTICA MENU DE NAVEGACION HACIA ADELANTE
---


---

### Tarea 1. Preparar la estructura de trabajo y verificar puertos TLS

Crearás una carpeta para la práctica, validarás que el contenedor `cbnode1` esté corriendo y que los puertos `TLS (18091)` respondan con el certificado por defecto de Couchbase.


- **Paso 1.** Abre el software de **Visual Studio Code**.

  > **Nota.** Puedes encontrarlo en el **Escritorio** o en las aplicaciones del sistema de **Windows**
  {: .lab-note .info .compact}

- **Paso 2.** Ya que tengas **Visual Studio Code** abierto, da clic en el icono de la imagen para abrir la terminal. **Se encuentra en la parte superior derecha.**.

  ![cbase1]({{ page.images_base | relative_url }}/1.png)

- **Paso 3.** Para ejecutar el siguiente comando, debes estar en el directorio **`cs300-labs`**. Puedes guiarte de la siguiente imagen.

  ```bash
  mkdir -p practica6-tls/
  mkdir -p practica6-tls/scripts
  mkdir -p practica6-tls/certs
  cd practica6-tls
  ```
  ![cbase2]({{ page.images_base | relative_url }}/2.png)

- **Paso 4.** En la terminal de **VSC**, copia y pega el siguiente comando que crea el archivo `.env` y carga el contenido de las variables necesarias.

  > **Nota.** El archivo `.env` estandariza credenciales y memoria.
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

  # Usuarios para la práctica
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

- **Paso 5.** Ahora, crea el archivo **Docker Compose** llamado **`compose.yaml`**. Copia y pega el siguiente código en la terminal.

  > **Notas**
  - El archivo `compose.yaml` mapea puertos `8091–8096` para la consola web y `11210` para clientes.
  - El healthcheck consulta el endpoint `/pools` que responde cuando el servicio está arriba (aunque aún no esté inicializado).
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

- **Paso 6.** Inicia el servicio. Dentro de la terminal, ejecuta el siguiente comando.

  > **Importante.** Para agilizar los procesos, la imagen ya está descargada en tu ambiente de trabajo, ya que puede tardar hasta 10 minutos en descargarse.
  {: .lab-note .important .compact}
  > **Importante.** El `docker compose up -d` corre en segundo plano. El healthcheck del servicio y la sonda de `compose.yaml` garantizan que Couchbase responda en `8091` antes de continuar.
  {: .lab-note .important .compact}

  ```bash
  docker compose up -d
  ```

  ![cbase3]({{ page.images_base | relative_url }}/3.png)

- **Paso 7.** Inicializa el clúster, ejecuta el siguiente comando en la terminal.

  > **Nota.** El `cluster-init` fija credenciales y cuotas de memoria (data/Index). Para un nodo local, 2 GB total y 512 MB para Index es razonable; ajusta según tu RAM. Habilitar `flush` permite vaciar el bucket desde la UI o CLI.
  {: .lab-note .info .compact}
  > **Importante.** El comando se ejecuta desde el directorio de la práctica **`practica6-tls`**.
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

- **Paso 8.** Verifica que el clúster esté **healthy** y que se muestre el JSON con las propiedades del nodo.

  > **Nota.**
  - El contenedor `cbnode1` aparece **Up**.  
  - `curl` devuelve JSON de la información del nodo.
  - Esta conexion es mediante HTTP.
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

### Tarea 2. Generación de certificados autofirmados

Activarás la auditoría, configurarás la ruta de logs, el tamaño de rotación y revisarás la configuración vía `REST`.


- **Paso 1.** Primero, accede al directorio **`certs`**, con el siguiente comando.

  ```bash
  cd certs
  ```

- **Paso 2.** Ahora, genera la clave privada de la CA (`2048 bits`). Copia y pega el siguiente comando.

  > **Importante.** El comando no tendrá salida, pero puedes ejecutar **`ls`** para verificar.
  {: .lab-note .important .compact}

  ```bash
  openssl genrsa -out ca.key 2048
  ```
  ![cbase6]({{ page.images_base | relative_url }}/6.png)

- **Paso 3.** Genera el certificado de la CA (válido por 365 días) para la prueba de la práctica.

  > **Importante.** Si todo salió bien, el comando no imprimirá salida. El archivo se guarda donde ejecutaste el comando (tu directorio actual) **`practica6-tls/certs`**.
  {: .lab-note .important .compact}

  ```bash
  MSYS2_ARG_CONV_EXCL='*' \
  openssl req -new -x509 -days 365 \
    -key ca.key -out ca.crt \
    -subj "/C=MX/ST=Estado/L=Ciudad/O=MiOrganizacion/CN=MiCA"
  ```
  ![cbase7]({{ page.images_base | relative_url }}/7.png)

- **Paso 4.** Ahora, genera la clave privada del servidor.

  > **Importante.** El comando no tendrá salida, pero puedes ejecutar **`ls`** para verificar.
  {: .lab-note .important .compact}

  ```bash
  openssl genrsa -out server.key 2048
  ```
  ![cbase8]({{ page.images_base | relative_url }}/8.png)

- **Paso 5.** Genera la solicitud de certificado (CSR).

  > **Importante.** Si todo salió bien, el comando no imprimirá salida. El archivo se guarda donde ejecutaste el comando (tu directorio actual) **`practica6-tls/certs`**.
  {: .lab-note .important .compact}
  
  ```bash
  export MSYS2_ARG_CONV_EXCL='*'
  openssl req -new -key server.key -out server.csr \
    -subj "/C=MX/ST=Estado/L=Ciudad/O=MiOrganizacion/CN=localhost" \
    -addext "subjectAltName = DNS:localhost,IP:127.0.0.1"
  ```
  ![cbase9]({{ page.images_base | relative_url }}/9.png)

- **Paso 6.** Crea un archivo de configuración para extensiones.

  ```bash
  cat > server.conf << EOF
  [req]
  distinguished_name = req_distinguished_name
  req_extensions = v3_req

  [req_distinguished_name]

  [v3_req]
  subjectAltName = @alt_names

  [alt_names]
  DNS.1 = localhost
  DNS.2 = *.localhost
  IP.1 = 127.0.0.1
  IP.2 = ::1
  EOF
  ```

- **Paso 7.** Ahora, se debe firmar el certificado del servidor con nuestra CA.

  > **Importante.** El comando no tendrá salida, pero puedes ejecutar **`ls`** para verificar.
  {: .lab-note .important .compact}

  ```bash
  openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -extensions v3_req -extfile server.conf
  ```
  ![cbase10]({{ page.images_base | relative_url }}/10.png)

- **Paso 8.** Verifica la creación del archivo **`ca.crt`**.

  > **Nota.** La salida del comando es muy extensa, la imagen solo representa parte de la salida.
  {: .lab-note .info .compact}

  ```bash
  openssl x509 -in ca.crt -text -noout
  ```
  ![cbase11]({{ page.images_base | relative_url }}/11.png)

- **Paso 9.** Verifica la creación del archivo **`server.crt`**.

  > **Nota.** La salida del comando es muy extensa, la imagen solo representa parte de la salida.
  {: .lab-note .info .compact}

  ```bash
  openssl x509 -in server.crt -text -noout
  ```
  ![cbase12]({{ page.images_base | relative_url }}/12.png)

- **Paso 10.** Verifica ahora la cadena de confianza entre **`ca.crt`** y **`server.crt`**.

  > **Nota.** La salida del comando es muy extensa, la imagen solo representa parte de la salida.
  {: .lab-note .info .compact}

  ```bash
  openssl verify -CAfile ca.crt server.crt
  ```
  ![cbase13]({{ page.images_base | relative_url }}/13.png)

![cbase5]({{ page.images_base | relative_url }}/5.png)
{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Configuración de Docker Compose

Configurar Docker Compose para ejecutar Couchbase con soporte TLS, utilizando los certificados generados.


- **Paso 1.** Primero, debes regresar al directorio raíz. Escribe el siguiente comando.

  ```bash
  cd ..
  ```

- **Paso 2.** Para que no cause conflicto con el clúster creado previamente, dálo de baja con el siguiente comando.

  ```bash
  docker compose  down
  ```

- **Paso 3.** Crear el archivo `docker-composehttps.yml` para crear el nuevo nodo que usará los certificados.

  ```bash
  cat > docker-composehttps.yml << 'EOF'
  services:
    couchbase:
      image: couchbase:enterprise-7.2.0
      container_name: couchbase-tls
      ports:
        # Puertos estándar de Couchbase
        - "8091:8091"   # Web Admin Console
        - "8092:8092"   # Views, queries, XDCR
        - "8093:8093"   # Query services (N1QL)
        - "8094:8094"   # Full-text Search
        - "8095:8095"   # Analytics
        - "8096:8096"   # Eventing
        - "11210:11210" # Data Service
        
        # Puertos seguros TLS
        - "18091:18091" # Web Admin Console (HTTPS)
        - "18092:18092" # Views, queries, XDCR (HTTPS)
        - "18093:18093" # Query services (HTTPS)
        - "18094:18094" # Full-text Search (HTTPS)
        - "18095:18095" # Analytics (HTTPS)
        - "18096:18096" # Eventing (HTTPS)
        - "11207:11207" # Data Service (TLS)
      
      volumes:
        # Montar certificados en el contenedor
        - ./certs:/opt/couchbase/var/lib/couchbase/inbox:ro
        - couchbase-data:/opt/couchbase/var
      
      environment:
        - CLUSTER_NAME=secure-cluster
        - CLUSTER_USERNAME=admin
        - CLUSTER_PASSWORD=adminhttps
        - CLUSTER_RAMSIZE=1024
        - CLUSTER_INDEX_RAMSIZE=512
        - CLUSTER_FTS_RAMSIZE=512
      
      healthcheck:
        test: ["CMD-SHELL", "curl -fsS -u $${CB_USERNAME}:$${CB_PASSWORD} http://127.0.0.1:8091/pools >/dev/null || exit 1"]
        interval: 10s
        timeout: 5s
        retries: 12
        start_period: 60s

  volumes:
    couchbase-data:
      driver: local

  networks:
    default:
      name: couchbase-network
  EOF
  ```

- **Paso 4.** Corre el contenedor, escribe el siguiente comando.

  > **Nota.** Puede tardar unos minutos en lo que descarga la imagen.
  {: .lab-note .info .compact}
  
  ```bash
  docker compose -f docker-composehttps.yml up -d
  ```

- **Paso 5.** Verifica con el siguiente comando que el contenedor esté **healthy**.

  ```bash
  docker compose ps
  ```
  ![cbase14]({{ page.images_base | relative_url }}/14.png)

- **Paso 6.** Ahora, crea el script de configuración inicial.

  ```bash
  cat > scripts/init-cluster.sh << 'EOF'
  #!/bin/bash

  # Esperar a que Couchbase esté disponible
  echo "Esperando a que Couchbase esté disponible..."
  until curl -s http://localhost:8091/pools > /dev/null; do
    sleep 5
  done

  echo "Couchbase está disponible. Configurando TLS..."

  # Configurar certificados TLS
  curl -X POST http://localhost:8091/controller/uploadClusterCA \
    -u admin:adminhttps \
    --data-binary @/opt/couchbase/var/lib/couchbase/inbox/ca.crt

  curl -X POST http://localhost:8091/node/controller/reloadCertificate \
    -u admin:adminhttps \
    --data-binary @/opt/couchbase/var/lib/couchbase/inbox/server.crt

  echo "Configuración TLS completada."
  EOF

  # Hacer el script ejecutable
  chmod +x scripts/init-cluster.sh
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Despliegue y configuración inicial

Configurarás Couchbase para usar TLS con los certificados generados.


- **Paso 1.** Verifica los logs del contenedor.

  ```bash
  docker-compose logs couchbase
  ```
  ![cbase15]({{ page.images_base | relative_url }}/15.png)

- **Paso 2.** Verifica la disponibilidad de los servicios de couchbase debe retornar un JSON con la información.

  ```bash
  curl -s http://localhost:8091/pools
  ```
  ![cbase16]({{ page.images_base | relative_url }}/16.png)

- **Paso 3.** Ahora, ejecuta el comando para inicializar el clúster.

  ```bash
  docker exec couchbase-tls couchbase-cli cluster-init \
    --cluster-username admin \
    --cluster-password adminhttps \
    --cluster-name secure-cluster \
    --cluster-ramsize 1024 \
    --cluster-index-ramsize 512 \
    --cluster-fts-ramsize 512 \
    --services data,index,query,fts
  ```
  ![cbase17]({{ page.images_base | relative_url }}/17.png)

- **Paso 4.** Ahora, verifica la configuración del clúster.

  > **Nota.** La salida del comando es un poco extensa, la imagen es representativa.
  {: .lab-note .info .compact}

  ```bash
  docker exec couchbase-tls couchbase-cli server-info \
    -c localhost:8091 \
    -u admin \
    -p adminhttps
  ```
  ![cbase18]({{ page.images_base | relative_url }}/18.png)

- **Paso 5.** Carga el certificado CA TLS al contenedor de couchbase.

  ```bash
  docker exec couchbase-tls curl -X POST http://localhost:8091/controller/uploadClusterCA \
    -u admin:adminhttps \
    --data-binary @/opt/couchbase/var/lib/couchbase/inbox/ca.crt
  ```
  ![cbase19]({{ page.images_base | relative_url }}/19.png)

- **Paso 6.** Carga el certificado del servidor TLS al contenedor de couchbase.

  ```bash
  docker exec couchbase-tls curl -X POST http://localhost:8091/node/controller/reloadCertificate \
    -u Administrator:password123 \
    --data-binary @/opt/couchbase/var/lib/couchbase/inbox/server.crt
  ```
  ![cbase20]({{ page.images_base | relative_url }}/20.png)

- **Paso 7.** Ahora, habilita la propiedad TLS en el clúster de Couchbase con el siguiente comando.

  ```bash
  docker exec couchbase-tls couchbase-cli setting-security \
    -c localhost:8091 \
    -u admin \
    -p adminhttps \
    --set \
    --tls-min-version tlsv1.2
  ```
  ![cbase21]({{ page.images_base | relative_url }}/21.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Validación y pruebas de conectividad TLS

Verifica que la implementación TLS funciona correctamente mediante diferentes métodos de validación.


- **Paso 1.** Prueba la conectividad TLS al puerto `18091` (Web Admin HTTPS).

  > **Nota.** La salida del comando es un poco extensa, la imagen es representativa.
  {: .lab-note .info .compact}

  ```bash
  openssl s_client -connect localhost:18091 -servername localhost
  ```
  ![cbase22]({{ page.images_base | relative_url }}/22.png)

- **Paso 2.** Verifica la configuración del certificado específicamente.

  > **Nota.** La salida del comando es un poco extensa, la imagen es representativa.
  {: .lab-note .info .compact}

  ```bash
  echo | openssl s_client -connect localhost:18091 -servername localhost 2>/dev/null | openssl x509 -noout -text
  ```
  ![cbase23]({{ page.images_base | relative_url }}/23.png)

- **Paso 3.** Prueba con la verificación del certificado.

  > **Nota.** Debería fallar por ser autofirmado.
  {: .lab-note .info .compact}
  
  ```bash
  echo | openssl s_client -connect localhost:18091 -servername localhost -verify_return_error
  ```
  ![cbase24]({{ page.images_base | relative_url }}/24.png)

- **Paso 4.** Abre la consola web y verifica que todo esté correctamente. Abre la siguiente URL en el navegador **Google Chrome**.

  > **Importante.** Usa los siguientes datos para autenticarte.
  - Acepta la advertencia de certificado autofirmado.
  - Verifica que aparece la interfaz de Couchbase.
  - **Usuario:** `admin`.
  - **Contraseña:** `adminhttps`.
  {: .lab-note .important .compact}

  ```bash
  https://localhost:18091/
  ```
  ![cbase25]({{ page.images_base | relative_url }}/25.png)

- **Paso 5.** Prueba la API REST con HTTPS.

  ```bash
  curl -k -u admin:adminhttps https://localhost:18091/pools
  ```
  ![cbase26]({{ page.images_base | relative_url }}/26.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Limpieza

Eliminarás el clúster para reutilizar los recursos en las próximas prácticas.


- **Paso 1.** En la terminal, aplica el siguiente comando para detener el nodo.

  > **Nota.** Si es necesario, puedes volver a encender los contenedores con el comando **`docker compose start`**
  {: .lab-note .info .compact}

  ```bash
  docker compose stop
  ```
  ![cbase27]({{ page.images_base | relative_url }}/27.png)

- **Paso 2.** Apagar y eliminar el contenedor (se conservan los datos en `./data`).

  > **Nota.** Si es necesario, puedes volver a activar los contenedores con el comando **`docker compose up -d`**
  {: .lab-note .info .compact}
  > **Importante.** Es normal el mensaje del objeto de red **`No resource found to remove`**.
  {: .lab-note .important .compact}

  ```bash
  docker compose down
  ```
  ![cbase28]({{ page.images_base | relative_url }}/28.png)

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
