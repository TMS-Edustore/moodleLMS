<!-- Basic Setup -->
<!-- Note: This old documentation uses PHP 5, but current Moodle 3.5 should use PHP 7 -->
<!-- this documentation is for linux (ubuntu/debian) -->


apt-get update
apt-get install git git-core git-doc
apt-get install php5-gd php5-curl php5-intl php5-xmlrpc
apt-get install php5 mysql-server mysql-client apache2  php5-mysql
apt-get install php5-json


Install Moodle code

cd /var/www
<!-- i prefer using /var/www/html -->
git clone https://github.com/moodle/moodle.git
cd moodle
git checkout -t origin/MOODLE_39_STABLE
chmod 0777 /var/www/moodle


Create the data area
mkdir /var/moodledata
chmod 0777 /var/moodledata

Create the database
<!-- for the database, i had an issue using mysql, as the latest version of moodle i got required mysql 8.4 and my ubuntu 24.04 only had the 8.0 package available, i decided to use mariadb -->

<!-- using mariadb -->
sudo apt update
sudo apt install mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation
mariadb --version

<!-- but if you prefer to use mysql, you can install it as follows -->
sudo apt update
sudo apt install mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
sudo mysql_secure_installation
mysql --version

<!-- dont do the two steps above, only one is what you need. although mariadb tries not tu interfere with mysql installation, but in my experience sometimes it does, and that my cause some uneccessary overhead. just choose and use one option -->

then you can proceed
mysql -u root -p
(asks for password here)
mysql> create database moodle default character set utf8;
mysql> grant all on moodle.* to moodleuser@localhost identified by 'mypassword';
mysql> exit


<!-- and finally -->
<!-- Don't skip this step. This secures the Moodle code, preventing it being overwritten by hackers. -->

chmod 0755 /var/www/moodle


<!-- now install apache2 if you dont already have it to serve the site -->
sudo apt update
sudo apt install apache2 -y
sudo systemctl start apache2
sudo systemctl enable apache2
<!-- you can verify the installation from -->
http://localhost
<!-- or you can check the status from  -->
sudo systemctl status apache2

<!-- installing php, you can start by verifying if php already exist-->
php -v
sudo apt update
sudo apt install php php-mysql php-gd php-xml php-mbstring php-curl php-zip php-intl -y
php -v


<!-- create a virtual host in sites enabled for your moodle -->
cd /etc/apache2/sites-enabled
vi moodle.conf
<!-- and paste the following content -->
<VirtualHost *:8084>
    DocumentRoot /var/www/html/moodle
    ServerName localhost

    <Directory /var/www/html/moodle>
        AllowOverride All
        Require all granted
    </Directory>

    # Add the following lines to configure PHP-FPM. 
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/var/run/php/php8.3-fpm.sock|fcgi://localhost"
    </FilesMatch>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>


<!-- the following line is not neccessary, but for better processing, its adviced -->
 <!-- <FilesMatch \.php$>
        SetHandler "proxy:unix:/var/run/php/php8.3-fpm.sock|fcgi://localhost"
</FilesMatch> -->

since you are creating a virtualhost to listen on port 8084, you will need to add this port in apache2 ports
root@TS-Mnaan:/etc/apache2# cat ports.conf
# If you just change the port or add more ports here, you will likely also
# have to change the VirtualHost statement in
# /etc/apache2/sites-enabled/000-default.conf

Listen 0.0.0.0:80
Listen 0.0.0.0:8082
Listen 0.0.0.0:8084

<IfModule ssl_module>
        Listen 0.0.0.0:443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 0.0.0.0:443
</IfModule>
root@TS-Mnaan:/etc/apache2#

<!-- the above lines constitute my configuration for ports.conf -->

<!-- When to Use PHP-FPM? -->
<!-- 
If you are using Nginx, you must use PHP-FPM (because Nginx does not support mod_php).
If you want better performance on Apache or other web servers.
If your server handles high traffic and needs efficient process management.
If you want more control over PHP execution and resource usage. 
-->

<!-- installing php-fpm -->
sudo apt update
sudo apt install php-fpm -y
sudo systemctl enable php-fpm
sudo systemctl start php-fpm

<!-- configuring the max_var_limit -->
<!-- locate the php.ini file and find the variable max_input_vars -->
<!-- for me, i also adjust the following -->
max_file_uploads
post_max_size
upload_max_filesize
the php.ini file is usually located in the 
/etc/php/php -v/fpm/php.ini example /etc/php/8.3/fpm/php.ini where the php version is 8.3
<!-- there is another php.ini file in /etc/php/php -v/apache2/php.ini and /etc/php/php -v/cli/php.ini, i have not tried to modify the one in cli to see if it works also though. but modifying the one in apache2 didnt work for the moodle configuration for me. -->

<!-- to search for the variables -->
cat -n /etc/php/8.3/fpm/php.ini | grep max_
<!-- the above line will give you the line numbers of the all the lines with max_, you can now trace and locate what you need -->


network configuration setup
in my case my server is running on ubuntu in wsl2 on a windows machine, so i need to port proxy from windows to wsl to access the server

<!-- run this on your windows machine as an adminsitration -->
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=<WindowsPort> connectaddress=<WSL_IP> connectport=<WSLPort>

<!-- if you want to check all port proxies -->
netsh interface portproxy show all

<!-- to be able to access your site on the network from another device, you need to update the config.php file -->
modify the following line
$CFG->wwwroot   = 'http://localhost:8084';
to:
$CFG->wwwroot   = 'http://windows-ip:8084';
