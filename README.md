## Monitoring Fonctionnel

### Objectifs

L'objectif de l'exercice est de mettre en place un système de monitoring et de pouvoir être en mesure de prévenir les administrateurs de tout type de dysfonctionnement de service qui pourrait avoir un impact négatif pour les utilisateurs.

Pour cela, nous allons avoir deux sites web que nous allons surveiller, l'un sera sur la solution Magento et l'autre sera sur la solution Prestashop. Ce sont des outils pour mettre en place des marketplace, c'est pour cela que l'enjeu est double, car le temps que le site n'est pas fonctionnel, c'est de l'argent perdu pour le client/la boite.

Pour monitorer ces deux sites webs, nous utiliserons l'outil Zabbix qui est un outil de surveillance, qui nous permet de garder un œil vigilant sur nos systèmes et réseaux, assurant ainsi la stabilité et la performance de notre infrastructure informatique.

Nous allons dans un premier temps installer la solution Magento sur un serveur Debian.

## Installation de Magento

Pour commencer, nous allons nous connecter a notre serveur par le biais de ssh :

```text-plain
ssh user@hostname
```

Nous allons mettre a jours les packets de notre serveur, sur ce rapport nous utiliserons le gestionnaire de packet de Debian (APT) :

```text-plain
sudo apt-get update && sudo apt-get upgrade
```

Installer maintenant les packages que nous utiliserons lors de l'installation de Magento (vim est optionelle) :

```text-plain
sudo apt-get install git curl software-properties-common gnupg2 vim
```

### Installation de PHP

Rajouter les repository dans les sources de APT :

```text-plain
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/sury-php.list
wget -qO - https://packages.sury.org/php/apt.gpg | sudo apt-key add -
```

Mettre a jour le gestionnaire de package :

```text-plain
sudo apt-get update
```

Installation des packages PHP et les modules supplémentaire :

```text-plain
sudo apt install php8.1-{bcmath,common,curl,fpm,gd,intl,mbstring,mysql,soap,xml,xsl,zip,cli}
```

Maintenant nous allons configurer le fichier “php.ini” vous l'adapter a l'installation de Magento :

```text-plain
sudo sed -i "s/memory_limit = .*/memory_limit = 768M/" /etc/php/8.1/fpm/php.ini
sudo sed -i "s/upload_max_filesize = .*/upload_max_filesize = 128M/" /etc/php/8.1/fpm/php.ini
sudo sed -i "s/zlib.output_compression = .*/zlib.output_compression = on/" /etc/php/8.1/fpm/php.ini
sudo sed -i "s/max_execution_time = .*/max_execution_time = 18000/" /etc/php/8.1/fpm/php.ini
```

### Installation de Nginx

Installation du package :

```text-plain
sudo apt install nginx
```

Verifier que le serveur nginx est fonctionnelle :

```text-plain
sudo systemctl status nginx
```

Maintenant nous allons créer le fichier de configuration nginx pour la solution Magento :

```text-plain
sudo vim /etc/nginx/sites-available/magento.conf
```

Voila le fichier de configuration :

```text-plain
upstream fastcgi_backend {
    server unix:/run/php/php8.1-fpm.sock;
}
server {
    server_name web.magento;
    listen 80;
    
    set $MAGE_ROOT /var/www/html/magento;
    set $MAGE_MODE developer; # or production
    
    access_log /var/log/nginx/magento2-access.log;
    error_log /var/log/nginx/magento2-error.log;
    
    include /var/www/html/magento/nginx.conf.sample;
}
```

Verifier que le fichier de configuration a la bonne syntax :

```text-plain
sudo nginx -t
```

Maintenant créer un lien symbolique pour que nginx prenne en compte le fichier de configuration :

```text-plain
sudo ln -s /etc/nginx/sites-available/magento.conf /etc/nginx/sites-enabled/magento.conf
```

### Installation de MariaDB

Installation du package :

```text-plain
sudo apt install mariadb-server
```

Maintenant vous pouvez utiliser l'utilitaire fournit par Mysql pour configurer votre database :

```text-plain
sudo mysql_secure_installation
```

Utiliser cette commande pour rentrer dans le prompt de Mysql :

```text-plain
sudo mysql
```

Utiliser ces requêtes SQL :

```text-plain
CREATE USER 'magentouser'@'localhost' IDENTIFIED BY 'Str0n9PasSworD';
CREATE DATABASE magentodb;
GRANT ALL PRIVILEGES ON magentodb.* TO 'magentouser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Installation de Elastic Search

Installer les packages nécessaire :

```text-plain
sudo apt install apt-transport-https ca-certificates gnupg2
```

Rajout des dépots de la solution :

```text-plain
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo sh -c 'echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" > /etc/apt/sources.list.d/elastic-7.x.list'
```

Mise a jour et installation du package :

```text-plain
sudo apt update
sudo apt install elasticsearch
```

Activer maintenant Elasticsearch au démarrage de la machine :

```text-plain
sudo systemctl --now enable elasticsearch
```

Tester maintenant Elasticsearch :

```text-plain
curl -X GET "localhost:9200"
```

### Installation de Composer

Télécharger le gestionnaire d'installation :

```text-plain
curl -sS https://getcomposer.org/installer -o composer-setup.php
```

Installation de l'outil :

```text-plain
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

Verification du fonctionnement :

```text-plain
composer -V
```

### Installation de Magento

Installation des packages :

```text-plain
sudo apt install zip unzip
```

Création du projet Magento :

```text-plain
sudo composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition=2.4.4 /var/www/html/magento
```

Pour utiliser la solution vous devrez récupérer vos identifiants sur le site. Pour cela créer un compte sur le site de Magento Marketplace et une fois cela fait vous pourrez recuperer vos identifiants sur la page : [https://marketplace.magento.com/customer/accessKeys/](https://marketplace.magento.com/customer/accessKeys/)


Une fois recuperer a la racine de votre dossier Magento créer le fichier avec les valeurs que vous avez récupérer :

```text-plain
sudo vim auth.json
```

Le contenu du fichier :

```text-plain
{
  "http-basic": {
      "repo.magento.com": {
          "username":"<your public key>",
          "password":"<your private key>"
      }
  }
}
```

Vous pouvez désormais procéder a l'installation de Magento en utilisant cette commandes et en changeant les parametres :

```text-plain
sudo bin/magento setup:install \
--base-url=http://web.magento/ \
--db-host=127.0.0.1 \
--db-name=magentodb \
--db-user=magentouser \
--db-password=Str0n9PasSworD \
--admin-firstname=Admin \
--admin-lastname=User \
--admin-email=admin@your-domain.com \
--admin-user=admin \
--admin-password=Str0n9PasSworD \
--language=en_US \
--currency=USD \
--timezone=America/Chicago \
--use-rewrites=1
```

Changer les permission sur le dossier magento :

```text-plain
sudo chown -R www-data. /var/www/html/magento/
```

Pour utiliser des donnés factices utiliser cette commande :

```text-plain
sudo bin/magento sampledata:deploy
```

Vous pouvez maintenant utiliser Magento, a la fin de l'installation l'URL de la page Admin vous sera fournit.

Nous allons maintenant installer la solution Prestashop sur une seconde machine virtuelle.

### Installation de Prestashop

De la meme maniere connecter a votre serveur via SSH :

```text-plain
ssh user@hostname
```

Mettez a jour les packages et mise a jour du serveur :

```text-plain
sudo apt-get update
sudo apt-get upgrade
```

### Installation de MariaDB

Installation du package :

```text-plain
sudo apt-get install mariadb-server mariadb-client 
```

Maintenant vous pouvez utiliser l'utilitaire fournit par Mysql pour configurer votre database :

```text-plain
sudo mysql_secure_installation
```

Ouvrer maintenant mysql :

```text-plain
sudo mysql
CREATE DATABASE prestashop;
CREATE USER 'prestauser'@localhost IDENTIFIED BY ‘PrestaPassword’;
GRANT ALL PRIVILEGES ON prestashop.* TO 'prestauser'@localhost;
EXIT;
```

Configurer le pour qu'il ce lance au démarrage :

```text-plain
systemctl enable mariadb
```

### Installation de PHP

Installer les packages :

```text-plain
apt install -y php7.3 php7.3-mysql libapache2-mod-php php7.3-xml php7.3-gd php7.3-curl php7.3-intl php7.3-mbstring
```

Activer PHP et les modules nécessaire :

```text-plain
sudo a2enmod php7.3
sudo a2enmod rewrite
```

Activer apache et relancer le serveur :

```text-plain
systemctl enable apache2
sudo systemctl reload apache2
sudo systemctl restart apache2
```

### Installation de Prestashop

Installation de package :

```text-plain
sudo apt-get install unzip
```

Déplacer vous dans le dossier tmp pour éviter de laisser des fichiers sur votre serveur :

```text-plain
cd /tmp
```

Téléchargement de prestashop :

```text-plain
wget https://github.com/PrestaShop/PrestaShop/releases/download/1.7.8.10/prestashop_1.7.8.10.zip
```

Deziper le fichier dans le répertoire voulu :

```text-plain
unzip prestashop_1.7.8.10.zip
unzip prestashop.zip -d /var/www/html/prestashop/
```

Changer les droits sur le dossier pour pouvoir y acceder a partir du serveur apache :

```text-plain
sudo chown -R www-data: /var/www/html/prestashop/
```

Redémarrer le serveur pour prendre les modification en compte :

```text-plain
sudo systemctl restart apache2
```

Maintenant vous pouvez accéder a votre site via le lien : [https://hostname.com](https://hostname.com)

A partir de la vous pourrez installer prestashop sur votre machine.

Une fois l'installation faite, supprimer le dossier install dans le repertoire prestashop.

### Installation de Zabbix

Mise a jour de votre serveur :

```text-plain
sudo apt update
```

Installation de mysql :

```text-plain
sudo apt install mysql-server
```

Lancer le gestionnaire d'installation :

```text-plain
mysql_secure_installation
```

Lancer le service en question :

```text-plain
sudo systemctl start mysql.service
```

Télécharger la bonne version de Zabbix en fonction de votre environnement :

```text-plain
wget https://repo.zabbix.com/zabbix/6.4/ubuntu-arm64/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb
```

Faite encore une fois une mise a jour avec le nouveau paquet installer :

```text-plain
sudo apt update
```

Installation des packets :

```text-plain
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts zabbix-agent
```

Ouvrer mysql :

```text-plain
sudo mysql
```

Utiliser ces requetes SQL :

```text-plain
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by 'password';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;
```

Utiliser cette commande pour importer les données sql, un mot de passe vous sera demandé :

```text-plain
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

Ouvrer encore une fois mysql :

```text-plain
sudo mysql
```

Et taper ces requetes :

```text-plain
set global log_bin_trust_function_creators = 0;
quit;
```

Configurer le fichier de configuration de zabbix :

```text-plain
sudo vim /etc/zabbix/zabbix_server.conf
```

Et changer cette valeur (elle sera commenté normalement) :

```text-plain
DBPassword=password
```

Maintenant vous pouvez configurer votre fichier de configuration nginx :

```text-plain
sudo vim /etc/zabbix/nginx.conf
```

Changer ces valeurs la :

```text-plain
listen 8080;
server_name example.com;
```

Relancer les services et activer les au démarrage :

```text-plain
sudo systemctl restart zabbix-server zabbix-agent nginx php8.1-fpm
sudo systemctl enable zabbix-server zabbix-agent nginx php8.1-fpm
```

Vous pouvez désormais accéder a zabbix en utilisant l'url de votre serveur.

## Monitoring

### Installation d’un système de surveillance et mesure des temps de réponse

Dans le cadre de l'exercice nous allons vérifier pour chacune des deux solutions :

-   Accès fonctionnel à la homepage (code HTTP 200OK)
-   Accès à une page catégorie (code HTTP 200OK)
-   Accès à une page produit (code HTTP 200OK)

Pour cela dans Zabbix dans un premier temps nous allons aller sur l'onglet Monitoring et rajouter nos deux serveurs.

Dans la partie Host → Create Host

Une fois les deux serveurs ajouté, nous allons cliquer sur le nom du serveur et ensuite nous allons sélectionner dans la partie “Configuration” → “Web”.

Une fois a cette endroit, nous allons cliquer sur le bouton “Create Web Scenario”. L'étape importante ici et d'aller dans l'onglet “Step”, et de configurer une étape qui ira checker si la page est disponible.

Une fois terminer, repeter l'opération pour les trois autres pages.

### Surveillance du tunnel de vente

Nous allons maintenant poursuivre l'exercice, en fessant une surveillance plus dynamique en vérifiant que le contenu de la page est le bon.

-   L'accès à la homepage en vérifiant qu’une chaîne de caractères est bien présente dans la page (ex: le titre de la page)
-   L'accès à une page catégorie en vérifiant qu’une chaîne de caractères est bien présente dans la page (ex: le nom de la catégorie ou le nom d’un produit affiché dans cette page)
-   L'accès à une page produit en vérifiant qu’une chaîne de caractères est bien présente dans la page (ex: une partie de la description du produit dans la page)
-   Le bon fonctionnement de l’ajout au panier. Pour cette étape, vous devez choisir un produit en stock et configurer votre système de surveillance pour effectuer la mise en panier et vérifier que le produit est bien dans le panier de l’utilisateur après l’ajout.

Pour cela nous allons de la meme maniere créer des “Web Scenario”, l'idée ici sera de rajouter dans l'onglet Step, une etape qui check une string sur la page. Repeter l'opération pour les 3 autres pages.

### Configuration d'alertes en cas de panne

Nous devons :

-   Paramétrer votre solution de monitoring (Zabbix ou Centreon) de façon à pouvoir émettre des alertes afin de notifier une personne via email.
-   Configurer les notifications afin qu'elles soient ré-émises périodiquement si le problème persiste.
-   Mettre en place de l'escalade de notification, c'est-à-dire pouvoir prévenir davantage de personnes si la panne dure plus d'un certain temps.

Pour cela nous allons configurer ce qu'il s'appelle des “Trigger”.

De la meme maniere sur la section “Hosts", cliquer sur le nom du serveur et sélectionner dans la partie “Configuration” l'item “Trigger”.

Vous serez dans la possibilité d'appuyer sur le bouton en haut gauche “Create Trigger”. Une fois sur cette fenêtre vous trouverez la partie “Expression", c'est ici que vous aller définir a quel moment le trigger sera activer. Dans notre cas nous utiliserons les actions que nous avons créer précédemment et nous allons comparer les valeurs pour voir si l'action effectuer est fonctionnelle. Si ce n'est pas le cas, le trigger s'activera.

Tout seule, dans notre cas les trigger ne seront pas d'une utilité énorme. Pour cela nous allons configurer les alertes pour qu'elle s'envoie par mail et qu'il y a une escalade si personne ne résout le probleme a temp.

Pour cela, dans Zabbix aller dans l'onglet “Alerts” → “Actions” → “Trigger Action”.

Une fois a cette endroit, en haut a gauche vous pouvez appuyer sur le bouton en haut a droite “Create Action”. L'idée ici est de configurer l'action donc deja dans les conditions sélectionner notre “Trigger”. Une fois cela dans le second d'onglet “Operations" vous serez en mesure de configurer les marches a suivre si le trigger a était déclenché.
