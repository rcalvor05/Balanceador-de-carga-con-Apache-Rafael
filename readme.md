# Tarea Balanceador

## Vagrantfile
Creamos el **vagrantfile** especificando que vamos a crear 4 maquinas virtuales con Debian 12
- **Balanceador** Reparte las peticiones de los clientes entre web 1 y web 2
- **Web1 y Web2** Son los 2 servidores web son identicas, solo cambia el nombre y las IPs, por eso las creamos con un bucle
- **Base de datos** Se comparte con las 2 webs, para que los datos sean los mismo


#### Hacemos un `vagrant up` y luego un `vagrant status` y debería acabar así
![Status](/img/status.png)

### Conexión 

Nos conectamos por ssh a cada maquina de esta manera
`vagrant ssh "NOMBRE DE LA MAQUINA"`
El nombre de la maquina aparece en el vagrant status que hemos echo antes

Yo como uso el terminal de kitty me da algunos problemas con ssh entonces para poder usar clear y demás en la terminal de ssh pongo esto `export TERM=xterm-256color`

Hacemos un `ip -br a` para comprobar que todas las maquinas estan en las redes que necesitamos y esta todo en orden según hemos configurado en el vagrantfile

Haremos unos ping en web1 y en el balanceador
### Web1
`ping -c 2 10.10.1.10`
`ping -c 2 10.10.2.20`

### Balanceador
`ping -c 2 10.10.2.20`

Los de web 1 deberán funcionar correctamente y del balanceador fallará
![pings](/img/pings.png)

## Base de datos`

En la maquina virtual de la base de datos instalaremos MariaDB

`sudo apt update`
`sudo apt install -y mariadb-server`

**Comprobación**
`systemclt status mariadb`
![status mariadb](/img/status-mariadb.png)

## Configuración red privada
Por defecto MariaDB solo acepta conexiones desde localhost esto es un problema porque las webs no se podrán conectar, la pondremos a escuchar su ip de red_bbdd

Entramos en este fichero `/etc/mysql/mariadb.conf.d/50-server.cnf` y buscamos esta linea
`bind-address            = 127.0.0.1` 
Cambiamos la ip de localhost por esta `10.10.2.20`

**REINICIAMOS MARIADB**
`sudo systemctl restart mariadb`

Hacemos la comprobación de que ha funcionado
![Testmariadb](/img/test.png)

En el terminal ponemos `sudo mariadb` dentro ejecutamos esto, cambiamos los asteriscos por una clave real
`CREATE DATABASE app_balanceador;`

`CREATE USER 'usuario_web'@'10.10.2.%' IDENTIFIED BY '********';`

`GRANT SELECT, INSERT ON app_balanceador.* TO 'usuario_web'@'10.10.2.%';`

`FLUSH PRIVILEGES;`

`USE app_balanceador;`

`CREATE TABLE visitas (`

`    id INT AUTO_INCREMENT PRIMARY KEY,`

`    servidor VARCHAR(50) NOT NULL,`

`    fecha DATETIME DEFAULT CURRENT_TIMESTAMP`

`);`

`EXIT;`

Comprobamos que todo esta en orden con este comando `sudo mariadb -e "SELECT user, host FROM mysql.user; SHOW TABLES FROM app_balanceador;"`
![goodmdb](/img/good.png)

## Probar conexión web1
Desde el terminal de **web1** pondremos los siguientes comandos
`sudo apt update`

`sudo apt install -y mariadb-client`

`mariadb -h 10.10.2.20 -u usuario_web -p -e "SHOW TABLES FROM app_balanceador;"`

Pedirá la contraseña y una vez se la demos nos saldra lo siguiente:
![mariadbweb1](/img/mariadbweb1.png)


## Configuración Web1, Web2
**TODA LA CONFIGURACIÓN SE DEBE HACER EN AMBAS MAQUINAS**

Descargamos Apache y PHP
`sudo apt install -y apache2 php libapache2-mod-php php-mysql`

Comprobamos la instalación
`systemctl status apache2`

![web1](/img/apache1.png)
![web2](/img/apache2.png)

### Archivo con los datos de conexión
`sudo nano /var/www/config.php`

Pondremos esto dentro del archivo
`<?php
$db_host = "10.10.2.20";
$db_user = "usuario_web";
$db_pass = "La misma contraseña que en la base de datos";
$db_name = "app_balanceador";`

### Página de la aplicación

1º Borramos la pagina por defecto de Apache
`sudo rm /var/www/html/index.html`

2º Creamos un archivo nuevo
`sudo nano /var/www/html/index.php`

```<?php

require '/var/www/config.php';

$servidor = gethostname();

try {
    $conexion = new mysqli($db_host, $db_user, $db_pass, $db_name);

    $sentencia = $conexion->prepare("INSERT INTO visitas (servidor) VALUES (?)");
    $sentencia->bind_param("s", $servidor);
    $sentencia->execute();

    $total = $conexion->query("SELECT COUNT(*) FROM visitas")->fetch_row()[0];
    $ultimas = $conexion->query("SELECT servidor, fecha FROM visitas ORDER BY id DESC LIMIT 5");
} catch (mysqli_sql_exception $e) {
    die("Error al conectar con la base de datos.");
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Práctica balanceador - Rafael Calvo Romero</title>
    <style>
        body { font-family: sans-serif; background: #1e293b; color: #f1f5f9; text-align: center; padding: 40px; }
        h1 span { color: #38bdf8; }
        table { margin: 20px auto; border-collapse: collapse; }
        td, th { border: 1px solid #475569; padding: 8px 16px; }
    </style>
</head>
<body>
    <h1>Te ha respondido: <span><?= htmlspecialchars($servidor) ?></span></h1>
    <p>Visitas totales registradas en la base de datos: <strong><?= $total ?></strong></p>
    <h2>Últimas visitas</h2>
    <table>
        <tr><th>Servidor</th><th>Fecha</th></tr>
        <?php while ($fila = $ultimas->fetch_assoc()): ?>
        <tr><td><?= htmlspecialchars($fila['servidor']) ?></td><td><?= $fila['fecha'] ?></td></tr>
        <?php endwhile; ?>
    </table>
</body>
</html>
```

Descargamos curl ya que en la imagen de debian que usa vagrant no viene descargado.
`sudo apt install -y curl`

Ahora comprobamos desde cada web 
`curl -s localhost | grep respondido`
![resweb1](/img/resweb1.png)
![resweb2](/img/resweb2.png)
