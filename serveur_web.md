# Introduction
Nous allons mettre en place un serveur web. Voici ce que nous allons installer :
- Nginx 1.23.3 (max version 12/02/23 avec bullseye)
- PHP 8.0 fpm
- MariaDb
- PhpMyAdmin 5.1.1
- Postfix avec relais Gmail
- Certbot
- Nodejs
- Socket.io
- Pm2


# Prérequis
1. Avoir un domaine chez un hébergeur avec la gestion de DNS
2. Avoir un VPS OVH sous la distribution Debian 11
3. Créer une adresse Gmail

### Configuration DNS domaine :
 `*` -> `A` -> `ip_vps`
 
 `www` -> `A` -> `ip_vps`

### Configuration Gmail :
Une fois votre compte créé, vous devez ***activer l'A2F*** afin d'avoir accès à l'option qui nous intéresse.
Rendez-vous dans les paramètres de sécurité et sélectionnez ***"Mots de passe des applications"***.
Il faut ensuite sélectionner l'application ***"Messagerie"*** puis l'appareil ***"Autres"***.
Gardez précieusement le mot de passe qui vous est donné, c'est ce mot de passe que nous allons utiliser dans notre configuration Postfix.


# Configuration
### 1 / Passage en root
Lors de l'achat d'un VPS chez OVH, nous n'avons pas l'accès root. Nous allons commencer par configurer notre accès root.

1.1 - Commencez par ajouter un mot de passe a l'utilisateur root 
```
sudo passwd root
```

1.2 - Passez ensuite votre VPS en super user
```
sudo apt update && sudo apt upgrade -y
sudo su -
sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/g' /etc/ssh/sshd_config
sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
service sshd restart
```


### 2 / Nginx
2.1 - Vous allez maintenant installer Nginx sous ca version maximum disponible pour la distribution ***Debian 11***
```
echo "deb https://nginx.org/packages/mainline/debian/ bullseye nginx" | sudo tee -a /etc/apt/sources.list
echo "deb-src https://nginx.org/packages/mainline/debian bullseye nginx" | sudo tee -a /etc/apt/sources.list
apt install gnupg    
wget https://nginx.org/keys/nginx_signing.key 
apt-key add nginx_signing.key
apt update && apt upgrade -y
apt install nginx -y
```



### 3 / Php
3.1 - Il faut a présent installé PHP, pour cette configuration nous avons choisi PHP8.0 fpm
```
apt -y install lsb-release apt-transport-https ca-certificates
wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list
apt update && apt upgrade -y
apt install php8.0 php8.0-fpm php8.0-mysqli php8.0-gd php8.0-mysql php8.0-cli php8.0-common php8.0-curl php8.0-opcache php8.0-imap php8.0-mbstring php-xml php8.0-xml -y
systemctl start nginx
```

3.2 - Créer un dossier sur votre VPS qui va vous servir de dossier racine à vos fichiers web ( dans notre cas nous l'appelons `htdocs` )
```
mkdir /htdocs
```

3.3 - Modifiez la configuration de PHP fpm 
```
nano /etc/php/8.0/fpm/pool.d/www.conf
```
- Modifiez `user = nginx` -> `user = www-data`
- Vérifiez : `listen = /run/php/php8.0-fpm.sock`

3.4 - Modifiez votre fichier de configuration principal de Nginx
```
nano /etc/nginx/nginx.conf
```
- Modifiez `user nginx` -> `user www-data`

3.5 - Modifiez votre fichier de configuration par défaut de Nginx
```
nano /etc/nginx/conf.d/default.conf
```
- **{server}** Modifiez :  `server_name localhost` -> `server_name domaine.com`
- **{server}** Modifiez :  `root /usr/share/nginx/html` -> `root /htdocs`
- **{server}** Ajoutez :  `index.php` a la liste des index
- **{server}** Ajoutez la location PHP qui se trouve ci-dessous
```
#Location PHP
  location ~ \.php$ {
      root           /htdocs;
      try_files      $uri =404;
      fastcgi_pass   unix:/run/php/php8.0-fpm.sock;
      fastcgi_index  index.php;
      fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
      include        fastcgi_params;
  }
```

3.6 - Enfin effectuez un redémarrage de PHP8.0 fpm et relancez de Nginx
```
service php8.0-fpm restart
systemctl reload nginx
```

### 4 / Maria Db
4.1 - Remplacez les `NAME` de `NAME@localhost` par le nom que vous souhaitez et `PASSWORD` par votre mot de passe que vous souhaitez
```
apt install mariadb-server -y
mysql_secure_installation (configurez le a votre guise)
mariadb
CREATE USER NAME@localhost IDENTIFIED BY 'PASSWORD';
GRANT ALL PRIVILEGES ON * . * TO NAME@localhost WITH GRANT OPTION;
quit
systemctl reload nginx
```

### 5 / PhpMyAdmin
5.1 - Installez  maintenant PhpMyAdmin sur votre serveur.
```
apt install unzip -y
wget https://files.phpmyadmin.net/phpMyAdmin/5.1.1/phpMyAdmin-5.1.1-all-languages.zip
unzip phpMyAdmin-5.1.1-all-languages.zip
mv phpMyAdmin-5.1.1-all-languages phpmyadmin
mv phpmyadmin /usr/share/phpmyadmin
```
5.2 - Editez une nouvelle fois votre fichier de configuration par défaut de Nginx
```
nano /etc/nginx/conf.d/default.conf
```

- **{server}** Ajoutez les deux location phpmyadmin qui se trouve ci-dessous 
```
#1 Location phpmyadmin
  location /phpmyadmin {
      root /usr/share/;
      index index.php index.html index.htm;

      location ~ ^/phpmyadmin/(.+\.php)$ {
          try_files $uri =404;
          root /usr/share/;
          fastcgi_pass unix:/var/run/php/php8.0-fpm.sock;
          fastcgi_index index.php;
          fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
          include /etc/nginx/fastcgi_params;
      }

      location ~* ^/phpmyadmin/(.+\.(jpg|jpeg|gif|css|png|js|ico|html|xml|txt))$ {
          root /usr/share/;
      }
  }

#2 Location phpmyadmin
  location /phpMyAdmin {
      rewrite ^/* /phpmyadmin last;
  }
```
5.3 - Une fois les locations ajoutées à votre fichier, relancez Nginx et corrigez quelques erreurs présentes sur votre PhpMyAdmin.
```
mkdir /usr/share/phpmyadmin/tmp
chmod 777 /usr/share/phpmyadmin/tmp
mv /usr/share/phpmyadmin/config.sample.inc.php /usr/share/phpmyadmin/config.inc.php
```

5.4 - Editez votre fichier de configuration PhpMyAdmin
```
nano /usr/share/phpmyadmin/config.inc.php 
```

- Modifiez `$cfg['blowfish_secret'] = '';` en y ajoutant une clé de 32 caractères (Exemple : `$cfg['blowfish_secret'] = 'Vs36nZu7A7Y2kg6c4DN88AuL4a25sF7N';`)
- Ajoutez a la fin de la page : `$cfg['TempDir'] = 'tmp';`

5.5 - Modifiez maintenant votre php.ini afin d'augmenter certaine limite (pour éviter des erreurs sur des importations de db trop grosse)
```
nano /etc/php/8.0/fpm/php.ini
```
- Modifiez :  `memory_limit = 128M` -> `memory_limit = 254M`
- Modifiez :  `post_max_size = 8M` -> `memory_limit = 50M`
- Modifiez :  `upload_max_filesize = 2M` -> `upload_max_filesize = 50M`

5.6 - Editez votre fichier de configuration principal de Nginx
```
nano /etc/nginx/nginx.conf
```
- **{http}** Ajoutez : `client_max_body_size 50M;`

5.7 - Pour finir redémarrez votre VPS et relancez Nginx
```
reboot
systemctl reload nginx
```

### 6 / Postfix Gmail
6.1 - Installez les paquets nécessaires pour configurer votre serveur de messagerie avec Postfix.
```
apt install postfix mailutils sasl2-bin libsasl2-2 libsasl2-modules
```

6.2 - Editez votre fichier main de postfix
```
nano /etc/postfix/main.cf
```
- Modifiez :  `relayhost =` -> `relayhost = [smtp.gmail.com]:587`
- Ajoutez a la fin de la page les lignes qui se trouve ci-dessous
```
  smtp_tls_security_level=secure (Peu être simplement a modifier car déjà présent)
  smtp_sasl_auth_enable = yes
  smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
  smtp_sasl_security_options = noanonymous
  smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
  smtp_use_tls = yes
```

6.3 - A présent, modifiez votre fichier `sasl_passwd`. Ce fichier est utilisé pour stocker les informations d'authentification SMTP nécessaires pour envoyer des emails via le serveur SMTP de Gmail (Notez qu'il faut remplacer `email@gmail.com` par votre email et `passwordApp` par le mot de passe qui vous a était génère dans "Mots de passe des applications") 
```
nano /etc/postfix/sasl_passwd
```
- Ajoutez : `[smtp.gmail.com]:587 email@gmail.com:passwordApp`

6.4 - Pour finir votre configuration de postfix, créez une table de hachage grâce à postmap sur votre fichier qui contient vos crédentials Gmail et redémarrez Postfix
```
postmap /etc/postfix/sasl_passwd
service postfix restart
```
6.5 - Vous pouvez à présent envoyer des mails depuis votre serveur ou depuis la fonction mail de PHP via votre site web. Pour vérifier depuis le serveur si vos e-mails s'envoient correctement, vous pouvez effectuer le test avec cette commande.
```
echo "Object mail" | mail -s "ceci est un email de test" destinataire@gmail.com
```

### 7 / Certbot
7.1 - Installez Certbot afin d'obtenir un certificat SSL pour votre domaine (Oubliez pas de modifier `domaine.com` par votre domaine)
```
apt install certbot python3-certbot-nginx -y
certbot --nginx -d domaine.com
```

### 8 / Socket.io && PM2
8.1 - Installez NodeJS & Socket.io afin de créer par la suite un Messenger en temps réel. Donc dans cette exemple nous allons créer un dossier `mychatapp` pour notre Messenger et y installé les paquets nécessaires.

```
apt install nodejs npm
cd /htdocs
mkdir mychatapp
cd mychatapp
npm init -y
npm install --save socket.io express
```

8.2 - Editez votre fichier de configuration par défaut de Nginx
```
nano /etc/nginx/conf.d/default.conf 
```
- **{server}** Ajoutez la location socket.io qui ce trouve ci-dessous (Modifiez `PORT` par le port que vous désirez)
```
#Location Socket.io
  location /socket.io {
  proxy_pass http://localhost:PORT;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  }
```
8.3 - Enfin, installez PM2 afin gérer et de maintenir vos processus Node.js
```
npm install pm2 -g
systemctl reload nginx
```
8.4 - Vous pouvez à présent gérer les processus Node.js à l'aide de PM2. Dans notre exemple le script s'appelle server.js
```
pm2 start server.js  // Démarre le serveur Node.js en utilisant le script server.js

pm2 list    // Vérifie si le serveur est en cours d'exécution.
pm2 stop server.js   // Arrête le serveur Node.js en cours d'exécution
pm2 restart server.js   // Redémarre le serveur Node.js en cours d'exécution
pm2 logs server.js  // Logs en temps réel du serveur Node.js en cours d'exécution
```

