# Práctica 4. Asegurar la granja web

En esta práctica el objetivo es llevar a cabo la configuración de la seguridad de la granja web. Para ello, llevaremos a cabo las siguientes tareas:
* Instalar un certificado SSL para configurar el acceso HTTPS a los servidores.
* Configurar las reglas del cortafuegos para proteger la granja web.


  
## Instalando un certificado SSL autofirmado para HTTPS

Vamos a generar e instalar el certificado SSL en nuestros dos servidores y habilitar el balanceador Nginx para que permita tráfico HTTPS.

Lo primero que necesitamos es activar el módulo SSl de apache y generar los certificados con privilegios de administrador:

    a2enmod ssl 
    service apache2 restart
    mkdir /etc/apache2/ssl
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/apache2/ssl/apache.key -out /etc/apache2/ssl/apache.crt

Al crear el certificado nos pedirá que rellenos algunos datos de identidad.

Los certificados sólo debemos crearlos una vez y copiarlos al resto de servidores, pero repetir el resto de tareas:

    #Máquina 2
    scp /etc/apache2/ssl/apache.* jescobar@192.168.56.120:~

    #Balanceador Nginx
    scp /etc/apache2/ssl/apache.* jescobar@192.168.56.130:/tmp

A continuación editamos el archivo de configuración */etc/apache2/sites-available/default-ssl* y le añadimos:

    SSLCertificateFile /etc/apache2/ssl/apache.crt 
    SSLCertificateKeyFile /etc/apache2/ssl/apache.key

Activamos el sitio default-ssl y reiniciamos apache:

    a2ensite default-ssl
    service apache2 reload

Si todo va bien deberíamos poder conectarnos a los servidores mediante HTTPS. Una prueba rápida podría ser:

    curl -k https://ipmaquina1/index.html

Finalmente, para el funcionamiento normal de la granja web, necesitamos configurar el balanceador para que permita las peticiones mediante SSL. 

Con añadir al archivo */etc/nginx/conf.d/default.conf* estas líneas y reiniciar Nginx habremos terminado:

    listen 443 ssl; 
    ssl on; 
    ssl_certificate /tmp/apache.crt; 
    ssl_certificate_key /tmp/apache.key;

Ahora ya podremos hacerle peticiones por HTTPS a la IP del balanceador.

## Configuración del cortafuegos

En esta práctica vamos trabajar principalmente con dos scripts donde incluimos reglas de *iptables* con las que configuramos el comportamiento del firewall.

Un script inicial que permite todo el tráfico:

    #!/bin/bash
    # (1) Eliminar todas las reglas (configuración limpia)
    iptables -F 
    iptables -X 
    iptables -Z 
    iptables -t nat -F 

    # política por defecto: aceptar todo 
    iptables -P INPUT ACCEPT 
    iptables -P OUTPUT ACCEPT 
    iptables -P FORWARD ACCEPT 
    
    iptables -L -n -v

Este otro script bloquea todo el tráfico inicialmente y a continuación le permitimos el uso de los puertos necesarios para el funcionamiento de SSH, HTTP y HTTPS:

    #!/bin/bash
    # (1) Eliminar todas las reglas (configuración limpia) 
    iptables -F
    iptables -X
    iptables -Z
    iptables -t nat -F

    # (2) Política por defecto: denegar todo el tráfico 
    iptables -P INPUT DROP
    iptables -P OUTPUT DROP
    iptables -P FORWARD DROP

    # (3) Permitir cualquier acceso desde localhost (interface lo) 
    iptables -A INPUT -i lo -j ACCEPT
    iptables -A OUTPUT -o lo -j ACCEPT

    # (4) Abrir el puerto 22 para permitir el acceso por SSH 
    iptables -A INPUT -p tcp --dport 22 -j ACCEPT
    iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT

    # (5) Abrir los puertos HTTP (80) de servidor web 
    iptables -A INPUT -p tcp --dport 80 -j ACCEPT
    iptables -A OUTPUT -p tcp --sport 80 -j ACCEPT

    # (6) Abrir los puertos HTTP (443)
    iptables -A INPUT -p tcp --dport 443 -j ACCEPT
    iptables -A OUTPUT -p tcp --sport 443 -j ACCEPT

    # (7) Abrir los puertos SSH (22)
    iptables -A INPUT -p tcp --dport 22 -j ACCEPT
    iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT

    iptables -L -n -v

