# Laboratorio de Sistemas Operativos y Redes

# Objetivo

Configurar, desplegar y probar una API RESTful Node.js con MySQL y exponerla junto a un frontend estático a través de Nginx. 
Se utilizará como base el proyecto nodejs-express-mysql de Bezkoder, que implementa operaciones CRUD sobre una entidad de ejemplo (`Tutorial`) usando Node.js, Express y MySQL.

## Prerrequisitos

El servidor **debe tener instalados** y funcionando con configuración por defecto los siguientes paquetes:

- Node.js 18+ y npm
- MySQL Server (configuración por defecto, root sin contraseña)
- Nginx (escuchando en puerto 80)
- PM2 (instalado globalmente con `npm`)

### Instalación de los Prerrequisitos

### 1. Actualizar lista de paquetes
```
sudo apt update 
```
### 2. Instalar Nginx (por defecto escucha en 80)
```
sudo apt install -y nginx
```
### 3. Instalar MySQL Server (root sin contraseña por defecto en Ubuntu 20.04)
```
sudo apt install -y mysql-server
```

### 4.  Node.js 18.x + npm
```
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -

sudo apt install -y nodejs

node -v
npm -v
```

### 5.  PM2 (global con npm)
```
sudo npm install -g pm2

pm2 -v
```
---

# Instrucciones para Instalar el Proyecto

## 1. Descargar y extraer el proyecto


**En su directorio home**, descargar el proyecto (archivo master.tar.gz) desde el siguiente link:

https://github.com/bezkoder/nodejs-express-mysql/archive/refs/heads/master.tar.gz

Y luego descomprimirlo. Los archivos se van a descargar en la carpeta `nodejs-express-mysql-master`

Cambiar el directorio de trabajo a `nodejs-express-mysql-master`

---

## 2. Instalar dependencias `node.js`
Cambiar el directorio de trabajo a `~/nodejs-express-mysql-master`
Y alli ejecutar:
```bash
npm install
```

---

## 3. Configurar la base de datos

Abrir una sesion de mysql con el usuario root y luego ingresar los siguientes comandos

```bash
sudo mysql -u root
```

```sql
CREATE DATABASE bezkoderdb;
CREATE USER 'bezkoderuser'@'localhost' IDENTIFIED WITH mysql_native_password BY 'bezkoderpass';
GRANT ALL PRIVILEGES ON bezkoderdb.* TO 'bezkoderuser'@'localhost';

USE bezkoderdb;

CREATE TABLE tutorials (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  published BOOLEAN DEFAULT false
);

FLUSH PRIVILEGES;
EXIT;
```

---

## 4. Editar archivo de configuración de base de datos

Editar `app/config/db.config.js`:

Reemplazar el contenido con:

```js
module.exports = {
  HOST: "127.0.0.1",
  USER: "bezkoderuser",
  PASSWORD: "bezkoderpass",
  DB: "bezkoderdb"
};
```

---

## 5. Crear el Frontend
Crear el archivo `/var/www/html/index.html`


Con el siguiente contenido:
```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Bezkoder Node API</title>
  <script>
    const API_BASE = '/api/tutorials';

    async function getAll() {
      const res = await fetch(API_BASE);
      const data = await res.json();
      document.getElementById('result').innerText = JSON.stringify(data, null, 2);
    }

    async function create() {
      await fetch(API_BASE, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title: "Desde Frontend", description: "Agregado desde HTML", published: false })
      });
      getAll();
    }

    async function getOne() {
      const id = document.getElementById('id').value;
      const res = await fetch(`${API_BASE}/${id}`);
      const data = await res.json();
      document.getElementById('result').innerText = JSON.stringify(data, null, 2);
    }

    async function update() {
      const id = document.getElementById('id').value;
      await fetch(`${API_BASE}/${id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title: "Actualizado", description: "Editado desde HTML", published: true })
      });
      getAll();
    }

    async function deleteOne() {
      const id = document.getElementById('id').value;
      await fetch(`${API_BASE}/${id}`, { method: 'DELETE' });
      getAll();
    }

    async function deleteAll() {
      await fetch(API_BASE, { method: 'DELETE' });
      getAll();
    }

    window.onload = getAll;
  </script>
</head>
<body>
  <h1>Frontend para API Node.js</h1>
  <p>ID: <input id="id" type="number" value="1"></p>
  <button onclick="getAll()">Listar todos</button>
  <button onclick="create()">Crear uno</button>
  <button onclick="getOne()">Obtener por ID</button>
  <button onclick="update()">Actualizar</button>
  <button onclick="deleteOne()">Eliminar por ID</button>
  <button onclick="deleteAll()">Eliminar todos</button>
  <pre id="result"></pre>
</body>
</html>
```

## 6. Configurar Nginx

### 6.1. Crear el archivo de configuración en `sites-available`(para luego activarlo desde `sites-enabled`):

Crear el archivo `/etc/nginx/sites-available/bezkoder`


Con el siguiente contenido:

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        add_header Access-Control-Allow-Origin "*" always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;
        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Content-Security-Policy "default-src 'self'" always;

        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin "*";
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
            add_header Access-Control-Allow-Headers "Content-Type, Authorization";
            add_header Content-Length 0;
            add_header Content-Type text/plain;
            return 204;
        }
    }
}

```
### 6.2. Eliminar la configuración por defecto:

Borrar el archivo `/etc/nginx/sites-enabled/default`


### 6.3. Activar el sitio y recargar Nginx:

**6.3.1.** Crear un enlace **simbólico** llamdo `/etc/nginx/sites-enabled/bezkoder` que apunte al archivo de configuración `/etc/nginx/sites-available/bezkoder`

**6.3.2.** Recargar la configuración de nginx (con `systemctl` `restart` o `reload`)
**Nota**: puede testear la sintaxis de la configuracion con `nginx -t`

---
# 7. Probar la instalación

### 7.1. Arrancar el servidor
```bash
node server.js
```
### 7.2. Probar llamadas a la API directamente con curl
Puede usar la ruta `/api/tutorials`

- Desde el servidor:

```bash
curl -i http://localhost/api/tutorials
```
**Nota:** Si esta todo correcto, debería recibir un código `200 OK` 

- Desde otro host:
```bash
curl -i http://<IP_DEL_SERVER>/api/tutorials
```
**Nota:** Si esta todo correcto, debería recibir un código `200 OK` 

### 7.3. Probar desde el Frontend

Abrir en el navegador la siguiente URL del frontend:

```http
http://<IP_DEL_SERVER>
```

Para utilizar el frontend, ingresar un ID si es necesario y hacer clic en los botones para `listar`, `crear`, `obtener`, `actualizar` o `eliminar` registros de la API. Los resultados se mostrarán debajo en formato JSON.

**La página principal del FE se vera asi:**

---

<h1>Frontend para API Node.js</h1>
<p>ID: [ <input type="number" value="1" /> ]</p>
<p>
  [<button>Listar todos</button>]
  [<button>Crear uno</button>]
  [<button>Obtener por ID</button>]
  [<button>Actualizar</button>]
  [<button>Eliminar por ID</button>]
  [<button>Eliminar todos</button>]
</p>
<pre>[
  {
    "id": 14,
    "title": "Desde Frontend",
    "description": "Agregado desde HTML",
    "published": 0
  },
  {
    "id": 15,
    "title": "Actualizado",
    "description": "Editado desde HTML",
    "published": 1
  }
]</pre>

---

# 8. Arrancar el servidor node.js (API) como demonio
**Nota** Comprobar que el servidor Node.js iniciado en el punto **7.1** esté detenido. Si sigue en ejecuión, detenerlo:
- Con `Ctrl+C` si esta corriendo en una terminal interactiva. 
- Si se ejecutó en segundo plano (background), buscar el PID y finalizarlo manualmente (`kill`)


### 8.1. Crear el archivo `~/iniciar_backend_pm2.sh`

```bash
nano ~/iniciar_backend_pm2.sh
```

Con el siguiente contenido, colocando en `APP_DIR` **la ruta real al directorio donde está el código del servidor**:
```bash
#!/bin/bash

APP_DIR="/home/<USER>/nodejs-express-mysql-master"   # <- CAMBIAR POR LA RUTA REAL !!!
APP_FILE="server.js"
APP_NAME="bezkoder-api"

# Ir al directorio del proyecto
cd "$APP_DIR" || { echo "No se pudo acceder a $APP_DIR"; exit 1; }

# Verificar si la app ya está corriendo
if pm2 list | grep -q "$APP_NAME"; then
  echo "El servicio '$APP_NAME' ya está en ejecución. Reiniciando..."
  pm2 restart "$APP_NAME"
else
  echo "Iniciando '$APP_NAME' con pm2..."
  pm2 start "$APP_FILE" --name "$APP_NAME"
  pm2 save
  echo "Configurando pm2 para iniciar automáticamente al arrancar el sistema..."
  pm2 startup | tail -n 1 | bash
fi

echo "Estado actual de PM2:"
pm2 list
```

### 8.2. Hacer el archivo `iniciar_backend_pm2.sh` ejecutable

Dar permisos de ejecución a `iniciar_backend_pm2.sh`.


## 8.3. Ejecutar `iniciar_backend_pm2.sh`
**8.3.1** Ejecutar el script desde la línea de comandos

**Nota**: Ingresar la contraseña si el sistema lo solicita.

**8.3.2** Verificar que la API quede funcionando como demonio y obtener PID de la API (`node`).

---

## 9 Configuración del Firewall

### 9.1 Configurar UFW (firewall)

>**IMPORTANTE:** si se está accediendo por SSH para realizar esta >configuración, asegurarse que el firewall (ufw) esté “inactivo”, >caso contrario perderían conexión con la máquina. 
>**Verificar:**
>
>```
>sudo ufw status
>Status: inactive
>```
>

### 9.2 Habilitar únicamente acceso SSH

1. Bloqueo de todo el tráfico que provenga del exterior:

```
sudo ufw default deny incoming
```

2. Autorización de todas las conexiones salientes:

```
sudo ufw default allow outgoing
```

3. Permitir conexiones ssh (puerto 22).

```
sudo ufw allow ssh
```

4. Verificar reglas default (antes de activar)

```
sudo cat /etc/default/ufw
```

5. Verificar que están presentes las siguientes líneas:

```
DEFAULT_INPUT_POLICY="DROP"
DEFAULT_OUTPUT_POLICY="ACCEPT"
```

6. Verificar que SSH (22) está autorizado (antes de activar)

```
sudo ufw show added
```

### 9.3 Activar el Firewall

```
sudo ufw enable
```

### 9.4 Listar reglas

```
sudo ufw status verbose
```

### 9.5 Comprobar que el acceso SSH funciona antes de continuar

Comprobar que el único acceso permitido a todos los servidores es SSH (puerto 22/TCP) intentando abrir con el browser las siguientes URLs:

```
http://<webserver1 IP>
http://<webserver1 IP>:8080
```


### 9.6 Habilitar acceso al puerto 80/TCP.

1. Abrir el puerto 80/TCP

```
sudo ufw allow 80/tcp
```

2. Confirmar los cambios

```
sudo ufw status verbose
```

3. Probar con el browser el acceso al servidor:

```
http://<IP_DEL_SERVIDOR>
```

## 10 Configurar HTTPS con self-signed certificate (certificado autofirmado)

En esta sección, se configura el acceso seguro (encriptado) HTTPS en el sitio utilizando un certificado “auto firmado”.

### 10.1 Generar clave privada (private key) y Certificado

#### 10.1.1. Preparar directorio para alojar llaves y certificados

```bash
sudo mkdir /etc/nginx/ssl
cd /etc/nginx/ssl
```

#### 10.1.2. Generar clave privada
  
```bash
sudo openssl genrsa -out /etc/nginx/ssl/bezkoder.key 2048
```
Esto crea una clave privada RSA de 2048 bits.


#### 10.1.3.  Generar solicitud de firma de certificado (CSR)
  
La CSR es el "pedido de certificado" con la info del servidor (país, organización, CN, etc.).

```
sudo openssl req -new -key /etc/nginx/ssl/bezkoder.key -out /etc/nginx/ssl/bezkoder.csr
```

OpenSSL va a preguntar campo por campo:


```
Country Name (2 letter code) [AU]:AR

State or Province Name (full name) [Some-State]: Buenos Aires

Locality Name (eg, city) []:Hurlingham

Organization Name (eg, company) [Internet Widgits Pty Ltd]: Unahur

Organizational Unit Name (eg, section) []: Laboratorio

Common Name (eg, YOUR name) []: <IP_DEL_SERVIDOR>

Email Address []: laboratorio@unahur2.edu.ar

Please enter the following 'extra' attributes
to be sent with your certificate request

A challenge password []: (Dejar en blanco)

An optional company name []: (Dejar en blanco)
``` 

#### 10.1.4. Visualizar el contenido de la solicitud de firma del certificado (csr)


```
sudo openssl req -in /etc/nginx/ssl/bezkoder.csr -noout -text
```
Ejemplo de salida:

```
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: C = AR, ST = Buenos Aires, L = Hurlingham, O = Unahur, OU = Laboratorio, CN = miwebsrv.com, emailAddress = laboratorio@unahur2.edu.ar
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:c9:5f:1a:90:b6:d4:28:9e:60:14:9a:54:ee:0d:
                    59:22:4e:a9:30:98:49:3d:af:6e:6f:41:59:f6:c9:
                    b0:2b:fa:af:ed:f7:56:fb:c6:ba:42:62:24:fa:0c:
                    77:de:56:39:e8:d8:d4:d6:04:ca:73:d3:09:52:c2:
                    92:b6:5a:66:26:84:96:4f:95:ff:a0:aa:b1:ed:39:
                    3c:cb:c6:cf:77:06:8d:13:63:52:1d:e9:ba:f3:80:
                    79:8e:11:f2:e6:aa:37:7b:81:6e:7c:0e:75:d1:df:
                    24:f8:7b:94:15:83:d3:59:81:2c:08:e0:10:77:3a:
                    8d:60:01:15:d7:19:b7:22:43:10:c3:05:52:0e:a7:
                    d3:50:b2:9f:ff:5c:90:32:a9:4b:49:82:42:8e:aa:
                    ed:5c:18:2f:b0:4d:1c:f3:4c:75:0e:84:92:ef:96:
                    ce:97:83:cf:19:35:7b:ed:ab:96:a1:af:e0:fb:c2:
                    2f:d6:74:cb:70:85:84:c5:13:2b:5c:b4:f5:5d:67:
                    f9:5b:87:2b:e1:3f:76:63:9b:2a:b2:ba:ba:22:42:
                    5a:6e:f3:13:54:ab:62:e1:e1:73:40:2e:6a:47:6f:
                    33:d0:fa:ed:15:d5:37:88:a3:74:64:ec:81:7b:a2:
                    65:64:f6:bd:b2:61:18:ea:88:c4:52:06:61:7a:9c:
                    be:41
                Exponent: 65537 (0x10001)
        Attributes:
            a0:00
    Signature Algorithm: sha256WithRSAEncryption
          a7:99:55:06:a6:ac:e2:16:41:a6:db:5f:d8:90:1c:3a:9f:91:
          85:d6:ed:2c:df:14:1f:11:ce:a5:40:96:2a:89:bd:5c:ef:14:
          8e:91:80:07:66:10:20:d9:31:75:1d:1c:74:ab:f8:74:7a:c5:
          6f:3e:ff:d2:e2:bb:9d:8e:cb:e3:33:5f:bd:6e:e0:37:49:74:
          64:cf:42:27:b3:d2:d8:ff:0e:8c:7c:f4:0e:a1:e3:49:7b:26:
          db:f4:5b:59:28:10:a5:bd:fc:7b:63:40:b6:85:d5:b4:a6:0a:
          3e:ad:36:0c:e6:29:8f:36:d6:c5:d8:c0:fd:4e:64:b5:a9:1d:
          0d:11:7d:9a:fc:9f:8e:45:f0:64:20:4e:dd:1e:28:65:22:d5:
          f9:cd:0e:47:29:7c:ba:14:e8:1c:81:a4:4a:ed:2e:ec:1a:9a:
          02:d8:bc:d1:09:c8:9b:3d:4a:2e:71:7d:90:04:dc:b0:c2:21:
          e8:e7:f2:e7:de:91:03:dd:be:72:01:7a:6c:f5:6b:de:15:68:
          1d:b1:be:5e:31:fb:60:92:3b:a5:ec:12:15:93:fe:ba:02:77:
          53:1c:de:b9:c5:66:c6:c5:ce:6c:49:27:1a:ab:5e:81:4b:ae:
          7c:39:3f:0a:02:08:a7:28:5b:33:2b:70:fe:7c:4f:14:58:a5:
          de:33:91:5d
```


#### 10.1.5. Generar el certificado autofirmado a partir de la CSR
Remitir la solicitud de firma de certificación a una entidad certificadora o **autofirmar**. 
Se firma con la clave privada y se obtiene el certificado público válido por 1 año.

Para **autofirmar** utilizar el siguiente comando:

```
sudo openssl x509 -req -days 365 -in /etc/nginx/ssl/bezkoder.csr \
  -signkey /etc/nginx/ssl/bezkoder.key -out /etc/nginx/ssl/bezkoder.crt
```

Ejemplo de salida:

```
Signature ok
subject=C = AR, ST = Buenos Aires, L = Hurlingham, O = Unahur, OU = Laboratorio, CN = miwebsrv.com, emailAddress = laboratorio@unahur2.edu.ar
Getting Private key
```


#### 10.1.6. Verificar que se hayan generado los siguientes archivos

`/etc/nginx/ssl/bezkoder.key` → clave privada

`/etc/nginx/ssl/bezkoder.csr` → solicitud de certificado (solo para ver en clase)

`/etc/nginx/ssl/bezkoder.crt` → certificado autofirmado final


>**Notas**
>
>* Puede listar el contenido de certificado Firmados o Auto-firmados asi:
>
>```
>sudo openssl x509 -subject -issuer -enddate -noout -in bezkoder.csr
>```
>
>* Puede generar el certificado autofirmado con un solo comando de la siguiente manera:
>```bash
>sudo mkdir -p /etc/nginx/ssl
>sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout /etc/nginx/ssl/>bezkoder.key   -out /etc/nginx/ssl/bezkoder.crt   -subj "/C=AR/ST=BuenosAires/>L=Hurlingham/O=Lab/OU=IT/CN=<IP_DEL_SERVIDOR>"
>```
>

### 10.2 Configurar Nginx con HTTPS
Editar `/etc/nginx/sites-available/bezkoder`:

```nginx
# Redirige todo a HTTPS
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

# Mismo sitio, ahora en 443 con TLS usando los certificado autofirmado
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate     /etc/nginx/ssl/bezkoder.crt;
    ssl_certificate_key /etc/nginx/ssl/bezkoder.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        add_header Access-Control-Allow-Origin "*" always;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
        add_header Access-Control-Allow-Headers "Content-Type, Authorization" always;
        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Content-Security-Policy "default-src 'self'" always;

        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Origin "*";
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
            add_header Access-Control-Allow-Headers "Content-Type, Authorization";
            add_header Content-Length 0;
            add_header Content-Type text/plain;
            return 204;
        }
    }
}

```

### 10.3 Reiniciar Nginx:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 10.4 Habilitar acceso al puerto 443/TCP en el firewall

1. Abrir el puerto 443/TCP

```
sudo ufw allow 443/tcp
```

2. Confirmar los cambios

```
sudo ufw status verbose
```

### 10.5 Probar Acceso a la Aplicación

Acceder en: `https://IP_DEL_SERVIDOR` (con advertencia de certificado no confiable).

---



## Anexo A.1. Recursos

Repositorio original: [https://github.com/bezkoder/nodejs-express-mysql](https://github.com/bezkoder/nodejs-express-mysql)


## Anexo A.2. Configuracion HTTP sin CORS ni Preflight
```nginx
server {
    listen 80;
    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Content-Security-Policy "default-src 'self'" always;
    }
}
```
## Anexo A.3. Configuración HTTPS sin CORS ni Preflight

```nginx
# Redirige HTTP -> HTTPS
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

# Mismo sitio en 443 con TLS (certificados ya firmados)
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate     /etc/nginx/ssl/bezkoder.crt;
    ssl_certificate_key /etc/nginx/ssl/bezkoder.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Content-Security-Policy "default-src 'self'" always;
    }
}
```


