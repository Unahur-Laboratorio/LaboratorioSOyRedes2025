# Despliegue de la Aplicación Node.js + MySQL en la nube **AWS**
> Implementación completa, paso a paso, para llevar la app del ejercicio (API Node.js del repo Bezkoder + MySQL) a AWS usando servicios:  **ECR** + **ECS Fargate** + **ALB** + **RDS MySQL** + **Secrets Manager**.

---

## 1. Prerrequisitos

1). **Acceso AWS CLI** en la instancia EC2.

2) **Docker** en la instancia EC2 para construir la imagen a subir a **ECR**.
   
3) Repositorio de la API descargado y modificado para tomar variables de entorno.

## 1.1 Configuración de AWS CLI

### 1.1.1. Arrancar Laboratorio
   
   Esperar hasta que se complete el arranque del laboratorio. Cuando el laboratorio este arrancado, el icono AWS cambiará a verde: <img src="images_docker/icono_verde.png" alt="icono" width="70" style="vertical-align:middle;" />
<br></br>

### 1.1.2. Obtener datos de configuración de AWS CLI

Hacer click en <img src="images_docker/aws_details.png" alt="icono" width="100" style="vertical-align:middle;" />
<br></br>
![Obtener Config](images_docker/image20.png)
<br></br>
En el panel derecho aparecerá la información para configurar la **Interfáz de Línea de Comandos de AWS**, hacer click en **Show**

<img src="images_docker/image21.png" alt="Descripción" width="500" />

Finalmente el panel mostrará las claves de acceso y token para configurar AWS CLI. (El ejemplo las oculta por seguridad)

<img src="images_docker/image22.png" alt="Descripción" width="400" />

---

> **El procdemiento continúa en la Instancia EC2 (shell)**

---

### 1.1.3. Configuración de la AWS CLI

> **Las instancias Amazon Linux tienen instalado por defecto a Docker y la AWS CLI. Solo hace falta configurar las credenciales como se indica en esta sección.**
>  

---

### 1.1.3.1. Crear archivos de configuración de AWS CLI

1. Abrir sesión de consola en la instancia EC2
2. Crear el directorio `~/.aws`
3. Crear el archivo `~/.aws/config`
   
   Con el siguiente contenido:
```aws
[default]
region = us-east-1
```

4. Crear el archivo `~/.aws/credentials`
    
    Y copiar en su interior el contenido de la ventana de **AWS Details (AWS CLI)** obtenida en el punto 2.
  
  Por ejemplo:
```bash
[default]
aws_access_key_id=ASIAEXAMPLEKEY12345
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG+bPxRfiCYEXAMPLEKEY
aws_session_token=IQoJb3JpZ2luX2V1EXAMPLETOKENaCXVzLXdlc3QtMiJHMEUCICDB4ZyVq...
oJgtHUWs9TX2vlpSqFyheiwm3ramZjCAJpTk14sVrqDjpniIZYJPRRD4mPHOPFRzrDKJwk7tO1y6...
+HiOLPxZTp2Ly3gXl2yhtTGVuUNb93YXh80CMea0bHzpEs0o15poEARvXTa4YIDaw086hdiAYuIg...
rV+k+LgNPQqVJlcnxG5JfpXEXAMPLETOKENCONTINUES==
```

---

### 1.1.3.2. Probar acceso a AWS

Para probar acceso a AWS ejecutar los siguientes comandos:

- Para verificar las credenciales:

```bash
aws sts get-caller-identity
```

Si está todo bien, la respuesta se verá similar a esta:

```json
{
    "UserId": "AROEXAMPLEUSERID12345:user4000004=_Student_View__Daniel_Buaon",
    "Account": "990000000001",
    "Arn": "arn:aws:sts::990000000001:assumed-role/voclabs/user4000004=_Student_View__Daniel_Buaon"
}
```

> **Configuración Completa** 

---

>**NOTA:**
Las credenciales de AWS del laboratorio son **temporales**. Expiran cuando la sesión de laboratorio expira a las **4 horas de arrancado**. 
Despues del tiempo de expiración, el laboratorio se detiene y los recursos de AWS quedan en suspenso y/o inactivos.

> **Cada vez que se arranca el laboratorio las credenciales temporales se renuevan, por lo que tienen que repetir el paso 1.1.3.1.4 actualizando el archivo `.aws/credentials`** 

>No es necesario actualizar `.aws/config`.

## 1.2 Docker
Verificar que la instancia EC2 tenga instalado Docker engine. Instalarlo si es necesario. 
> Puede seguir el procedmiento de la practica de Nube 1 para instalarlo.

## 1.3 Descargar el Repositorio de la API y modificarlo.

> El repositorio es el mismo del laboratorio 7.1-Docker-App Multicontenedor. No es necesario volver a descargar. Puede omitir el paso 1.3.1.

### 1.3.1 Descargar el repositorio y descomprimirlo

Desde el directorio home, ejecutar:

```bash
wget https://github.com/bezkoder/nodejs-express-mysql/archive/refs/heads/master.tar.gz

tar -xzf master.tar.gz
```

### 1.3.2 Modificar configuración para soportar variables de entorno

Editar el archivo `nodejs-express-mysql-master/app/config/db.config.js` y verificar que su contenido sea el siguiente:

```js
module.exports = {
  HOST: process.env.DB_HOST || "127.0.0.1",
  USER: process.env.DB_USER || "examuser",
  PASSWORD: process.env.DB_PASSWORD || "exampass",
  DB: process.env.DB_NAME || "examdb"
};
```
> Nota: el contenido de `db.config.js` es idéntico al del laboratorio 7.1-Docker-App Multicontenedor

## 2 Construir imagen Docker para la API Node.js

### 2.1 Crear el Dockerfile

Crear o editar el **Dockerfile** dentro de `nodejs-express-mysql-master` verificando que tenga el siguiente contenido:

```Dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
```
> Nota: el contenido del `Dockerfile` es idéntico al del laboratorio 7.1-Docker-App Multicontenedor
---

## 2.2 Construir y subir la imagen de la API a ECR

### 2.2.1. Crear repositorio ECR 
  
- **Alternativa 1** - Desde AWS CLI

 ```bash
 aws ecr create-repository --repository-name exam-api --region us-east-1
 ```

 - **Alternativa 2** - Desde Consola ECR
 
    - En consola AWS buscar el servicio **ECR (Elastic Container Registry)**

    - Click en **Create**

    - En **Repository Name** ingresar: **exam-api**

    - Click en **Create**

### 2.2.1. Conectar (login) Docker a ECR

#### - Obtener el ACCOUNT_ID

El ID de su cuenta de AWS (Account_Id) se puede obtener con el mismo comando AWS CLI del punto 1.1.3.2:

```bash
aws sts get-caller-identity
```

En la respuesta aparecerá indicado como **Account**
Por ej:
```
"Account": "990000000001"
```

**Tomar nota del AccountId**

#### - Login a ECR

   ```bash
   aws ecr get-login-password --region us-east-1 \
   | docker login --username AWS --password-stdin <TU_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
   ```

Por ejemplo: 
```bash
   aws ecr get-login-password --region us-east-1 \
   | docker login --username AWS --password-stdin 990000000001.dkr.ecr.us-east-1.amazonaws.com
   ```

> Si esta todo bien, verá **Login Succeeded** al final de la salida del comando.
>


### 2.3 Construir y subir la imagen (push)

```bash
cd nodejs-express-mysql-master
docker build -t exam-api:latest .
docker tag exam-api:latest <TU_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/exam-api:latest
docker push <TU_AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/exam-api:latest
```

> Reemplazar `<TU_AWS_ACCOUNT_ID>` por el Account_Id obtenido en 2.2.1
>

Al terminar verificar la creación y subida de la imagen a ECR:
- En la consola AWS buscar el servicio ECR
- En el menú de la izquierda ir a **Private registry** - **Repositories** y seleccionar el repositorio **exam-api** y hacer click en el nombre.
- Deberá ver la lista de imagenes cargadas (latest)
  
---

---

## 3. Creación de una instancia MySQL 8.0 en RDS

- En la consola de AWS buscar el servicio **Aurora and RDS**

- En el menú de la izquierda seleccionar **Databases**
- Hacer click en **Create database**

- En **Choose a database creation method** seleccionar **Standard create**
- En **Engine options** seleccionar **MySQL**  
  > debe elegir la opción que dice solo "MySQL"
- En **Engine version** seleccionar MySQL 8.0.x (MySQL 8.0.42)
- En **Templates** seleccionar **Sandbox**
- En **Availability and durability** seleccionar **Single-AZ DB instance deployment** 
- En **DB instance identifier** ingresar: **database-1**
- En **Credentials Settings:
  - **Master username**: `examuser`
  - **Credentials management** seleccionar **Self managed**
  - **Master password**: `exampass`
  - **Confirm master password**: `exampass`
- Bajar hasta el final de la página y seleccionar **Additional configuration**
- En **Database options**
  - **Initial database name**: `examdb`

> **Dejar el resto de los campos en sus valores por defecto**

- Bajar al final de la página y hacer click en **Create database**
  
Si aparece un pop-up con complementos sugeridos hacer click en **Close**

> **Esperar hasta que el estado de la base de datos sea `Available`, esto puede demorar unos minutos.**

- Una vez creada la instancia de base de datos, copiar el **Endpoint** RDS (p.ej. `examdb.xxxxxx.us-east-1.rds.amazonaws.com`).

---

## 4 Secrets Manager: credenciales y config de DB

- En la consola de AWS buscar el servicio **AWS Secrets Manager**

- Hacer click en **Store a new secret**
  > Si aparece un mensaje de error **Failed to fetch a list of Amazon DocumentDB clusters.** ignorarlo.
- En **Chose secret type** seleccionar **Other type of secret**.
- En **Key/value pairs** seleccionar **Plaintext** y reemplazar el contenido de la ventana por el siguiente JSON:

```json
{
  "DB_HOST": "<endpoint RDS>",
  "DB_USER": "examuser",
  "DB_PASSWORD": "exampass",
  "DB_NAME": "examdb"
}
```
> Reemplazar <endpoint RDS> por el Endpoint guardado al crear la base de datos.

Por ejemplo:
```json
{
  "DB_HOST": "database-1.ce6j8j1l80yk.us-east-1.rds.amazonaws.com",
  "DB_USER": "examuser",
  "DB_PASSWORD": "exampass",
  "DB_NAME": "examdb"
}
```

- Hacer click en **Next**
- En **Configure Secret**
  - **Secret name**: `exam-api-db-secret`
- Hacer click en **Next**
  > Si aparece un mensaje de error **Failed to fetch a list of Amazon DocumentDB clusters.** ignorarlo.
- Hacer click en **Next**
- Revisar la configuración y hacer click en **Store**

## 5 Iniciar el esquema en RDS

### 5.1 Revisar los SG de RDS

Debe haber un rds-sg que permita el ingreso de MySQL desde todo origen.

- Ir al servicio EC2
- En el menu de la izquierda Ir a seguridad y elegir Security Groups
- Crear un SG llamado **rds-sg** con una regla IN 3306 source 0.0.0.0

- Ir al servicio RDS y seleccionar la Base de datos
- Seleccionar la Base de Datos e ir a Modify
- Buscar la Connectivity y agregar el SG


### 5.2 Probar conectividad con MySQL

En la consola de la instancia EC2
```bash
sudo dnf install mariadb105
mysql -h <endpoint> -u admin -p
```

Ejemplo
```
mysql -h database-1.cjxro5kry7ld.us-east-1.rds.amazonaws.com -u examuser -p
``` 

Si está todo bien, se conectará a MySQL. No cerrar la sesión.

### 5.3 En mysql ingresar:

```sql
USE examdb;
CREATE TABLE IF NOT EXISTS tutorials (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  published BOOLEAN DEFAULT false
);
```

## 6 Configuración de la Orquestación de Contendores (ECS) 

### 6.1 Creación del Cluster ECS

- En la consola de AWS buscar el servicio **ECS Elastic Container Service**
- En el menu de la izquierda seleccionar **Clusters**
- Luego hacer click en **Create cluster**
- En **Cluster configuration**
  - **Cluster name**: `exam-api-cluster`
- En **Infrastructure** seleccionar **Fargate and Managerd Instances**
- En **Instance profile** seleccionar **EMR_EC2_Default**
- En **Infrastructure role** seleccionar el rol que termina con **role/LabRole**
- En **Instance selection** seleccionar **Use ECS default**
- Hacer click en **Create**

### 6.2 Definición de la Task (Contenedor API)

- En la consola ECS, en el menú de la izquierda seleccionar **Task Definitions**
- Hacer click en **Create new task definition** y luego seleccionar **Create new task definition**
- En **Task definition configuration**
  - **Task definition family**: `exam-api-task`
- En **Infrastructure requirements** seleccionar **AWS Fargate**
* **Fargate**, compatibilidad `FARGATE`.
* En **Task role** seleccionar **LabRole**
* En **Task execution role** seleccionar **LabRole**
* En **Container - 1**
  * **Container details**
    * **Name**: `exam-api`
    * **Image URI** click en **Browse ECR images** y seleccionar el repositorio: `exam-api`, luego marcar la imagen con **Image tag**: **latest** y hacer click en **Select image digest**
    * En **Port mappings** 
      * **Container port**: `8080`
      * **Protocol**: `TCP`
      * **App protocol**: `HTTP`
  * En **Environment variables** hacer click en **Add environment variable** 
    * En **Key** colocar la <key>, por ej DB_HOST, en **Value type** colocar **ValueFrom** y en **Value** colocar
    <secret ARN>:<key>:: 
    
    - El <secret ARN> lo obtiene de la consola de Secret Manager.
    Ejemplo:  `arn:aws:secretsmanager:us-east-1:807733421542:secret:exam-api-db-secret-9PCC7`

    - Las <key> son `DB_HOST`, `DB_USER`, `DB_PASSWORD` y `DB_NAME`

    Ejemplo de Value completo:
    `arn:aws:secretsmanager:us-east-1:807733421542:secret:exam-api-db-secret-9PCC7w:DB_HOST::`
  
- Crear Task
- Revisar configuracion JSON


### 6.3 Crear el Balanceador de carga

#### 6.3.1 Crear el target group
- En la consola de AWS buscar el servicio **EC2**
- En el menu de la izquierda buscar **Load Balancing** y seleccionar **Target Groups**
- Hacer click en **Create target group**
- En **Choose target type** seleccionar **IP addresses**
- En Target group name: exam-api-tg
- Protocol HTTP
- Port 8080
- IP address type IPv4
- Protocol version HTTP1
- Click en next
- En IP addresses, Step 2 remove ip address
- Ports 8080

#### 6.3.2 Crear el ALB público

- En la consola de AWS buscar el servicio **EC2**
- En el menu de la izquierda buscar **Load Balancing** y seleccionar **Load Balancers**
- Hacer click en Create load balancer
- En Load balancer types, seleccionar Application Load Balancer y hacer click en **Create**
- En Load balancer name: exam-api-lb
- Scheme Internet facing
- Marcar todas las AZ
- Crear un SG sg-alb con in 80 0.0.0.0
- Seleccionar el SG sg-alb. Mantener también el SG default.
- Listener http 80
- Default action - routing action - Forward to target group,
- Select target group exam-api-tg
- click create load balancer


### 6.4. Crear el servicio ECS(Fargate)

- En la consola de AWS buscar el servicio **ECS**
- En el menu de la izquierda seleccionar **Clusters** y luego hacer click en el nombre del cluster (**exam-api-cluster**)
- En la pestala Services hacer click en Create
- Task definition family: exam-api
- Ir a Environment - Compute configuration - advanced
  - Compute options, seleccionar Launch type
  - En Launch type, seleccionar FARGATE y Platform version: LATEST
- Ir a Load balancing
  - Seleccionar User load balancing
  - Load balancer type: seleccionar Application Load Balancer
  - Container: seleccionar exam-api 8080:8080
  - Application Load balancer: seleccionar Use an existing load balancer
  - Load balancer: seleccionar exam-api-lb
  - En Listener, seleecionar User an existing listener y luego seleccionar Listener HTTP:80
  - En Target group, seleccionar User an existing target group y luego Target group name exam-api-tg
  - Click en Create

Probar la API con curl o navegador a
http://<ALB DNS>/
http://<ALB DNS>/api/tutorials

---

## 7) Frontend 

### 7.1 Configuracion como sitio estatico en — S3

#### 7.1.1 Crear index.html (Frontend)

Crear index.html con el siguiente contenido:
> Remplazar <ALB_DNS> por el DNS Name el Load Balancer

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Examen Node API</title>
  <script>
    // Reemplazá <ALB_DNS> por el DNS del Application Load Balancer creado en AWS
    const ALB_DNS = '<ALB_DNS>'; // Ejemplo: 'exam-api-lb-1909225646.us-east-1.elb.amazonaws.com'
    const API_BASE = `http://${ALB_DNS}/api/tutorials`;

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

#### 7.1.2 Crear Frontend como sitio web estático

Ir al servicio S3
1. Crear un bucket S3 público (p.ej. `exam-frontend-bucket-<tu-iniciales>`).
2. Deseleccionar block public access
3. Create bucket
4. Subí el archivo `index.html` del frontend al bucket S3 creado.
5. Configurá el bucket para *static website hosting* desde las opciones (Properties) del bucket.
6. Copiá la URL del endpoint del sitio estático para acceder al frontend.
7. Actualizá la URL del ALB en el JS de `index.html` para que las llamadas a la API se dirijan al DNS del ALB creado en AWS.
8. Configurar permisos del bucket

```jason  
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicRead",
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::<TU_BUCKET>/*"
  }]
}
``` 

Probar el Frontend abriendo la URL del Bucket website endpoing con curl o explorador:
http://<TU_BUCKET>.s3-website-us-east-1.amazonaws.com

>**El Frontend se visualizará correctamente, pero las llamadas a la API van a fallar por CORS**

### 7.2 Agregar API Gateway para manejar CORS

#### 7.2.1 Crear API HTTP

- Ir al servicio API Gateway
- Seleccionar APIs y luego click en Create API
- Seleccionar HTTP API y hacer click en Build
  - API Name: exam-api
  - Click en Add integration
  - Seleccionar HTTP
  - Method: ANY
  - URL endpoint: colocar `http://<DNS Name del load balancer>/{proxy}`
  - Click en Next
- En Configure routes
  - Click en Add route
    - Method: ANY
    - Resource path: /{proxy+}
    - Integration target: seleccionar el URL target creado mas arriba (`http://<DNS Name del load balancer>/{proxy}`)
    - Click en Next
    - Stage name: $default
    - Auto-deploy: activado.
    - Click en Next
- Click en Create

#### 7.2.2 Configurar CORS

Abrir la API recién creada: API Gateway → APIs → exam-api

En el menú izquierdo buscar CORS

Click en Configure

Completar:

Allowed origins: (URL del Frontend en S3) ejemplo:
http://exam-api-frontent.s3-website-us-east-1.amazonaws.com

Allowed methods: GET,POST,PUT,DELETE,OPTIONS

Allowed headers: Content-Type,Authorization

Click en Save

#### 7.2.3 Modificar Frontend para usar API Gateway

- Obtener la URL del API Gateway:
  - Ir a API Gateway - APIs - exam-api
  - En el menu de la izquireda hacer click en API:exam-api...(<API-ID>)
  - En Default endpoint está la URL de la API

- Editar index.html del frontend y cambiar el `API_BASE` a la URL de API Gateway. 
  
Por ejemplo:
```javascript
const API_BASE = 'https://<api-id>.execute-api.us-east-1.amazonaws.com/api/tutorials';
```

Is a S3 y volver a subir el archivo index.html editado al bucket.

Probar el Frontend abriendo la URL del Bucket website endpoing con curl o explorador:

`http://<TU_BUCKET>.s3-website-us-east-1.amazonaws.com`

## Anexo: Pruebas de la API con curl

Una vez desplegada la API y obtenida la URL pública del Application Load Balancer (ALB) o del API Gateway, puede probar los endpoints principales usando `curl` desde cualquier terminal.

> Reemplace `<API_URL>` por la URL de su ALB o API Gateway, por ejemplo:  
> `http://exam-api-lb-xxxxxxxxxx.us-east-1.elb.amazonaws.com/api/tutorials`  
> o  
> `https://<api-id>.execute-api.us-east-1.amazonaws.com/api/tutorials`

### Listar todos los tutorials

```bash
curl <API_URL>/api/tutorials
```

### Crear un tutorial

```bash
curl -X POST <API_URL>/api/tutorials \
  -H "Content-Type: application/json" \
  -d '{"title":"Desde curl","description":"Agregado desde curl","published":false}'
```

### Obtener un tutorial por ID

```bash
curl <API_URL>/api/tutorials/1
```

### Actualizar un tutorial

```bash
curl -X PUT <API_URL>/api/tutorials/1 \
  -H "Content-Type: application/json" \
  -d '{"title":"Actualizado desde curl","description":"Editado via curl","published":true}'
```

### Eliminar un tutorial por ID

```bash
curl -X DELETE <API_URL>/api/tutorials/1
```

### Eliminar todos los tutorials

```bash
curl -X DELETE <API_URL>/api/tutorials
```