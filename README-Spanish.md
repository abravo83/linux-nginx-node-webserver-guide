# INSTRUCCIONES PARA LA CONFIGURACIÓN DE UN SERVIDOR LINUX COMO SERVIDOR WEB NODE CON VARIOS SERVIDORES EXPRESS

## El problema

Cuando necesitamos ejecutar múltiples aplicaciones web Node.js/Express en un mismo servidor, nos encontramos con una limitación importante: los puertos estándar web.

Por convención, los navegadores web se conectan al puerto 80 para HTTP y al puerto 443 para HTTPS. Si intentamos ejecutar varias aplicaciones Express simultáneamente (por ejemplo, una aplicación Angular/React y una API), todas necesitarían escuchar estos mismos puertos.

<br/>

![Figura 1: Un servidor con varios node escuchando al puerto 443](img/fig-1.png)

Esto genera un conflicto: ¿Qué aplicación debería responder a las peticiones entrantes?

Aunque configuremos diferentes dominios (por ejemplo, `www.midominioweb.com` para la aplicación frontend y `www.midominioapi.com` para la API), el problema persiste. Los dominios solo determinan a qué dirección IP se dirigen las peticiones, pero no especifican el puerto. Por lo tanto, no podemos asignar diferentes dominios a diferentes aplicaciones Express que escuchan en el mismo puerto.

## Las soluciones

- La primera solución sería contratar un servidor independiente para cada aplicación web. Sin embargo, esto resulta costoso e ineficiente desde el punto de vista de recursos y gestión.

- La segunda solución consiste en configurar las aplicaciones para que escuchen en diferentes puertos. Por ejemplo, mantener la aplicación principal React/Angular en el puerto 443 (accesible mediante `https://www.midominioweb.com`), mientras que la API escucharía en otro puerto (accesible mediante `https://www.midominioweb.com:3030`). Cuando no se especifica un puerto en la URL, el navegador utiliza por defecto el puerto 443 para HTTPS y el 80 para HTTP.

  Aunque esta solución podría funcionar inicialmente, presenta dos problemas importantes:

  1. No es profesional requerir que los usuarios finales incluyan números de puerto específicos en las URLs
  2. Complica significativamente la gestión y renovación de certificados SSL necesarios para HTTPS

- La solución recomendada es utilizar un servidor web que actúe como **Reverse Proxy**. Este servidor será el único que escuche en los puertos 443 y 80, y se encargará de reenviar las peticiones a diferentes servidores web que corren localmente en otros puertos (por ejemplo, 3010, 3011...). Estos servidores locales permanecen seguros al no estar expuestos directamente a Internet, ya que sus puertos no están abiertos en el firewall.

  Al combinar esta configuración con subdominios, podemos ofrecer múltiples servicios web a través de un único dominio principal:

  - `https://www.midominioweb.com` (frontend principal)
  - `https://api.midominioweb.com` (API)
  - `https://mywebapp2.midominioweb.com` (otra aplicación)

  El reverse proxy analiza el dominio de cada petición entrante y la redirige al servidor local correspondiente.

  Esta arquitectura es una solución probada que funciona perfectamente con gestores de certificados SSL como _certbot_, permitiendo la automatización completa del proceso de emisión y renovación de certificados.

  <br/>

![Figura 1: Un servidor escuchando con un reverse proxy al puerto 443](img/fig-2.png)

<br/>

## El proceso de configuración

El proceso para configurar un servidor web desde cero va más allá de la configuración de un reverse proxy y múltiples servidores node corriendo de forma local. Pero tampoco mucho más allá. Vamos a cubrir el proceso completo ya que todos nos encontramos con tener que realizar el proceso completo y se agradece tener los pasos en una única guía.

Este proceso pretende configurar un servidor para que sea seguro y funcional para producción, por lo que te va a parecer que algunos pasos sean innecesarios, pero si se hacen es para mejorar la seguridad.

Es posible que te preguntes qué sistema operativo tener en el servidor. Mi recomendación es Ubuntu Server o Rocky Linux (Por su uso extendido) en su última versión LTS y SIN entorno de escritorio para no gastar recursos innecesariamente ya que las operaciones las vamos a realizar por la terminal.

Antes de adquirir o contratar el servidor, ya sea un VPS o una instancia cloud de un servidor, es MUY recomendable que estés preparado para seguir al menos los primeros pasos previos a la instalación de servicios web ya que una vez se levante el servidor puedes estar seguro que algún bot automático va a encontrar tu web y va a intentar loguearse por fuerza bruta usando usuarios por defecto (como root) y contraseñas comunes.

Es por esos que nuestros primeros pasos van encaminados a esto.

1. Encontrar y analizar los datos de acceso que nos han dado para acceder a nuestro servidor web.
2. Modificar los datos de acceso y la posibilidad de acceso desde cuentas conocidas (p.e. como 'root', 'admin', 'administrator'...)
3. Comprobar y Levantar el firewall con las excepciones necesarias para que podamos seguir conectándonos al servidor.
4. Actualizar --Hasta aquí las operaciones a realizar de forma urgente nada más levantar el servidor--
5. Instalar Node
6. Instalar NGINX (Nuestro Reverse Proxy)
7. Instalar MySQL o MongoDB o el servidor DB que vayamos a usar (Si los necesitamos)
8. Crear un usuario SIN PERMISOS SUDO que será el propietario de las carpetas donde estén contenidos nuestros servidores node.
9. Crear scripts BASH con los comandos para levantar los servidores node.
10. Crear servicios que ejecuten dichos scripts para que nuestros servidores node funcionen como servicios del servidor.
11. Configurar nuestros dominios y subdominios para que dirigan a nuestro servidor.
12. Configurar nuestro reverse proxy para que dirija las peticiones de cada dominio o subdominio a sus servidores locales node correspondientes.
13. Configurar certbot para que expida y además se quede encargado de renovar los certificados SSL cuando corresponda.
14. Comprobar todo (Servicios enabled, Reinicio del servidor con todo levantado automáticamente) y salir a pisar hierba.

## 1. ENCONTRAR Y ANALIZAR LOS DATOS DE ACCESO QUE NOS HAN DADO PARA ACCEDER A NUESTRO SERVIDOR WEB

Una vez hemos terminado el proceso de contratación al cabo de unos poco minutos nuestro servidor estará disponible para su acceso. Los datos de acceso nos los pueden mandar por email, u obtenerlos a través del panel de control de nuestro proovedor.

Debemos analizar cual es el nombre de usuario que nos han dado. Si por ejemplo es `root` tenemos un problema en cuanto a que `root` es un superusuario que está presente en todas las máquinas Linux, pero al que por seguridad muchas veces no se le da acceso remoto. Sin embargo, si nos han dado datos de acceso con `root` significa que este usuario SI tiene acceso remoto, y esto es problemático en cuanto a que los bots automáticos que buscan acceder por fuerza bruta a los servidores están constatemente buscando combinaciones de `root` con contraseñas, algunas sencillas como `1234` o `password`, otras un poco más complicadas, pero de forma constante.

Para poder acceder por fuerza bruta necesitan dar con tu usuario y contraseña. Si de alguna forma conocen tu usuario de acceso ya tienen la mitad del trabajo hecho. Puede que por ejemplo, conozcan el rando de IPs que tiene tu proovedor y que el acceso lo da con el usuario root, u otra circunstancia que les hace saber tu nombre de usuario. Es por ello de vital importancia que si tu proovedor te ha dado un nombre de usuario demasiado común, como `root`, `admin` o el nombre del provedor, como puede ser `arsys` o `hostinger` (No digo que esto proveedores usen esos nombre de usuario, son sólo ejemplos) es de vital importancia que o bien crees un nuevo usuario y quites acceso remoto al otro usuario.

Luego hay que analizar la constraseña, aunque esta hay que modificarla siempre. En realidad tanto usuario como contraseña es muy recomendable que se modifiquen. La diferencia o necesidad del análisis es la urgencia con la que debes realizar estas acciones. Con usuario `root` o `admin` o `administrator` el cambio es urgentísimo. Con usuario con un nombre común como el nombre del proovedor del servidor el cambio es urgente. Con un nombre de usuario con aspecto de ser aleatorio como `a12hjdMa` el cambio no sería tan urgente, pero igualmente recomendable.

Es posible que nuestro proovedor nos permita elegir el formato de acceso, ya que no óolo podemos acceder mediante usuario y contraseña. También podemos acceder mediante usuario y clave compartida, que deberíamos instalar en el ordenador. Como este caso es menos frecuente lo vamos a dejar aparte, aunque es un método más seguro pero con en inconveniente de tener que instalar la clave en todos los ordenadores desde los que se puede realizar el acceso.

Otro caso que se nos podría dar es que nos hayan dado un puerto específico para conectarnos de forma remota diferente del puerto por defecto. En ese caso, enhorabuena, tu servidor es bastante más seguro sólo por esto, aunque ahora el puerto es otro parámetro a recordar junto al usuario y contraseña. Pero primero vamos a explicar un poco cómo nos conectamos de forma remota para el que no esté familiarizado con SSH.

### La conexión remota mediante el servicio SSH

Para conectarnos de forma remota a nuestro servidor el proovedor habrá levantado el servidor con el servicio SSH habilitado. SSH significa _Secure Shell_ y nos permite desde cualquier terminal, como `Git Bash`, `CMD`, `PowerShell` o `Terminal Mac`, aunque también podemos usar un programa como `PuTTy` para realizar la conexión y tener la terminal. En todo caso, para poder conectarte vía SSH necesitas abrir un programa de terminal como los anteriormente mencionados.

Este servicio tiene por defecto un puerto de conexión: **El puerto 22**. Si nuestro proovedor no dice nada en los datos de acceso significa que usamos dicho puerto.
En ese caso podríamos conectarnos a nuestro servidor usando alguno de los siguiente nombres de usuario y direcciones fictícias

```Bash
ssh <tu nombre de usuario>@<la direccion de tu servidor>
```

por ejemplo

```Bash
ssh root@122.122.122.122
```

o usando un nombre de dominio:

```Bash
ssh root@sajkjsd-122.cloud.hostinger.com
```

Si la dirección es correcta inmediatamente nos solicitará la contraseña. CUIDADO ya que al escribir la contraseña NO SE MOSTRARÁN NI SIQUIERA CARACTERES OCULTOS para mejorar la seguridad en caso de que alguien pudiera ver tu pantalla que no tuviera ni siquiera el dato de la longitud de tu contraseña. Eso hace más fácil poder equivocarnos, así que se cuidadoso al introducir la contraseña, ya que no vas a poder ver nada de ella para saber si has introducido un caracter de más o de menos.

**Si nuestro proovedor nos ha dado un puerto específico distinto del 22 para SSH**

Entonces la conexión será así, por ejemplo, al puerto 2244:

```Bash
ssh -p 2244 root@122.122.122.122
```

## 2. MODIFICAR LOS DATOS DE ACCESO

Para mejorar la seguridad de nuestro servidor, seguiremos estos pasos:

#### 2.1 Crear un nuevo usuario con privilegios de administrador (Sudo)

Primero, crearemos un nuevo usuario con privilegios _sudo_. Para poder crear un usuario _sudo_ nuestro usuario actual debe tener a su vez privilegios _sudo_. Por ello debemos añadir delante del comando a realizar la palabra `sudo`. Es posible que nos solicite contraseña `sudo` al ir a realizar la acción. Es la misma contraseña del usuario que usaste para el acceso. Como excepción, el usuario `root` no necesita añadir el comando sudo.

```bash
# Crear nuevo usuario
sudo adduser tunuevousuario

# Añadir al grupo sudo
sudo usermod -aG sudo tunuevousuario
```

LLegado este punto es recomendable probar si tienes acceso SSH mediante este nuevo usuario. O bien cierra la terminal actual y abre una nuevo, o usa el comando `exit` y luego vuelve a usar el comando

#### 2.2 Configurar SSH para mayor seguridad

Editaremos el archivo de configuración SSH usando el editor Nano:

```bash
sudo nano /etc/ssh/sshd_config
```

Realizaremos los siguientes cambios en dicho archivo de texto. Busca la línea que ya hay escrita con el nombre del parámetro a configurar y modifícala. NO AGREGUES UNA NUEVA LÍNEA CON LA INSTRUCCIÓN. Debes modificar la línea existente.

```text:/etc/ssh/sshd_config
# Deshabilitar acceso root
PermitRootLogin no

# Deshabilitar autenticación por contraseña (opcional, solo si usas claves SSH - SI NO ES IMPORTANTE DEJARLO EN yes)
PasswordAuthentication yes

# Cambiar puerto SSH (opcional pero recomendado - MUY IMPORTANTE - El número es a modo de ejemplo.
# Debes buscar tu propio puerto. Simplemente asegúrate que está disponible -
# consulta el listado del enlace)
Port 2244
```

A la hora de cambiar el puerto SSH recuerda 2 cosas: Anotar bien el puerto. No usuar un puerto que esté usando otro servicio.

[Te dejo un listado con los puertos más comunes para que evites usarlos si te decides cambiar el puerto SSH](https://www.stationx.net/common-ports-cheat-sheet/)

#### 2.3 Reiniciar el servicio SSH

```bash
sudo systemctl restart sshd
```

#### 2.4 Verificar acceso con el nuevo usuario

Antes de cerrar la sesión actual, abre una nueva terminal y verifica que puedes acceder con el nuevo usuario:

```bash
ssh -p 2244 tunuevousuario@tu-ip-servidor
```

⚠️ **IMPORTANTE**:

- No cierres la sesión original hasta confirmar que puedes acceder con el nuevo usuario
- Guarda el nuevo puerto SSH si lo has cambiado
- Si usas un firewall, asegúrate de permitir el nuevo puerto SSH:

```bash
sudo ufw allow 2244/tcp
```

#### 2.5 Deshabilitar usuarios innecesarios (opcional, pero, de nuevo, recomendable)

Si quieres deshabilitar usuarios como 'admin' o 'administrator' para mejorar la seguridad y evitar que se puedan conectar via SSH:

```bash
sudo passwd -l nombreusuario
```

Esto bloqueará la cuenta sin eliminarla.

## 3. COMPROBAR Y LEVANTAR EL FIREWALL

El firewall es una parte esencial de la seguridad de nuestro servidor. En Ubuntu Server, utilizaremos UFW (Uncomplicated Firewall), que viene preinstalado pero generalmente desactivado.

### 3.1 Verificar el estado del firewall

```bash
sudo ufw status
```

Si aparece como "inactive", necesitaremos configurarlo.

### 3.2 Configuración básica del firewall

Antes de activar el firewall, debemos asegurarnos de permitir las conexiones SSH para no quedarnos fuera del servidor:

```bash
# Si usamos el puerto SSH por defecto (22)
sudo ufw allow 22/tcp

# Si hemos cambiado el puerto SSH (ejemplo: 2244)
sudo ufw allow 2244/tcp
```

### 3.3 Activar el firewall

```bash
sudo ufw enable
```

⚠️ **IMPORTANTE**: Asegúrate de haber permitido el acceso SSH antes de activar el firewall.

### 3.4 Reglas básicas recomendadas

```bash
# Denegar todo el tráfico entrante por defecto
sudo ufw default deny incoming

# Permitir todo el tráfico saliente por defecto
sudo ufw default allow outgoing

# Permitir HTTP
sudo ufw allow 80/tcp

# Permitir HTTPS
sudo ufw allow 443/tcp
```

### 3.5 Verificar las reglas configuradas

```bash
sudo ufw status verbose
```

### 3.6 Comandos útiles adicionales

**(NO PARA USAR AHORA, como referencia)**

```bash
# Eliminar una regla
sudo ufw delete allow 80/tcp

# Recargar las reglas
sudo ufw reload

# Desactivar el firewall (no recomendado en producción)
sudo ufw disable
```

### 3.7 Comprueba si tu proovedor no usa otro firewall configurable desde el panel de control

Es posible que tu proovedor use una capa adicional de firewall. Es buena práctica entonces usar ambos firewalls ya que si se cae el firewall externo no vas a tener ningún aviso y tu web quedaría desprotegida.

Si tu proveedor usa dicho firewall externo vas a tener que permitir el tráfico interno y externo del nuevo puerto SSH y de cualquier otro servicio que agregues. Si es un servidor web comprueba que los puertos 80 y 443 están abiertos para entrada y salida.

### 3.8 🛟 Si te has quedado bloqueado fuera del servidor 🛟

Puede que accidentalmente de hayas quedado bloqueado fuera del servidor. No desperes, a todos nos ha pasado, y no todo está perdido.

En este caso vas a tener que volver al panel de control de tu proveedor y comprobar si no tienen otro tipo de acceso que llamen "Acceso por Panel" o "Remoto". Esto nos permitiría acceder mediante un escritorio remoto como si estuviéramos físicamente delante del monitor del servidor, por lo que no importaría la configuración del firewall o de SSH. Eso si, debemos conocer nuestro usuario y contraseña. Desde aquí podríamos acceder y modificar las configuraciones del SSH o del firewall que nos han bloqueado acceso al servidor.

Si no hay dicho acceso... entonces, tendrás que usar la opción del panel de control que te permita reinstalar el servidor y comenzar el proceso de nuevo.

Si no hay nada de esto, ponte en contacto con el proveedor para que te vuelva a dejar el servidor reinstalado con las configuraciones por defecto.

### 3.9 ⚠️ **NOTAS IMPORTANTES**:

- Siempre verifica dos veces que has permitido el acceso SSH antes de activar el firewall

- Si necesitas acceder a otros servicios (como bases de datos), recuerda abrir los puertos correspondientes.

  Aunque **mi recomendación es que NO habras dichos puertos en el caso de Bases de Datos siempre que tu API esté alojada en el mismo servidor** que tu bbdd, ya que va a realizar una conexión local y no necesita exponerse a internet, que siempre es un riesgo si no es necesario.

- Es recomendable mantener el número de puertos abiertos al mínimo necesario

- Considera usar rangos de IPs permitidas para servicios críticos si es posible

## 4. ACTUALIZAR EL SISTEMA

Vamos a aplicar las actualizaciones pendientes por si hay algún parche de seguridad disponible que se deba aplicar. Para ello vamos a realizar `update` de `apt` (aptitude) que es el gestor de paquetes de `Ubuntu` y después `upgrade` para aplicar las actualizaciones. El gestor de paquetes de Rocky Linux es `dfn`, aunque aquí solo voy a presentar la instalación con `apt` no debería de diferenciarse mucho si se usa `dfn`.

```bash
sudo apt update && upgrade -y
```

Es posible y recomendable que tengamos que reiniciar el sistema

```bash
sudo reboot
```

## 5. INSTALAR NODE

Para instalar Node.js en nuestro servidor, tenemos varias opciones. La recomendada es usar el gestor de versiones NVM (Node Version Manager) que nos permitirá instalar y gestionar múltiples versiones de Node.js. Esto a su vez nos va a permitir poder levantar servidores con distintas versiones de Node dentro de un mismo servidor.

### 5.1 Instalar NVM

Primero, descargamos y ejecutamos el script bash de la instalación de NVM con la última versión, pero es recomendable que consultes el [repositorio de nvm](https://github.com/nvm-sh/nvm/) para averiguar cual es la última versión actual de nvm.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

Después de instalar reiniciamos la terminal o ejecutamos para que se refresque y poder tener disponible el comando `nvm`:

```bash
source ~/.bashrc
```

### 5.2 Verificar la instalación de NVM

```bash
nvm --version
```

### 5.3 Instalar Node.js

Para instalar la última versión LTS de node (recomendada para producción):

```bash
nvm install --lts
```

O si necesitas una versión específica:

```bash
nvm install 20.11.1  # Por ejemplo, para instalar la versión 20.11.1
```

O si quieres instalar la última versión 22 (Opción de uso recomendable para cuando necesites varias versiones de Node):

```bash
nvm install 22  # Por ejemplo, para instalar la versión 22.##.#
```

Node utiliza como versiones LTS los números pares. Actualmente las LTS son la 22, la 20 sigue en mantenimiento y la 18 está en proceso de ser deprecada y de dejar de mantenerse. Es recomendable usar estas versiones de Node porque siguen recibiendo actualizaciones de seguridad.

### 5.4 Verificar la instalación

```bash
node --version
npm --version
```

### 5.5 Comandos útiles de NVM

```bash
nvm ls                 # Listar versiones instaladas
nvm use 16.20.0        # Cambiar a una versión específica
nvm use 20             # Cambiar a la última versión LTS instalada que comience con 20
nvm current            # Ver versión actual en uso
```

## 6. INSTALAR NGINX

NGINX será nuestro servidor reverse proxy que gestionará todas las peticiones entrantes y las redirigirá a nuestros servidores Node.js locales.

### 6.1 Instalación de NGINX

```bash
# Actualizamos el repositorio para tener la última versión
sudo apt update

# Instalamos NGINX
sudo apt install nginx -y
```

Esto ejecutará el instalador con las opciones por defecto, que son las necesarias para nuestro uso. NGINX se instalará como un servicio de nuestro servidor. Para interactuar con este servicio y con todos los servicios instalados en el servidor usaremos la herramienta `systemctl`

### 6.2 Verificar la instalación

```bash
# Comprobar el estado del servicio
sudo systemctl status nginx
```

Debería salir un texto con un punto verde donde nos dice si el servicio está funcionando, si está `enabled` (Es decir, si está habilitado y por tanto se iniciará automáticamente con cada reinicio del sistema) y las últimas salidas de consola de la aplicación.

### 6.3 Comandos básicos de gestión

```bash
# Iniciar NGINX
sudo systemctl start nginx

# Detener NGINX
sudo systemctl stop nginx

# Reiniciar NGINX
sudo systemctl restart nginx

# Recargar la configuración sin detener el servicio (Últil cuando hemos realizado modificaciones de configuración y queremos que se ejecuten sin reiniciar el servicio)
sudo systemctl reload nginx

# Habilitar NGINX para que se inicie con el sistema
sudo systemctl enable nginx
```

### 6.4 Verificar el firewall

Asegúrate de que NGINX puede recibir tráfico permitiendo los puertos HTTP y HTTPS (En teoría este paso ya lo deberíamos haber realizado, pero no está de más asegurarse permitiendo acceso a la aplicación a través del firewall):

```bash
sudo ufw allow 'Nginx Full'
```

### 6.5 Verificar la instalación y funcionamiento

Abre un navegador y visita la IP de tu servidor. Deberías ver la página de bienvenida por defecto de NGINX. En este momento ya tendrías el servidor web de NGINX funcionando.

## 7. INSTALAR EL SERVIDOR DE BASE DE DATOS

La elección del servidor de base de datos dependerá de las necesidades específicas de tu proyecto. Aquí cubriremos la instalación tanto de MySQL como de MongoDB, que son las opciones más comunes.

### 7.1 Instalación de MySQL

```bash
# Actualizar repositorios
sudo apt update

# Instalar MySQL Server
sudo apt install mysql-server -y

# Iniciar el servicio MySQL
sudo systemctl start mysql

# Habilitar MySQL para que inicie con el sistema
sudo systemctl enable mysql

# Ejecutar el script de seguridad de MySQL
sudo mysql_secure_installation
```

Durante la ejecución de `mysql_secure_installation`, se te harán varias preguntas:

- Configurar la validación de contraseña (recomendado: Y)
- Nivel de validación de contraseña (recomendado: 2)
- Cambiar la contraseña de root
- Eliminar usuarios anónimos (recomendado: Y)
- Deshabilitar acceso root remoto (recomendado: Y)
- Eliminar base de datos de prueba (recomendado: Y)
- Recargar privilegios (recomendado: Y)

### 7.2 Instalación de MongoDB

[Instrucciones de instalación oficiales](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/)

```bash
# Importar la clave pública de MongoDB
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg \
   --dearmor

# Crear archivo de lista del repositorio en Ubuntu 24 para MongoDB (Consulta la guía para otras versiones)
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

# Actualizar el repositorio
sudo apt update

# Instalar MongoDB
sudo apt install -y mongodb-org

# Iniciar el servicio MongoDB
sudo systemctl start mongod

# Habilitar MongoDB para que inicie con el sistema
sudo systemctl enable mongod
```

### 7.3 Verificación de la instalación

Para MySQL:

```bash
# Verificar estado del servicio
sudo systemctl status mysql

# Verificar versión
mysql --version
```

Para MongoDB:

```bash
# Verificar estado del servicio
sudo systemctl status mongod

# Verificar versión
mongod --version
```

### 7.4 Consideraciones de seguridad importantes

#### 7.4.1 **Acceso remoto**:

Por defecto, tanto MySQL como MongoDB solo escuchan conexiones locales (127.0.0.1). Es recomendable mantener esta configuración si tu API está en el mismo servidor.

#### 7.4.2 **Puertos y Firewall**:

SI aún así quieres dar acceso externo, por seguridad, es recomendable cambiar los puertos por defecto de las bases de datos:

Para MySQL (cambiar puerto 3306):

```bash
# Editar la configuración de MySQL
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

# Buscar o añadir la línea:
port = 3307  # O el puerto que prefieras

# Reiniciar MySQL
sudo systemctl restart mysql
```

Para MongoDB (cambiar puerto 27017):

```bash
# Editar la configuración de MongoDB
sudo nano /etc/mongod.conf

# Buscar la sección net: y modificar el puerto:
net:
  port: 27018  # O el puerto que prefieras

# Buscar la sección bindIp y modificar la ip a la que se escucha:

bindIp: \* # Escuchará a todas las ips
bindIp: 127.0.0.1 # Escuchará solo al propio servidor de manera local

# Reiniciar MongoDB

sudo systemctl restart mongod
```

Después deberás abrir los nuevos puertos en tu firewall:

```bash
# Para MySQL (ejemplo con puerto 3307)
sudo ufw allow 3307/tcp

# Para MongoDB (ejemplo con puerto 27018)
sudo ufw allow 27018/tcp
```

---

⚠️ **IMPORTANTE**:

- Documenta los nuevos puertos en un lugar seguro
- Actualiza la configuración de conexión en tus aplicaciones
- Considera usar un puerto no estándar para dificultar el escaneo automático
- Aun así, es preferible mantener el acceso solo a nivel local

---

#### 7.4.3 **Usuarios y permisos**: Crea usuarios específicos para tus aplicaciones con los mínimos privilegios necesarios:

Para MySQL:

```bash
sudo mysql
```

```sql
CREATE USER 'tuusuario'@'localhost' IDENTIFIED BY 'tucontraseña';
GRANT ALL PRIVILEGES ON tubasededatos.* TO 'tuusuario'@'localhost';
FLUSH PRIVILEGES;
```

Para MongoDB:

```bash
mongosh
```

```javascript
use admin
db.createUser({
  user: "tuusuario",
  pwd: "tucontraseña",
  roles: [{ role: "readWrite", db: "tubasededatos" }]
})
```

⚠️ **IMPORTANTE**:

- Usa contraseñas fuertes y únicas
- Evita usar el usuario root/admin para las aplicaciones
- Mantén las bases de datos actualizadas con los últimos parches de seguridad
- Realiza backups regulares
- Si es posible, mantén las conexiones solo a nivel local (localhost / 127.0.0.1)

#### 7.4.4 **MySQL: Realiza las operaciones necesarias para crear la estructura (y datos si ya tuvieras) de las bases de datos de tus aplicaciones**:

Si ya tienes una base de datos creada en tu ordenador es el momento de exportar su estructura y datos via consulta SQL y transferirla al servidor usando SFTP (Mismo acceso que SSH, puerto incluido si lo has modificado) para poder ejecutarla en el servidor para que te cree la base de datos y los datos que hayas exportado.

El usuario sería el usuario de base de datos que has creado en el punto anterior para esta base de datos. No confundir con el usuario del sistema que vamos a crear en el siguiente punto.

```bash
# Primero crear la base de datos si no existe
mysql -u tuusuariodebasededatos -p -e "CREATE DATABASE IF NOT EXISTS nombre_base_datos;"

# Luego importar el archivo
mysql -u tuusuariodebasededatos -p nombre_base_datos < backup.sql
```

## 8. CREAR USUARIO SIN PRIVILEGIOS PARA LOS SERVIDORES NODE Y ESTRUCTURA DE CARPETAS

Por razones de seguridad, es una buena práctica ejecutar nuestros servidores Node.js bajo un usuario sin privilegios de administrador. Esto limita el daño potencial en caso de que una de nuestras aplicaciones sea comprometida.

### 8.1 Crear el nuevo usuario (modifica nodeuser por el nombre que prefieras)

```bash
# Crear usuario sin directorio home
sudo useradd --no-create-home nodeuser

# O si prefieres crear el directorio home (opcional)
sudo useradd --create-home nodeuser

# Establecer una contraseña (aunque no la necesitaremos para ejecutar servicios)
sudo passwd nodeuser
```

### 8.2 Crear y configurar el directorio para las aplicaciones

```bash
# Crear directorio para las aplicaciones
sudo mkdir -p /var/www/nodeapps

# Establecer nodeuser como propietario
sudo chown -R nodeuser:nodeuser /var/www/nodeapps

# Establecer permisos adecuados
sudo chmod -R 755 /var/www/nodeapps
```

### 8.3 Verificar la configuración

```bash
# Verificar propietario y permisos
ls -la /var/www/nodeapps

# Probar acceso como nodeuser
sudo -u nodeuser ls -la /var/www/nodeapps
```

### 8.4 Consideraciones importantes

- El usuario `nodeuser` no podrá ejecutar comandos con `sudo`
- Solo podrá acceder y modificar archivos dentro de `/var/www/nodeapps`
- A la hora de transferir archivos via SFTP a `/var/www/nodeapps` es recomendable usar `nodeuser` para que ese usuario sea el propietario de esos archivos transferidos.
- Los servicios de Node.js se ejecutarán bajo este usuario
- Para realizar operaciones que requieran más privilegios, deberás usar tu usuario con privilegios sudo

### 8.5 Estructura recomendada de directorios (modificar myapp# por el nombre que identifique tu proyecto)

Vamos a crear, aparte de nuestros directorios que contendrán los proyectos webs, un directorio donde guardaremos los scripts que inicio del servicio web para cada proyecto.

```bash
/var/www/nodeapps/
├── myapp1/
│   ├── src/
│   └── node_modules/
├── myapp2/
│   ├── src/
│   └── node_modules/
└── start_scripts
```

Para crear esta estructura:

```bash
# Crear directorios para las aplicaciones
sudo -u nodeuser mkdir -p /var/www/nodeapps/myapp1
sudo -u nodeuser mkdir -p /var/www/nodeapps/myapp2
sudo -u nodeuser mkdir -p /var/www/nodeapps/start_scripts

# Verificar la estructura
tree /var/www/nodeapps
```
