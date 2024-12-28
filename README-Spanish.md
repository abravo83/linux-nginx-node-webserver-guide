# INSTRUCCIONES PARA LA CONFIGURACIÓN DE UN SERVIDOR LINUX COMO SERVIDOR WEB NODE CON VARIOS SERVIDORES EXPRESS

## El problema

Queremos poder tener disponible un servidor web Express o similar corriendo a través de un proceso node.

El problema es que un servidor web sólo responde a los puertos 80 si es http o al puerto 443 si es https por defecto.

Si configuramos un servidor Express que, por ejemplo, corra una aplicación web hecha con Angular o React y luego queremos añadir otra aplicación web que tenga una API y que también funcione escuchando al puerto 443 y 80. Cuando le hagamos la petición al servidor... ¿Desde cuál de los servidores express-node contesta?

![Figura 1: Un servidor con varios node escuchando al puerto 443](img/fig-1.png)

¿Y si añadimos aún más?

El problema será por tanto que nuestro servidor no va a saber desde que node contestar. Si le hemos añadido un dominio `www.midominioweb.com` y aunque añadamos un segundo dominio `www.midominioapi.com`, va a dar igual, ya que los dominios sólo indican a que IP dirigir las peticiones, y no a que puerto. Por lo cual no podemos esperar que cada uno de los dominios sea contestado por diferentes servidores Express.

## Las soluciones

- Una solución sería ir contratando un servidor por aplicación web. Pero va a ser una solución algo costosa e ineficiente.

- Otra solución podría ser especificar diferentes puertos y que los servidores web no estén escuchado al mismo puerto, y esta solución es quizás más factible para una API dejando la web React/Angular en el 443 para que se pueda llamar desde `https://www.midominioweb.com` (Cuando no se especifica puerto va al 443 si tiene https y al 80 si tiene http) mientras que las peticiones a la API podrían estar dirigidas específicamente a otros puertos: `https://www.midominioweb.com:3030`.

  Esta solución nos es válida en principio pero si añadimos más webs que sirvan frontends no podemos esperar que los usuaios de esas webs usen dominios que especifique puertos. Es una solución que da poca seriedad. Además esta solución nos va a traer otros problemas a la hora de expedir y renovar certificados SSL para poder usar `https://`

- La solución que se propone aquí es la de usar un tipo de servidor web que sea el único que escucha a esos puertos 443 y 80 y que este a su vez reenvíe esas peticiones a otros servidores web corriendo de forma local en el servidor pero en otros puertos diferentes (p.e. 3010, 3011...) y estos servidores locales no van a estar expuestos directamente -Sus puertos no van a estar abiertos en el firewall-. A este tipo de servidor web que reparte peticiones a otros se le conoce como un **Reverse Proxy**. Si además la combinamos con el uso de subdominios conseguimos que a través de un único domínio podemos hacer disponibles muchísimos servicios web: `https://www.midominioweb.com`, `https://api.midominioweb.com`, `https://mywebapp2.midominioweb.com` ya que el reverse proxy analizará de que dominio viene cada petición y lo repartirá al servidor local web adecuado.

  Además es una solución común que va a funcionar correctamente con clientes de gestión de certificados ssl como _certbot_ y que nos van a permitir dejar automatizado el proceso de renovación de esos certificados.

![Figura 1: Un servidor escuchando con un reverse proxy al puerto 443](img/fig-2.png)

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
