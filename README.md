Aquí tienes una versión reescrita y explicada de manera diferente del proyecto Docker Compose que me proporcionaste:

---

## **Configuración de dos sitios web con Docker Compose**

En este proyecto se configuraron dos sitios web denominados **fabulasmaravillosas** y **fabulasoscuras**, ambos accesibles a través de una misma dirección IP de un servidor DNS. A continuación se detallan los pasos para conseguirlo:

### **1. Archivo Docker Compose**

El archivo `docker-compose.yml` define dos servicios: uno para los servidores web (donde se alojan las páginas) y otro para el servidor DNS. Además, se configura una red que permite la comunicación entre estos servicios.

#### **Servicio web**

En esta sección se define el contenedor para el servidor web, utilizando una imagen de **PHP 7.4 con Apache**. Además, se configuran varios parámetros como el puerto de escucha (80) y los volúmenes que vinculan el contenido de las páginas web desde el host hacia el contenedor. La red utilizada se denomina `apared` y se asigna una IP estática.

```yaml
web:
    image: php:7.4-apache
    container_name: apaserver
    ports:
      - "80:80"
    volumes:
      - ./www:/var/www
      - ./confApache:/etc/apache2
    networks:
      apared:
        ipv4_address: 172.39.4.2
```

#### **Servicio DNS**

Este servicio utiliza la imagen de **Ubuntu con Bind9**, que es un servidor DNS. Al igual que el servicio web, se configura el puerto (53) y se asigna una IP dentro de la misma red `apared`, pero con una dirección IP diferente.

```yaml
dns:
    container_name: dnsserver
    image: ubuntu/bind9
    ports:
      - "51:53"
    volumes:
      - ./confDNS/conf:/etc/bind
      - ./confDNS/zonas:/var/lib/bind
    networks:
      apared:
        ipv4_address: 172.39.4.3
```

#### **Configuración de la red**

Se define la red `apared` utilizando el modo de red **bridge**, lo que permite que los contenedores se comuniquen entre sí y con el host. También se establece un rango de direcciones IP para los servicios.

```yaml
networks:
  apared:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.39.0.0/16
```

### **2. Configuración de Apache**

Dentro de la carpeta **`confApache`**, se deben colocar dos archivos de configuración de Apache en las carpetas `sites-available` y `sites-enabled`. Estos archivos configuran los sitios web y sus alias.

#### **fabulasmaravillosas.conf**

Este archivo se usa para configurar el primer sitio, con el nombre de dominio **fabulasmaravillosas.asircastelao.int**. En el archivo se definen el administrador del servidor y la ruta donde se encuentra el archivo `index.html`.

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName fabulasmaravillosas.asircastelao.int
    ServerAlias www.fabulasmaravillosas.asircastelao.int
    DocumentRoot /var/www/fabulasmaravillosas
</VirtualHost>
```

#### **fabulasoscuras.conf**

De forma similar, se configura el segundo sitio web con el nombre **fabulasoscuras.asircastelao.int**. También se define la ruta donde se encuentra el archivo `index.html`.

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName fabulasoscuras.asircastelao.int
    ServerAlias www.fabulasoscuras.asircastelao.int
    DocumentRoot /var/www/fabulasoscuras
</VirtualHost>
```

### **3. Configuración de DNS**

Dentro de la carpeta **`confDNS`**, se deben configurar tanto las zonas como los registros de los sitios web.

#### **Archivo de zonas (db.asircastelao.int)**

En este archivo se definen los registros de recursos (RR) para los dominios. Específicamente, se asocia la dirección IP de cada sitio web con su nombre de dominio correspondiente.

```text
$TTL 604800  
@       IN      SOA     ns.asircastelao.int. some.email.address.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative 

@       IN      NS      ns.asircastelao.int.
ns       IN      A       172.39.4.3
fabulasoscuras       IN      A       172.39.4.2
fabulasmaravillosas  IN      A       172.39.4.2
```

#### **Archivos de configuración de Bind9**

Además, se deben configurar tres archivos importantes para Bind9, que permiten que el servidor DNS funcione correctamente:

1. **named.conf.local**: Define la zona maestra del dominio y el archivo de zonas.
   
```text
zone "asircastelao.int" {
    type master;
    file "/var/lib/bind/db.asircastelao.int";
    allow-query {
        any;
    };
};
```

2. **named.conf.options**: Configura opciones como la resolución recursiva y los servidores de reenvío (por ejemplo, Google DNS y Cloudflare DNS).

```text
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    dnssec-validation no;
    forwarders {
        8.8.8.8;
        1.1.1.1;
    };
    listen-on { any; };
    listen-on-v6 { any; };
};
```

3. **named.conf**: Simplemente incluye los archivos anteriores.

```text
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```

4. **named.conf.default-zones**: Define las zonas inversas para la resolución de direcciones IP.

### **4. Creación de las páginas web**

Las carpetas **`fabulasmaravillosas`** y **`fabulasoscuras`** se deben crear dentro de la carpeta **`www`**, que se mapea al contenedor Apache. En cada una de estas carpetas, debes colocar el archivo **`index.html`** correspondiente a cada página web.

Si Apache no está instalado en el sistema, es necesario instalarlo con los siguientes comandos:

```bash
sudo apt install apache2
sudo ufw allow 'Apache'
sudo systemctl restart apache2
```

### **5. Configuración del DNS en el sistema**

Para que el sistema utilice el servidor DNS configurado en el contenedor, se debe editar el archivo **`/etc/systemd/resolved.conf`** y agregar la IP del servidor DNS:

```text
DNS=172.39.4.3#51
```

Después, se reinicia el servicio para aplicar los cambios:

```bash
sudo systemctl restart systemd-resolved
```

### **6. Verificación**

Una vez que todos los pasos anteriores estén completos, ejecuta el siguiente comando para levantar los contenedores:

```bash
docker compose up
```

En el navegador, puedes acceder a los sitios web mediante los alias configurados, como **`www.fabulasmaravillosas.asircastelao.int`** o **`www.fabulasoscuras.asircastelao.int`**. Si se presentan errores, asegúrate de que Apache no esté en ejecución en el sistema host, ya que puede interferir con el servidor web dentro del contenedor. Si es necesario, detén Apache con:

```bash
sudo systemctl stop apache2
```

---

