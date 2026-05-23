
# 🦖 Installation de Pterodactyl en français

[![Discord](https://img.shields.io/discord/1352228798585638983?label=&logo=discord&logoColor=ffffff&color=7389D8&labelColor=6A7EC2)](https://discord.gg/4fk7jZvKKw)

Ce guide résume en français les points importants de la documentation officielle de **Pterodactyl**, un panel open-source de gestion de serveurs de jeux. Pterodactyl permet d'administrer des serveurs comme Minecraft, Rust, Terraria, Teamspeak, Garry's Mod, ARK et beaucoup d'autres grâce à des conteneurs Docker isolés.

Le projet repose principalement sur deux composants :

- 🖥️ **Panel** : interface web d'administration, développée avec PHP et React.
- ⚙️ **Wings** : daemon installé sur les machines qui exécutent les serveurs de jeux, développé en Go et basé sur Docker.

> ℹ️ Ce README ne remplace pas la documentation officielle. Il sert de guide rapide en français et renvoie vers les pages officielles dès qu'un détail dépend de votre système.

## 📚 Documentation officielle

- [Introduction officielle](https://pterodactyl.io/project/introduction.html)
- [Installation du Panel](https://pterodactyl.io/panel/1.0/getting_started.html)
- [Configuration du serveur web](https://pterodactyl.io/panel/1.0/webserver_configuration.html)
- [Installation de Wings](https://pterodactyl.io/wings/1.0/installing.html)
- [Certificats SSL](https://pterodactyl.io/tutorials/creating_ssl_certificates.html)
- [Configuration MySQL/MariaDB](https://pterodactyl.io/tutorials/mysql_setup.html)

## ✅ Avant de commencer

Vous devez avoir :

- 🐧 un serveur Linux avec accès `root` ou `sudo` ;
- 🌐 un nom de domaine pointant vers le serveur, surtout si vous activez HTTPS ;
- 🧠 des bases solides en administration Linux ;
- 📦 un hébergement compatible Docker, idéalement KVM ou serveur dédié ;
- 💾 une sauvegarde de vos données importantes avant toute installation.

Pterodactyl ne fonctionne pas sous Windows pour Wings. Les environnements OpenVZ, OVZ, Virtuozzo ou certains LXC posent souvent problème avec Docker.

Pour vérifier la virtualisation :

```bash
systemd-detect-virt
```

Si le résultat indique `openvz` ou `lxc`, demandez confirmation à votre hébergeur avant de continuer.

## 🧩 Systèmes supportés

D'après la documentation officielle actuelle, le Panel et Wings fonctionnent sur les distributions suivantes.

| Système | Versions supportées | Notes |
| --- | --- | --- |
| Ubuntu | 22.04, 24.04 | Ubuntu 22.04 demande des dépôts supplémentaires pour PHP. |
| Debian | 11, 12, 13 | Dépendances Debian documentées officiellement. |
| RHEL / Rocky Linux / AlmaLinux | 8, 9 | Des dépôts supplémentaires peuvent être nécessaires. |

Wings mentionne aussi Ubuntu 20.04 comme supporté, mais pour une nouvelle installation complète il vaut mieux partir sur une version récente et maintenue.

## 📦 Dépendances principales

Pour le **Panel** :

- PHP `8.2` ou `8.3` recommandé ;
- extensions PHP : `cli`, `openssl`, `gd`, `mysql`, `PDO`, `mbstring`, `tokenizer`, `bcmath`, `xml` ou `dom`, `curl`, `zip`, et `fpm` avec NGINX ;
- MySQL `5.7.22+` ou MySQL `8`, ou MariaDB `10.2+` ;
- Redis ;
- un serveur web : NGINX, Apache ou Caddy ;
- `curl`, `tar`, `unzip`, `git` ;
- Composer v2.

Pour **Wings** :

- Docker ;
- `curl` ;
- un noyau Linux compatible Docker ;
- un certificat SSL si le Panel utilise HTTPS.

## 🖥️ Installation officielle du Panel

Les commandes exactes peuvent varier selon la distribution. Consultez toujours la page officielle avant de les lancer.

### 1. 📦 Installer les dépendances

Exemple pour une base Debian/Ubuntu :

```bash
apt update
apt -y install software-properties-common curl apt-transport-https ca-certificates gnupg tar unzip git redis-server nginx mariadb-server
```

Installez ensuite PHP `8.2` ou `8.3` et les extensions nécessaires selon votre système. La documentation officielle donne un exemple complet pour Ubuntu avec les dépôts PHP adaptés.

Installez Composer :

```bash
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
```

### 2. ⬇️ Télécharger le Panel

```bash
mkdir -p /var/www/pterodactyl
cd /var/www/pterodactyl

curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
tar -xzvf panel.tar.gz
chmod -R 755 storage/* bootstrap/cache/
```

### 3. 🗄️ Préparer la base de données

Connectez-vous à MariaDB ou MySQL :

```bash
mariadb -u root -p
```

Créez la base et l'utilisateur en remplaçant `mot_de_passe_solide` par un mot de passe unique :

```sql
CREATE USER 'pterodactyl'@'127.0.0.1' IDENTIFIED BY 'mot_de_passe_solide';
CREATE DATABASE panel;
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'127.0.0.1' WITH GRANT OPTION;
exit
```

### 4. ⚙️ Configurer l'application

```bash
cd /var/www/pterodactyl

cp .env.example .env
COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader
php artisan key:generate --force

php artisan p:environment:setup
php artisan p:environment:database
php artisan p:environment:mail
php artisan migrate --seed --force
php artisan p:user:make
```

Sauvegardez impérativement la clé `APP_KEY` du fichier `.env`. Sans cette clé, certaines données chiffrées deviennent irrécupérables, même avec une sauvegarde de la base de données.

```bash
grep APP_KEY /var/www/pterodactyl/.env
```

### 5. 🔐 Corriger les permissions

Avec NGINX, Apache ou Caddy sur Debian/Ubuntu :

```bash
chown -R www-data:www-data /var/www/pterodactyl/*
```

Sur RHEL, Rocky Linux ou AlmaLinux, l'utilisateur peut être `nginx` ou `apache` selon votre serveur web.

### 6. 🔁 Configurer les tâches de fond

Ajoutez le cron suivant avec `sudo crontab -e` :

```cron
* * * * * php /var/www/pterodactyl/artisan schedule:run >> /dev/null 2>&1
```

Créez ensuite le service systemd `pteroq.service` selon l'exemple officiel, puis activez-le :

```bash
sudo systemctl enable --now redis-server
sudo systemctl enable --now pteroq.service
```

Sur RHEL, Rocky Linux ou AlmaLinux, le service Redis peut s'appeler `redis.service` au lieu de `redis-server.service`.

### 7. 🌍 Configurer le serveur web

La configuration dépend fortement de votre choix : NGINX, Apache ou Caddy, avec ou sans SSL.

La documentation officielle fournit les blocs de configuration à copier et adapter : [configuration du serveur web](https://pterodactyl.io/panel/1.0/webserver_configuration.html).

Si vous utilisez NGINX avec SSL, vous devez créer les certificats avant de redémarrer NGINX. Caddy peut gérer automatiquement les certificats si sa configuration est correcte.

## 🪽 Installation officielle de Wings

Wings doit être installé uniquement avec Pterodactyl `1.x`.

### 1. 🐳 Installer Docker

Installation rapide proposée par la documentation :

```bash
curl -sSL https://get.docker.com/ | CHANNEL=stable bash
sudo systemctl enable --now docker
```

Vous pouvez aussi suivre la [documentation officielle Docker](https://docs.docker.com/engine/install/) pour une installation manuelle.

### 2. ⬇️ Télécharger Wings

```bash
sudo mkdir -p /etc/pterodactyl
curl -L -o /usr/local/bin/wings "https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_$([[ "$(uname -m)" == "x86_64" ]] && echo "amd64" || echo "arm64")"
sudo chmod u+x /usr/local/bin/wings
```

### 3. 🧭 Créer le node dans le Panel

Dans l'interface administrateur du Panel :

1. ouvrez **Nodes** ;
2. créez un nouveau node ;
3. ouvrez l'onglet **Configuration** ;
4. copiez le contenu dans `/etc/pterodactyl/config.yml`, ou utilisez la commande générée par le Panel.

Si le Panel est en HTTPS, Wings doit aussi disposer d'un FQDN et de certificats SSL valides.

### 4. 🧪 Tester Wings

```bash
sudo wings --debug
```

Si aucun problème n'apparaît, arrêtez le processus avec `CTRL+C`, puis créez le service systemd `wings.service` comme indiqué dans la documentation officielle.

Activez ensuite Wings :

```bash
sudo systemctl enable --now wings
```

### 5. 🔌 Ajouter les allocations

Une allocation est une combinaison IP + port assignable à un serveur de jeu.

Pour trouver l'adresse IP principale :

```bash
hostname -I | awk '{print $1}'
```

N'utilisez pas `127.0.0.1` comme allocation. Ajoutez les allocations depuis **Nodes > votre node > Allocation** dans le Panel.

## 🔥 Ports courants à ouvrir

Adaptez les ports à vos besoins réels et à votre pare-feu.

| Usage | Ports courants |
| --- | --- |
| Panel HTTP | `80/tcp` |
| Panel HTTPS | `443/tcp` |
| Wings SFTP | `2022/tcp` par défaut |
| Wings API | `8080/tcp` par défaut |
| Serveurs Minecraft | `25565/tcp` par défaut |

Exemple `iptables` à adapter, uniquement si vous savez ce que vous faites :

```bash
sudo iptables -A INPUT -p tcp -m multiport --dports 80,443,2022,8080,25565 -j ACCEPT
sudo iptables-save > /etc/iptables.rules
```

## ⚡ Option rapide : script communautaire

Il existe un script communautaire qui automatise une partie de l'installation :

```bash
bash <(curl -s https://pterodactyl-installer.se)
```

Ce script n'est pas la méthode officielle de Pterodactyl. Avant de l'utiliser, lisez son dépôt, comprenez ce qu'il modifie sur votre serveur et vérifiez qu'il supporte votre distribution : [pterodactyl-installer](https://github.com/pterodactyl-installer/pterodactyl-installer).

Pour une installation Panel + Wings via ce script, l'option habituellement utilisée est l'installation combinée du Panel et de Wings. Répondez attentivement aux questions sur la base de données, le domaine, le pare-feu, HTTPS et Let's Encrypt.

## ⚠️ Conseils importants

- 🧠 Ne lancez pas de commandes serveur sans les comprendre.
- 🔑 Sauvegardez la clé `APP_KEY`, la base de données et les fichiers de configuration.
- 🔒 Utilisez HTTPS en production.
- ♻️ Gardez Docker, Wings, le Panel et le système à jour.
- 🧾 Vérifiez les logs du Panel, de NGINX/Apache/Caddy, de Redis, de Docker et de Wings en cas de problème.
- 📚 Préférez les liens officiels pour les détails qui changent souvent.

## 🔗 Ressources utiles

- [Site officiel de Pterodactyl](https://pterodactyl.io/)
- [Documentation officielle](https://pterodactyl.io/project/introduction.html)
- [GitHub du Panel](https://github.com/pterodactyl/panel)
- [GitHub de Wings](https://github.com/pterodactyl/wings)
- [Eggs officiels](https://eggs.pterodactyl.io/)
- [Eggs communautaires](https://pterodactyleggs.com/)
- [Discord officiel de Pterodactyl](https://discord.gg/pterodactyl)
- [Script communautaire d'installation](https://github.com/pterodactyl-installer/pterodactyl-installer)
