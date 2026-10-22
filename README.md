# Laboratorio 10: Primera aproximación a AWS

Este laboratorio tiene como propósito publicar un pequeño sitio estático mediante el uso de Cloud Computing. El sitio se entrega preparado y contiene una página HTML con una fotografía. El trabajo práctico consiste en configurar los servicios de AWS necesarios para que ambos archivos puedan consultarse desde un navegador.

Para realizar lo anterior se utilizarán dos formas de publicación. Primero se creará un servidor virtual mediante Amazon EC2 y luego se publicarán los mismos archivos mediante Amazon S3.

Las habilidades que se espera ejercitar en esta actividad son:

1. Configurar MFA para el usuario raíz y crear un usuario IAM administrador para el trabajo habitual
2. Crear una instancia EC2 y acceder mediante EC2 Instance Connect
3. Desplegar un servidor HTTP y configurar su acceso
4. Publicar un sitio estático en S3
5. Distinguir entre el acceso público a un objeto y el acceso temporal mediante una URL prefirmada
6. Eliminar los recursos utilizados

## Entorno de trabajo

Para desarrollar este laboratorio se requiere:

- Una cuenta AWS recién creada y activada, con acceso al correo electrónico asociado y a la contraseña del usuario raíz (*root*)
- Un teléfono disponible para configurar una aplicación autenticadora y generar códigos MFA durante la actividad
- Un computador con navegador y acceso al repositorio del laboratorio

La actividad se realiza desde la consola web de AWS, por lo que puede utilizarse cualquier computador con navegador, independientemente de su sistema operativo. Se utilizará la consola en inglés y la región **South America (São Paulo), `sa-east-1`**.

Algunos de los recursos que se utilizarán pueden generar cargos y deben eliminarse al finalizar, siguiendo las instrucciones de la sección de limpieza.

## Preparación

La carpeta `website/` del repositorio contiene:

- **`index.html`**: página estática sobre la Universidad de los Andes, con estilos incorporados y sin dependencias externas
- **`fic.webp`**: fotografía del edificio de la Facultad de Ingeniería y Ciencias utilizada por la página
- **`private.txt`**: archivo de prueba para comprobar los permisos de acceso a objetos de S3

No se requiere descargar el repositorio completo. En EC2, los archivos se descargarán directamente desde GitHub al servidor. Para S3, se descargarán individualmente al computador y luego se cargarán mediante la consola de AWS.

## Procedimiento

### 1. Configurar la seguridad de la cuenta AWS

**AWS Identity and Access Management (IAM)** es el servicio que permite administrar los usuarios y sus permisos de acceso a AWS. Los permisos se definen mediante políticas, que indican qué acciones se permiten o deniegan sobre los recursos de la cuenta.

El usuario raíz es el que se utiliza al crear la cuenta. En esta etapa se utilizará para configurar MFA y crear un usuario IAM con permisos de administración. El resto del laboratorio se realizará con el usuario IAM. Ambos usuarios deben contar con autenticación multifactor (MFA), que incorpora una verificación adicional a la contraseña.

Estas medidas permiten separar el acceso principal de la cuenta del trabajo habitual y reducir el riesgo de accesos no autorizados:

- **Reservar el usuario raíz para tareas específicas**: este usuario tiene acceso completo a la cuenta y puede realizar operaciones que no están disponibles para otros usuarios. Evitar su uso cotidiano reduce la exposición de sus credenciales
- **Utilizar un usuario IAM para el trabajo habitual**: permite administrar sus permisos y revocar su acceso sin utilizar ni compartir las credenciales del usuario raíz. Además, las acciones realizadas quedan asociadas al usuario correspondiente
- **Configurar MFA en ambos usuarios**: exige una verificación adicional al iniciar sesión. Si se obtiene la contraseña, esta por sí sola no basta para completar el ingreso a la consola

La buena práctica asociada con esto consiste en separar los accesos y protegerlos con MFA. En este laboratorio se utilizará `AdministratorAccess` para facilitar la configuración inicial. En un entorno de trabajo, los permisos deben limitarse a las tareas que cada usuario necesita realizar.

#### 1.1 Configurar MFA para el usuario raíz

1. Abrir la consola de AWS, seleccionar **Root user** e ingresar con el correo y la contraseña utilizados al crear la cuenta
2. En el menú de cuenta, abrir **Security credentials**
3. En **Multi-factor authentication (MFA)**, seleccionar **Assign MFA device**
4. Asignar un nombre, como `root-mfa`, seleccionar **Authenticator app** y continuar
5. En el teléfono, utilizar una aplicación autenticadora compatible con TOTP; si no se dispone de una, instalarla
6. Seleccionar **Show QR code** y escanearlo desde la aplicación; asignar al registro un nombre que permita reconocer al usuario raíz
7. Ingresar un código en **MFA code 1**, esperar que la aplicación genere el siguiente e ingresarlo en **MFA code 2**; seleccionar **Add MFA**
8. Comprobar que el dispositivo aparezca asociado al usuario raíz

Si AWS solicita configurar MFA al iniciar sesión por primera vez, completar ese procedimiento y verificar el dispositivo en **Security credentials**. No se debe repetir la configuración. El código QR y la clave de configuración son información confidencial y no deben incluirse en capturas, repositorios ni informes.

#### 1.2 Crear un usuario IAM para el trabajo habitual

1. Mantener abierta la sesión del usuario raíz y acceder a **IAM → Users → Create user**
2. Ingresar un nombre de usuario propio, por ejemplo `nombre-admin`
3. Habilitar **Provide user access to the AWS Management Console** y seleccionar **I want to create an IAM user** si aparece esta opción
4. Configurar una contraseña distinta de la del usuario raíz y mantener habilitada la opción que exige cambiarla al iniciar sesión por primera vez
5. En **Set permissions**, seleccionar **Attach policies directly**
6. Buscar y seleccionar la política administrada por AWS **AdministratorAccess**
7. Revisar los datos y seleccionar **Create user**
8. Guardar de forma segura la contraseña inicial, el nombre del usuario y la **Console sign-in URL** de la cuenta

La política `AdministratorAccess` otorga permisos de administración sobre los servicios y recursos de AWS, aunque algunas operaciones siguen siendo exclusivas del usuario raíz.

#### 1.3 Iniciar sesión con el usuario IAM

1. Cerrar la sesión del usuario raíz mediante **Sign out**
2. Abrir la **Console sign-in URL** guardada; esta dirección completa automáticamente el ID de la cuenta AWS a la que se iniciará sesión
3. Ingresar con el nombre del usuario IAM y su contraseña inicial
4. Establecer la nueva contraseña cuando AWS lo solicite
5. Confirmar en el menú de cuenta que la sesión corresponde al usuario IAM creado y no al usuario raíz

Para iniciar sesión como usuario IAM se utiliza el nombre del usuario, no el correo electrónico del usuario raíz. A partir de este paso, se debe trabajar con el usuario IAM.

#### 1.4 Configurar y comprobar MFA para el usuario IAM

1. Desde la sesión del usuario IAM, abrir **IAM → Users** y seleccionar el usuario creado
2. En **Security credentials → Multi-factor authentication (MFA)**, seleccionar **Assign MFA device**
3. Asignar un nombre, como `usuario-mfa`, y seleccionar **Authenticator app**
4. Escanear el nuevo código QR en la aplicación autenticadora y asignar al registro el nombre del usuario IAM
5. Ingresar dos códigos consecutivos y seleccionar **Add MFA**
6. Cerrar la sesión e iniciar una nueva con el usuario IAM; comprobar que AWS solicita el código MFA y permite acceder después de ingresarlo

Se puede utilizar la misma aplicación autenticadora para ambos usuarios, pero cada uno debe tener su propio registro. Antes de continuar, comprobar que ambos tengan MFA configurado y que la sesión activa corresponda al usuario IAM.

### 2. Publicar el sitio en EC2

**Amazon Elastic Compute Cloud (EC2)** permite crear servidores virtuales, denominados *instancias*. En una instancia se puede instalar un sistema operativo y ejecutar programas. En este laboratorio se utilizará Ubuntu y se instalará Apache para responder las solicitudes HTTP del navegador.

Para comenzar, buscar **EC2** en la consola de AWS y abrir el servicio. En el selector de región de la barra superior, seleccionar **South America (São Paulo), `sa-east-1`**.

#### 2.1 Crear el security group

Un **security group** actúa como un firewall virtual: sus reglas determinan qué tráfico puede ingresar o salir de la instancia. Se habilitará SSH para administrar el servidor desde un terminal.

1. Abrir **EC2 → Network & Security → Security Groups → Create security group**
2. Asignar el nombre `icc4203-lab10-web-sg` y una descripción, manteniendo las demás opciones por omisión
3. Agregar una regla de entrada **SSH**, puerto **22**, con **Source: Anywhere-IPv4 (`0.0.0.0/0`)**
4. Mantener la regla de salida por omisión y crear el grupo

Esta regla permite conexiones SSH desde cualquier dirección IPv4, aunque sigue siendo necesario autenticarse para acceder al servidor. Se utilizará esta configuración durante el laboratorio.

#### 2.2 Lanzar la instancia

La instancia se crea a partir de una imagen que contiene el sistema operativo. Su disco se proporciona mediante **Amazon Elastic Block Store (EBS)**, un servicio de almacenamiento que permite conservar los archivos de la instancia. En el volumen EBS se instalarán Ubuntu, Apache y los archivos del sitio.

En **Instances → Launch instances**, configurar:

| Campo | Valor |
| --- | --- |
| Name | `icc4203-lab10-web` |
| Application and OS Images | Ubuntu Server 26.04 LTS, imagen oficial de Canonical |
| Architecture | 64-bit (x86) |
| Instance type | `t3.small` |
| Key pair | Proceed without a key pair |
| Auto-assign public IP | Enable |
| Firewall | Select existing security group: `icc4203-lab10-web-sg` |
| Storage | 8 GiB, gp3, con Delete on termination habilitado |

Mantener las demás opciones por omisión. No se requiere asignar un perfil IAM a la instancia ni completar **User data**. Seleccionar **Launch instance** y esperar que la instancia aparezca en estado **Running** y que todas las verificaciones de **Status checks** se completen correctamente.

#### 2.3 Instalar Apache y cargar la página

**EC2 Instance Connect** permite establecer una conexión SSH con la instancia desde la consola web de AWS. El terminal se abre en el navegador, pero los comandos se ejecutan en el servidor, no en el computador del estudiante. **Apache** es el servidor HTTP que entregará los archivos del sitio; no es un servicio de AWS, sino un programa instalado en la instancia.

1. Seleccionar la instancia y luego **Connect**
2. En **EC2 Instance Connect**, confirmar la conexión mediante la IPv4 pública y el usuario `ubuntu`
3. Seleccionar **Connect** para abrir el terminal

Una vez conectado a la instancia se debe actualizar la lista de paquetes disponibles, instalar Apache y habilitar su inicio automático:

```bash
sudo apt update
sudo apt install -y apache2 wget
sudo systemctl enable --now apache2
```

La carpeta `/var/www/html/` contiene los archivos que Apache entrega por HTTP. Descargar el HTML y la imagen desde el repositorio público directamente a esa carpeta:

```bash
sudo wget -O /var/www/html/index.html https://raw.githubusercontent.com/ICC4203-202620/lab10/main/website/index.html
sudo wget -O /var/www/html/fic.webp https://raw.githubusercontent.com/ICC4203-202620/lab10/main/website/fic.webp
```

Las direcciones de `raw.githubusercontent.com` entregan el contenido de los archivos, no la página de GitHub que los muestra. La opción `-O` indica dónde se guardará cada archivo; así la primera descarga reemplaza la página original de Apache.

Comprobar el servicio y la respuesta local:

```bash
systemctl is-active apache2
curl http://127.0.0.1/
```

El primer comando debe indicar `active`. El segundo debe mostrar el código HTML de la página que está desplegada.

#### 2.4 Habilitar el acceso HTTP

1. Volver a la pestaña de la consola de AWS, sin cerrar el terminal de EC2
2. Abrir **EC2** y seleccionar **Instances** en el menú lateral
3. Buscar `icc4203-lab10-web` en la columna **Name** y seleccionar su casilla; comprobar que su estado sea **Running**
4. En el panel inferior, abrir la pestaña **Details**, localizar **Public IPv4 address** y copiar la dirección indicada
5. Abrir una nueva pestaña del navegador e ingresar `http://IP_PUBLICA`, reemplazando `IP_PUBLICA` por la dirección copiada
6. Comprobar que la conexión no se completa, aunque Apache respondió desde el terminal que estaba activo
7. Volver a la consola de EC2 y, con la instancia seleccionada, abrir la pestaña **Security** del panel inferior
8. En **Security groups**, seleccionar el enlace del grupo `icc4203-lab10-web-sg`
9. En la página del grupo, abrir **Inbound rules** y seleccionar **Edit inbound rules → Add rule**
10. Agregar **Type: HTTP**, puerto **80**, **Source: Anywhere-IPv4 (`0.0.0.0/0`)** y seleccionar **Save rules**
11. Volver a la pestaña donde se intentó abrir el sitio y recargar; comprobar que aparecen el título **Universidad de los Andes** y la fotografía

Lo que se acaba de ver es que el servidor estaba funcionando pero no tenía autorizadas conexiones por el puerto 80. La regla agregada permite que ahora también pueda recibir solicitudes HTTP desde Internet.

### 3. Publicar el mismo sitio en S3

**Amazon Simple Storage Service (S3)** permite almacenar archivos como *objetos*, agrupados en contenedores llamados *buckets*. Cada objeto tiene un nombre que lo identifica dentro del bucket. En esta actividad se utilizará S3 para entregar el HTML y la imagen directamente al navegador, sin necesitar instalar un servidor HTTP. S3 almacena y entrega los archivos, pero no ejecuta código.

#### 3.1 Crear el bucket

El nombre del bucket forma parte de su dirección de acceso. No puede coincidir con el de otro bucket, aunque pertenezca a otra cuenta de AWS. Por eso, cada estudiante debe agregar un identificador propio al nombre propuesto. Si AWS indica que el nombre ya está ocupado, se debe cambiar ese identificador.

Para crear el bucket:

1. En el buscador superior de la consola de AWS, buscar **S3** y abrir el servicio
2. En el menú lateral, seleccionar **General purpose buckets**; si el menú está cerrado, desplegarlo
3. Seleccionar **Create bucket**
4. En **Bucket name**, ingresar un nombre de la forma `icc4203-lab10-<token>`, reemplazando `<token>` por una combinación aleatoria de seis letras minúsculas y números
5. Confirmar que **AWS Region** corresponda a **South America (São Paulo), `sa-east-1`**
6. En **Object Ownership**, mantener **ACLs disabled**; en **Block Public Access settings for this bucket**, mantener **Block all public access** habilitado
7. Mantener **Bucket Versioning: Disable** y el cifrado **SSE-S3**, sin modificar las demás opciones
8. Seleccionar **Create bucket** al final del formulario y comprobar que AWS informe que fue creado correctamente

#### 3.2 Cargar los archivos

1. Descargar individualmente [index.html](https://raw.githubusercontent.com/ICC4203-202620/lab10/main/website/index.html), [fic.webp](https://raw.githubusercontent.com/ICC4203-202620/lab10/main/website/fic.webp) y [private.txt](https://raw.githubusercontent.com/ICC4203-202620/lab10/main/website/private.txt): hacer clic derecho sobre cada enlace y seleccionar la opción del navegador para guardar el enlace como archivo
2. Guardar los tres archivos en el computador, conservando sus nombres y extensiones, y comprobar dónde quedaron guardados
3. Volver a la consola de S3 y seleccionar **General purpose buckets** en el menú lateral
4. Buscar el bucket recién creado y seleccionar su nombre para abrirlo
5. En la pestaña **Objects**, seleccionar **Upload**
6. Seleccionar **Add files**, ir a la ubicación donde se guardaron los archivos en el computador y agregar `index.html`, `fic.webp` y `private.txt`
7. Comprobar que la lista de archivos por cargar contenga `index.html`, `fic.webp` y `private.txt`; luego seleccionar **Upload** al final de la página
8. Esperar que AWS informe que la carga finalizó correctamente y seleccionar **Close** para volver a la lista de objetos del bucket

Los tres archivos deben quedar directamente en la raíz del bucket, sin carpetas intermedias. Los archivos no deben ser modificados.

#### 3.3 Observar el estado privado inicial

1. En la consola de AWS, abrir **S3** y seleccionar **General purpose buckets** en el menú lateral
2. Buscar el bucket donde se cargaron los archivos y seleccionar su nombre
3. En la pestaña **Objects**, comprobar que aparezcan `index.html`, `fic.webp` y `private.txt`
4. Seleccionar el nombre `index.html` para abrir la información del objeto
5. En **Properties**, localizar **Object URL**, copiar la dirección e identificar sus componentes
6. Abrir una ventana de incógnito y pegar la dirección en la barra del navegador
7. Volver a la consola de S3 y seleccionar el nombre del bucket en la ruta de navegación superior para regresar a la lista de objetos
8. Seleccionar `private.txt`, copiar su **Object URL** y abrirla también en la ventana de incógnito

Ambas solicitudes deben responder **Access Denied**. La existencia de una URL no concede acceso anónimo al objeto.

#### 3.4 Compartir temporalmente el objeto privado

Una **URL prefirmada** permite compartir temporalmente el acceso a un objeto utilizando los permisos de quien la genera. Quien recibe la URL puede realizar la operación autorizada durante su vigencia, sin iniciar sesión en AWS ni hacer público el objeto.

1. Volver a la consola de S3 y seleccionar el nombre del bucket en la ruta de navegación superior para regresar a la lista de objetos
2. En **Objects**, marcar la casilla de `private.txt`, sin abrir su nombre, y seleccionar **Object actions → Share with a presigned URL**
3. Seleccionar una duración de 1 minuto y crear la URL
4. Abrirla en una ventana de incógnito y comprobar que permite leer el objeto
5. En la página del objeto, hacer clic derecho y seleccionar **Inspeccionar** para abrir las herramientas de desarrollo del navegador
6. Abrir la pestaña **Red** (**Network**) y marcar **Deshabilitar caché** (**Disable cache**); mantener abiertas las herramientas durante la comprobación
7. Una vez transcurrido un minuto desde la creación de la URL, recargar la página y comprobar que el acceso se rechaza

No se debe incluir la URL completa en capturas o informes, ya que permite acceder al objeto mientras esté vigente.

#### 3.5 Habilitar el website endpoint

El **website endpoint** es la dirección que S3 proporciona para acceder al bucket como sitio web. A diferencia de la URL de un objeto específico, permite abrir la dirección del sitio y recibir el documento inicial configurado, en este caso `index.html`.

1. Volver a la consola de S3, seleccionar **General purpose buckets** en el menú lateral y abrir el bucket del laboratorio seleccionando su nombre
2. Abrir la pestaña **Properties**, buscar la sección **Static website hosting** y seleccionar **Edit**
3. En **Static website hosting**, seleccionar **Enable** y, en **Hosting type**, seleccionar **Host a static website**
4. En **Index document**, escribir `index.html`; dejar vacío **Error document** y seleccionar **Save changes**
5. En la pestaña **Properties**, volver a la sección **Static website hosting**, copiar el **Bucket website endpoint** y comparar su estructura con la Object URL de `index.html`
6. Abrir el endpoint en una nueva pestaña del navegador

La solicitud todavía debe recibir una respuesta **403 Forbidden**, porque habilitar el sitio no otorga permisos de lectura. El website endpoint utiliza HTTP.

#### 3.6 Publicar la página y su imagen

Una **bucket policy** es una política escrita en JSON que establece quién puede realizar determinadas acciones sobre el bucket o sus objetos. En este caso se autorizará la lectura pública de los dos archivos del sitio. Las **ACL** son otro mecanismo de permisos de S3; se mantendrán deshabilitadas para controlar el acceso mediante la política.

1. Volver a la pestaña de la consola de S3 donde se encuentra abierto el bucket y seleccionar **Permissions**
2. Buscar la sección **Block public access (bucket settings)** y seleccionar **Edit**
3. Desmarcar las dos opciones relativas a políticas públicas (`BlockPublicPolicy` y `RestrictPublicBuckets`), manteniendo activadas las dos relativas a ACL
4. Seleccionar **Save changes** y completar la confirmación solicitada por AWS
5. En la pestaña **Permissions**, buscar la sección **Bucket policy** y seleccionar **Edit**
6. Pegar la política siguiente en el editor, reemplazando ambas apariciones de `BUCKET_NAME` por el nombre del bucket:

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "PublicReadOnlyForWebsite",
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:GetObject",
         "Resource": [
           "arn:aws:s3:::BUCKET_NAME/index.html",
           "arn:aws:s3:::BUCKET_NAME/fic.webp"
         ]
       }
     ]
   }
   ```

7. Seleccionar **Save changes** y comprobar que AWS informe que la política se guardó correctamente

Comprobar en una ventana de incógnito:

- El **Bucket website endpoint** muestra la página de la Universidad de los Andes con su fotografía
- La **Object URL** de `index.html` también permite leer la página
- La **Object URL** de `fic.webp` permite ver la fotografía
- La **Object URL** de `private.txt` continúa respondiendo **Access Denied**

Las ACL permanecen deshabilitadas. La política autoriza únicamente la lectura de `index.html` y `fic.webp`, no la escritura ni la lectura de los demás objetos. Permitir el acceso al HTML no permite acceder automáticamente a la imagen. A diferencia de EC2, en S3 no fue necesario instalar Apache ni configurar una instancia: el servicio entrega directamente los archivos estáticos.

Comparar el sitio publicado en S3 con `http://IP_PUBLICA`, utilizando la dirección de la instancia obtenida en el paso 2.4. Ambos deben mostrar la misma página y fotografía. Como el HTML utiliza la ruta relativa `fic.webp`, cada sitio entrega su propia copia de la imagen.

## Limpieza de recursos

Realizar la limpieza al finalizar las comprobaciones del laboratorio. No se requiere mantener recursos activos para el laboratorio siguiente.

El usuario IAM y los dispositivos MFA se deben conservar para las siguientes actividades. No se debe desactivar MFA como parte de la limpieza.

Eliminar únicamente los recursos creados durante esta actividad.

1. En **S3 → General purpose buckets**, seleccionar el bucket de la actividad
2. Utilizar **Empty**, comprobar el nombre del bucket y confirmar; el bucket debe estar vacío antes de eliminarse
3. Volver a la lista, seleccionar **Delete** y confirmar el nombre
4. En el buscador superior de la consola de AWS, buscar **EC2** y abrir el servicio; confirmar que la región siga siendo **South America (São Paulo), `sa-east-1`**
5. Seleccionar **Instances** en el menú lateral y marcar la casilla de `icc4203-lab10-web`
6. En el panel inferior, abrir la pestaña **Storage** y, en **Block devices**, registrar el **Volume ID** del disco de la instancia; este identificador comienza con `vol-` y permitirá comprobar su eliminación
7. Con la instancia seleccionada, abrir **Instance state → Terminate (delete) instance** y confirmar mediante **Terminate (delete)**
8. Esperar que la instancia figure en estado **Terminated**; actualizar la lista si es necesario
9. En el menú lateral, ir a **Elastic Block Store → Volumes** y buscar el **Volume ID** registrado; comprobar que ya no aparezca en la lista
10. Si el volumen permanece, esperar que su estado sea **Available**, marcar su casilla y seleccionar **Actions → Delete volume**; comprobar que el identificador corresponda al registrado y confirmar la eliminación
11. En el menú lateral, ir a **Network & Security → Security Groups**, buscar `icc4203-lab10-web-sg` y marcar su casilla
12. Seleccionar **Actions → Delete security groups**, comprobar el nombre del grupo y confirmar; si AWS indica que todavía está en uso, esperar a que termine la eliminación de la instancia y volver a intentarlo
13. Confirmar que el bucket fue eliminado, la instancia está terminada y no quedan el volumen EBS ni el security group de la actividad; luego, cerrar la sesión de la consola
