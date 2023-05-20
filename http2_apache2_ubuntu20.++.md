Based on https://gist.github.com/GAS85/8dadbcb3c9a7ecbcb6705530c1252831

# Requirements

* A self-managed VPS or dedicated server with Ubuntu 20.04 running Apache 2.4.xx.
* A registered domain name with working HTTPS (TLS/SSL). HTTP/2 only works alongside HTTPS because most browsers, including Firefox and Chrome, don’t support HTTP/2 in cleartext (non-TLS) mode.

## Step 1: Install Apache2

Per default it will be [apache2 version 2.4.41](https://packages.ubuntu.com/focal/web/apache2) what is enought for http2 support.

      sudo apt install apache2

## Step 2: Tell Apache to use PHP FastCGI

You want to make Apache use a compatible PHP implementation by changing mod_php to php-fpm (PHP FastCGI). If your website or app breaks on FastCGI, you can always revert back to mod_php until further troubleshooting.

Install PHP FastCGI module for PHP 7.4, it is [default version for Ubuntu 20.04](https://packages.ubuntu.com/focal/php/php-fpm)

    sudo apt install php7.4-fpm 

Enable required modules, **proxy_fcgi** and **setenvif**:

    sudo a2enmod proxy_fcgi setenvif

Enable **php7.4-fpm**:

    sudo a2enconf php7.4-fpm 

_Disable_ the **mod_php** module:

    sudo a2dismod php7.4

Restart Apache:

    sudo service apache2 restart

## Step 3: Change MPM from "prefork" to "event"

(This step is not needed for Ubuntu 22.04 and above)

Since the default "prefork" MPM (Multi-Processing Module) is not fully compatible with HTTP/2, you’ll need to change Apache’s current MPM to "event" (or "worker"). This is shown by the error message in Apache versions greater than 2.4.27 as – AH10034: The mpm module (prefork.c) is not supported by mod_http2.

Keep in mind that your server requires more horsepower for HTTP/2 than for HTTP/1.1, due to the multiplexing feature and other factors. That said, smaller servers with low traffic may not see much difference in performance.

First, disable the "prefork" MPM:

    sudo a2dismod mpm_prefork 

Enable the "event" MPM:

    sudo a2enmod mpm_event 

Restart Apache2 and PHP 7.4:

    sudo service apache2 restart && sudo service php7.4-fpm restart

## Step 4: Add a line to your Virtual Host file

Add the following line to your site’s current Virtual Host config file. This can go anywhere between the <VirtualHost>...</VirtualHost> tags. If you want to serve HTTP/2 for all your sites, add this to your global /etc/apache2/apache2.conf file instead of per each individual site’s Virtual Host file.

    Protocols h2 h2c http/1.1

_Explanation_: h2 is TLS-encrypted HTTP/2, h2c is cleartext HTTP/2, and http/1.1 is ordinary HTTP/1.1.

Having http/1.1 at the end of the line provides a fallback to HTTP/1.1, while h2c is not strictly necessary.

## Step 5: Enable the mod_http2 Apache module

Now you can enable the http2 module in Apache:

    sudo a2enmod http2

Check Apache2 config and if no errors, restart Apache:

    sudo apachectl configtest && sudo service apache2 restart

## Step 6 create http2.conf for entire Server HTTP2

Create a new http2.conf

    sudo nano /etc/apache2/conf-available/http2.conf

and add all the following rows:

    <IfModule http2_module>
        Protocols h2 h2c http/1.1
        H2Direct on
    </IfModule>

Enable the http2.conf by running

    sudo a2enconf http2

Check Apache2 config and if no errors, restart your Apache2

    sudo apachectl configtest && sudo service apache2 restart
    
and enhance your ssl-vhost file (default-ssl.conf):

    sudo nano /etc/apache2/sites-available/default-ssl.conf

Amend in your configuration file:

    ...
    Protocols h2 h2c http/1.1
    H2Push on
    H2PushPriority * after
    H2PushPriority text/css before
    H2PushPriority image/jpg after 32
    H2PushPriority image/jpeg after 32
    H2PushPriority image/png after 32
    H2PushPriority application/javascript interleaved
    ...

P.S. All in one command (you still have to edit your VirtualHost and ssl config):

    sudo apt update
    sudo apt upgrade
    sudo apt install apache2 php7.4-fpm 
    sudo a2enmod proxy_fcgi setenvif
    sudo a2enconf php7.4-fpm 
    sudo a2dismod php7.4 
    sudo service apache2 restart
    sudo a2dismod mpm_prefork 
    sudo a2enmod mpm_event 
    sudo service apache2 restart 
    sudo service php7.4-fpm restart
    sudo a2enmod http2
    sudo service apache2 restart
    sudo echo "<IfModule http2_module>" > /etc/apache2/conf-available/http2.conf
    sudo echo "Protocols h2 h2c http/1.1" >> /etc/apache2/conf-available/http2.conf
    sudo echo "H2Direct on" >> /etc/apache2/conf-available/http2.conf
    sudo echo "</IfModule>" >> /etc/apache2/conf-available/http2.conf
    sudo a2enconf http2
    sudo apachectl configtest && sudo service apache2 restart