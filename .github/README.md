# Magedock

An ecommerce-friendly Docker powered enviroment.

## Changelog

 * 2019-09-03: Up-to-date with changes added in Laradock v8.0.

## About

This is a Docker powered environment based on [Laradock](http://laradock.io/), aimed to provide a suitable workspace for Magento and Symfony based solutions.

## What's different?

 * Working *nginx* configuration files for Magento 2 and Symfony 4.
 * (Real) UTF-8 in MySQL ([using utf8mb4](https://medium.com/@adamhooper/in-mysql-never-use-utf8-use-utf8mb4-11761243e434))
 * SOAP and XSL extensions installed by default.
 * Node "LTS" version installed by default (not *latest*).
 * `vue-cli` not installed by default.
 * `rollup` installed by default.
 * `letsencrypt` installed on `workspace` container.
 * [Sonic](https://crates.io/crates/sonic-server) service (`docker-compose up sonic`).
 * [NSQ](https://nsq.io/) services (`docker-compose up nsq nsqadmin`).

This repo also contains an extensive documentation on how to get your project up and running as fast as possible.

## Setup

### Docker

As usual, make sure your environment already has Docker installed along with docker-composer. Make sure your user was also added to the *docker* group:

```
sudo usermod -aG docker $USER
```

You might need to restart your system after that. Generate a new `.env` file and apply your custom configuration.

```
cp env-example .env
```

Start your containers.

```
docker-compose up -d nginx mysql phpmyadmin
```

It is advisable that you avoid login into the *workspace* container as *root*, unless you're doing administration tasks. You can login using the `laradock` user account doing the following:

```
docker-compose exec --user laradock workspace bash
```

This user is able to run `artisan`, `bin/console` and `bin/magento` without problems an it's generally a good way to avoid issues.
To logout, just enter `exit`. You can stop all containers by doing `docker-compose down`.

### MySQL

Since version 8.0, MySQL uses *caching_sha2_password* as the authorization plugin. Trying to log in as *root* using the usual method will fail. You can bypass that by doing:

```
docker-compose exec mysql mysql -u root -p
```

This repo includes a way for letting you add a new MySQL user on startup. Find the file located in `mysql/docker-entrypoint-initdb.d/createuser.sql.example`, rename it to `createuser.sql` and apply the changes you consider necessary.

### Adding locales

Any locales you need to add to your setup must be included on both the `php-fpm` and `workspace` containers. The example below adds the `es_AR` locale.

```
RUN apt-get update && apt-get install -y \
    locales \
    && echo '' >> /usr/share/locale/locale.alias \
    && sed -i 's/# es_AR.UTF-8 UTF-8/es_AR.UTF-8 UTF-8/' /etc/locale.gen \
    && locale-gen
```

Add the locales before the call to `locale-gen` in both Dockerfiles. Save and then rebuild the containers.

## Laravel projects

This tutorial will guide you throughout the process of setting up a Laravel project.

### Pulling an existing project

If you're initializing a project from an existing repo then you don't need to do all steps. First, move to your projects folder and clone the repo.

```
git clone [project-repository]
```

#### Installing dependencies

Initialize your containers and log in as `laradock`:

```
docker-compose up -d mysql nginx
docker-compose exec --user laradock workspace bash
```

Then, move to the project folder and run both `composer install` and `npm install` (you can alternatively use `yarn`):

```
composer install && npm install
```

#### Configuration file

Create a new `.env` file:

```
cp .env.example .env
```

Generate the secret key:

```
php artisan key:generate
```

Update the database credentials:

```
DB_HOST=mysql
DB_USERNAME=db_user
DB_PASSWORD=db_password
```

#### Server configuration

Exit `laradock` and move to the `nginx/sites` folder.

```
exit
cd nginx/sites
```

Create a configuration file for your application:

```
cp laravel.conf.example laravel.conf
```

Modify the file so it specified the folder where the project is located. Set its local domain. Don't forget to modify your local `/etc/hosts`:

```
127.0.0.1   your-laravel-app.test
```

Move to the `laradock` folder and run `docker-compose`:

```
docker-compose up -d nginx mysql phpmyadmin
```

### Fresh install

Enter to your workspace:

```
docker-compose exec --user laradock workspace bash
```

We'll create a new Laravel project using the `create-project` utility provided by Composer. The next command will create a new project based on the 5.7 branch:

```
composer create-project laravel/laravel my-project "5.7.*" --prefer-dist
```

#### Installing dependencies

Composer will install all its dependencies automatically and it will also create a default `.env` file. Now you'll need to install all frontend dependencies (you can use `yarn` instead of `npm`):

```
npm install
```

#### Environment file

Open the `.env` file and modify the `DB_HOST` so it connects to the `mysql` container:

```
DB_HOST=mysql
```

Change the credentials as well:

```
DB_USERNAME=db_user
DB_PASSWORD=db_password
```

#### Server setup

Open a terminal in your local machine and modify the `/etc/hosts` file so it includes your new application's domain:

```
127.0.0.1  my-project.test
```

Switch to the `nginx/sites/` folder. Create a new configuration file based on the example:

```
cp laravel.conf.example my-project.conf
```

You'll have to modify the new file so it specifies the new domain and the folder where it is located. Once you finish those changes, save the file.
Finally, reset your containers:

```
docker-compose down
docker-compose -up mysql nginx
```

Open a browser and open the URL you entered on the configuration file. The default Laravel Welcome page should appear.

#### Initialize repository

Initialize your repo and set the origin (use the SSH repo):

```
git init
git remote add origin [your-repository]
```

Add your files, commit and push:

```
git add .
git commit -m "first commit"
git push origin master
```

## Symfony 4 projects

### Fresh install

Start your containers:

```
docker-compose up -d nginx mysql
```

Login as laradock and create a new Symfony 4 project using *Composer*:

```
docker-compose exec --user laradock workspace bash
composer create-project symfony/skeleton symfony
```

Install dependencies:

```
cd symfony
composer install
```

Add version control:

```
git init
git init .
git commit -m "first commit"
git remote add origin [your-repo]
git push origin master
```

Customize you `.env` file. Exit and stop your containers.

```
exit
docker-compose down
```

Setup your newly created site by copying the Symfony 4 example config.

```
cd nginx/sites
cp symfony4.conf.example symfony.conf
```

Open this new file and adjust the local domain:

```
    server_name symfony.test;
```

Add the local domain to your local `etc/hosts`:

```
127.0.1.1	symfony.test
```

Restart your containers. You should be able to see the Symfony welcome page.

## Magento 2 projects

### Installing Magento from Packagist

If you already have a Magento Marketplace account (https://marketplace.magento.com/), you can download all Magento packages directly from Packagist. Before continuing you'll have to activate authentication through *Composer*. Find this line on you `.env` file:

```
WORKSPACE_COMPOSER_AUTH=false
```

Switch the value to `true`:

```
WORKSPACE_COMPOSER_AUTH=true
```

Now update the file located in `workspace/auth.json` and enter you credentials. These are both the *Public Key* and *Private Key* located in the **Access Keys** section of you Magento Marketplace account. After that, your file should look similar to this:

```json
{
    "http-basic": {
        "repo.magento.com": {
            "username": "your_username",
            "password": "your_password"
        }
    }
}
```

Activating authentication requires rebuilding the `workspace` container.

```bash
docker-compose build workspace
```

Now start your containers as usual and login.

```
docker-compose up -d nginx mysql phpmyadmin
docker-compose exec --user laradock workspace bash
```

Now create a new Magento 2 project. The following command will create a `magento2` folder and download all source from the Magento 2 repository.

```
composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition magento2
```

If you need to create a new project using a specific branch of Magento, you need to include it as part of the command. For example, creating a new project using Magento 2.2.8 should look like this:

```
composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition=2.2.8 magento2
```

### Alternative: Installing Magento from archive

Magento 2 is also distributed as a `.zip` and `.tar.gz`. We can download this file to our workspace to start the installation process. First, login as `laradock`:

```
docker-compose up -d --build nginx mysql workspace
docker-compose exec --user laradock workspace bash
```

Now, download Magento 2 codebase using `curl`. The file to download will depend on the version you want to run.

```
curl -L https://github.com/magento/magento2/archive/2.3.2.tar.gz -o magento2.tar.gz
```

Unzip the file. Files are extracted to a `magento2-2.x.x` folder. Rename it if needed.

```
tar -xzvf magento2.tar.gz
mv magento2-2.3.2/ magento2/
```

Install dependencies through *Composer*:

```
cd magento2/
composer install
```

### Create Magento 2 database

Before continuing, use another CLI to connect to your running MySQL container and create an empty `magento2` database.

```
mysql -h 127.0.0.1 -u magedock -p
create database magento2;
exit
```

### Server configuration

Magento 2 already includes some `nginx` configuration files that we'll to import from our config. If your Magento 2 installation is not located on the `magento2` folder then you'll need to adjust the newly created configuration file by hand.

```
docker-compose down
cd nginx/sites
cp magento2.conf.example magento2.conf
```

Restart the containers to load you new configuration.

```
docker-compose up -d nginx mysql
```

### Running the installer

Start the installation process. Change the arguments in case you need to.

```
docker-compose exec --user laradock workspace bash
cd magento2
./bin/magento setup:install --base-url=http://magento2.test/ --db-host=mysql --db-name=magento2 --db-user=magedock --db-password=your_password --admin-firstname=admin --admin-lastname=admin --admin-email=admin@admin.com --admin-user=admin --admin-password=admin_secret --language=en_US --currency=USD --timezone=America/Chicago --use-rewrites=1
```

Pay close attention to the output as it will contain the Admin URL. Save it so you can use it later. Finally, activate the *developer mode*:

```
php bin/magento deploy:mode:set developer
```

Try accessing your application's URL through a browser adding the route given to you at the end of the installation process. Login using the credentials you specified.

### Setup Magento 2 crontab

Create a new file inside the `workspace/crontab` folder that identifies the application you're adding the crontab for:

```
touch workspace/crontab/magento2
```

Open this file on an text editor and paste the following:

```
* * * * * laradock /usr/bin/php /var/www/magento2/bin/magento cron:run | grep -v Ran jobs by schedule >> /var/www/magento2/var/log/magento.cron.log
* * * * * laradock /usr/bin/php /var/www/magento2/bin/magento setup:cron:run >> /var/www/magento2/var/log/setup.cron.log
```

Adjust the path to the magento script. Rebuild your `workspace` and restart your containers.

```
docker-compose down
docker-compose build workspace
docker-compose up -d nginx mysql
```

After that, try logging into you Magento application as an admin. You shouldn't see any warning messages about invalid indexes now.

### Multiple instances

If you try to run multiple instances of Magento 2 on the same workspace you may receive an error message from nginx. If that's the case, try removing the following configuration section from the new instance:

```
upstream fastcgi_backend {
    server  php-fpm:9000;
}
```

Restart your containers. As long you keep that configuration on another Magento app you shouldn't run into any errors.

### Using an existing database

If Magento was installed on another computer, instead of reinstalling the application, try copying the files `config.php` and `env.php` that were generated after the installation to you local `app/etc` folder. If you're also porting the database, right after you finished migrating, open the database manager of your preference and locate the configuration values with the following paths: `web/secure/base_url` and `web/unsecure/base_url`. Update the values so they correspond to your local domains.

### Bash aliases

A set of additional aliases is also provided to simplify some common tasks during the development process. These are available once you login into the `workspace` container and switch to the Magento project folder.

 * `m2cl`: Runs `php bin/magento cache:clean`.
 * `m2fl`: Runs `php bin/magento cache:flush`.
 * `m2modeset`: Runs `php bin/magento deploy:mode:set`. You'll need to call it specifying the mode to use (`m2modset developer`).
 * `m2enable`: Runs `php bin/magento module:enable`. You'll have to provide the module name (`m2enable Vendor_CustomSearch`).
 * `m2disable`: Runs `php bin/magento module:disable`. You'll have to provide the module name (`m2disable Vendor_CustomSearch`).
 * `m2up`: Runs `php bin/magento setup:upgrade`. If a module name is provided then it will try to enable first (`m2up Vendor_CustomSearch`).
 * `m2compile`: Runs `php bin/magento setup:di:compile` and cleans cache afterwards.
 * `m2dep`: Runs `php bin/magento setup:static-content:deploy`.
 * `m2depf`: Same as above but including `-f` to force asset compilation.
 * `m2reindex`: Runs `php bin/magento indexer:reindex`.
 * `m2i18n`: Runs `php bin/magento i18n:collect-phrases`. You'll need to include the module/theme path and the output file (`m2i18n app/design/frontend/Vendor/theme --output="app/design/frontend/Vendor/theme/i18n/es_AR.csv"`).
 * `m2tclean`: Removes any compiled asset for the given theme. Must be called including the theme path. Can remove assets for different locales (`m2tclean Vendor/theme es_AR`).
 * `m2tcl`: Same as above but calling `php bin/magento setup:static-content:deploy -f` and `php bin/magento cache:clean` afterwards. Use this during theme development to reload your styles (`m2tcl Vendor/theme en_US es_AR`).
 * `m2lgrep`: Runs grep on all `.less` files inside the Magento default themes. Useful for searching LESS variables by name. Example: `m2lgrep @copyright__background-color`.
 * `m2tgrep`: Runs grep on all `.xml`, `.phtml` and `.html` files inside the Magento default themes. You can specify an optional subfolder. Example: `m2tgrep title Magento_Customer`.
 * `m2mgrep`: Runs grep on all `.xml`, `.phtml` and `.html` files inside the given module. Module name must be in snake-case. Example: `m2mgrep price catalog-search`.

## Shopify projects

There's already a Laravel extension for working with Shopify. You can check it [here](https://github.com/ohmybrew/laravel-shopify).

### Polaris setup

Keep in mind that if you're using [Polaris](https://github.com/Shopify/polaris) inside your project you won't be able to run most of the examples that the site provides right away. This is because some examples use syntax that is not supported by default (ES7). If you try to transpile some of those examples you might receive an error. To avoid that, do the following.

Make sure you already have React instead of Vue:

```
php artisan preset react
npm install
```

Install Polaris:

```
npm install @shopify/polaris --save
```

To build projects using Babel you'll need to add some extra dependencies.

#### Babel 7.x

Since Babel 7, support for stage presets have been dropped. Make sure to add these dependencies to your package.json:

```
"@babel/core": "^7.0.0",
"@babel/preset-env": "^7.0.0",
"@babel/preset-react": "^7.0.0",
"@babel/plugin-proposal-class-properties": "^7.0.0",
"@babel/plugin-proposal-decorators": "^7.0.0",
"@babel/plugin-proposal-do-expressions": "^7.0.0",
"@babel/plugin-proposal-export-default-from": "^7.0.0",
"@babel/plugin-proposal-export-namespace-from": "^7.0.0",
"@babel/plugin-proposal-function-bind": "^7.0.0",
"@babel/plugin-proposal-function-sent": "^7.0.0",
"@babel/plugin-proposal-json-strings": "^7.0.0",
"@babel/plugin-proposal-logical-assignment-operators": "^7.0.0",
"@babel/plugin-proposal-nullish-coalescing-operator": "^7.0.0",
"@babel/plugin-proposal-numeric-separator": "^7.0.0",
"@babel/plugin-proposal-optional-chaining": "^7.0.0",
"@babel/plugin-proposal-pipeline-operator": "^7.0.0",
"@babel/plugin-proposal-throw-expressions": "^7.0.0",
"@babel/plugin-syntax-dynamic-import": "^7.0.0",
"@babel/plugin-syntax-import-meta": "^7.0.0",
```

Then, create a `.babelrc` file with the following content:

```
{
    "presets": [
        "@babel/preset-env",
        "@babel/preset-react"
    ],
    "plugins": [
      "@babel/plugin-proposal-class-properties"
    ]
}

```

Finally, to add styles you can either inport the CSS in your page:

```html
<link rel="stylesheet" href="https://sdks.shopifycdn.com/polaris/2.11.0/polaris.min.css" />
```

Or import the SCSS in your `app.scss`:


```scss
import '@shopify/polaris/styles.css';
```

Check that everything is working by running the following command:

```
npm run dev
```

### Using ngrok

In order to run your app, you'll have to generate a secure remote URL for it. That way, it will be accessible as an embedded app through Shopify. To achieve this you can use ngrok (https://ngrok.com/). Go to the website and start by creating an account. Once done, download the software and unzip it. It is a good idea to make this executable globally available so it can be executed from any folder.

```
sudo mv ngrok /usr/local/bin
```

On the website you'll find a special command that initializes the authetication process. You'll have to run this command to generate a special session file to tell `ngrok` who you are.

Once ready, execute `ngrok` by providing the local domain where your app is running.

```
ngrok shopifyapp.test:80
```

By default, `ngrok` will not route to the HTTPS port. For that to happen, include the protocol as part of the command like this:

```
ngrok http https://my-shopify.app.test
```

`ngrok` will generate a new domain for your app. Copy it and then kill your running containers. Open the server configuration that you already created for your app and then search for this line:

```
    server_name my-shopify-app.test;
```

We're going to add an additional alias for this local domain. For example, if the URL generated on the previous step was `fedcba98.ngrok.io`, then you'll replace the above line with the following:

```
    server_name my-shopify-app.test fedcba98.ngrok.io;
```

Make sure the server also includes the following lines for configuring the SSL certificates:

```
 listen 443 ssl;
 ssl_certificate /etc/nginx/ssl/default.crt;
 ssl_certificate_key /etc/nginx/ssl/default.key;
```

Save the file and then start your containers. Go to the *Partners* section in Shopify, log in and create your new app. When you're asked by the app URL, just enter the one generated by `ngrok`. Laravel will map all the traffic to this domain.

## Let's Encrypt

### Generating certificates for nginx using certbot

Login into your workspace as `laradock` and create the following folder structure inside `var/www`.

```
docker-compose exec --user laradock workspace bash
cd /var/www
mkdir -p letsencrypt/.well-known/acme-challenge
```

We will be using this folder to validate our site against Let's Encrypt. But before that, we need to make a few changes in our workspace.

```
exit
docker-compose down
```

Before creating the new certificates, we'll add a new cronjob that invokes the renewal process. Add a new file inside `workspace/crontab` called `certrenew`. Add the following line:

```
0 */12 * * * root bash -c "source /root/aliases.sh && certrenew -c >> /dev/null 2> /var/certs/certrenew.log"
```

The `certrenew` command is added as an alias of `certbot renew`. The `-c` flag is added so new certificates are copied to the default directory (`/var/certs`). Now rebuild the workspace container.

```
docker-compose build workspace
```

Find your site's configuration file inside the `nginx/sites` folder and add the following:

```
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/letsencrypt;
    }

    location = /.well-known/acme-challenge/ {
		    return 404;
    }
```

These lines are required to validate the authorization process. Once you try to generate the new certificates, Let's Encrypt will make a request to `http://your-domain/.well-known/acme-challenge/RANDOM_STRING`. Here we are instructing our server to go find that content on the folder we created on the first step.

Now initialize the `nginx` container. Then, log into the `workspace` container as root:

```
docker-compose up -d nginx
docker-compose exec workspace bash
```

We'll generate the new certificates by calling `certmake`. This alias will call `letsencrypt certonly` and generates two `.pem` files. You nees to specify a domain, an email and the webroot folder. You can also add `-d` to perform a dry-run.

```
certmake your-domain.com user@example.com /var/www/letsencrypt
```

If everything went right, a success message should appear telling that a new set of certificates has been created. These will be located in `/etc/letsencrypt/live/your-domain.com/`. To make these files available to Apache and Nginx, `certmake` also creates a copy in `/var/certs/your-domain.com`.

Logout and modify your site's configuration. It should look similar to this:

```
listen 443 ssl;
listen [::]:443 ssl default_server ipv6only=on;
ssl_certificate /var/certs/your-domain.com/fullchain.pem;
ssl_certificate_key /var/certs/your-domain.com/privkey.pem;
```

Restart your `nginx` container.

```
docker-compose restart nginx
```

Now you should be able to connect to your sites through HTTPS. If you plan to redirect HTTP traffic you can check the following snippet:

```
    server {
      listen 80;
      server_name your-domain.com;
      return 301 https://your-domain.com$request_uri;
    }
```

## Sonic

From Sonic's website:

> Sonic is a fast, lightweight and schema-less search backend. It ingests search texts and identifier tuples that can then be queried against in a microsecond's time.

In order to start Sonic just run:

```bash
docker-compose up -d sonic
```

Sonic will start listening on port 1491. You can adjust this value by modifying `SONIC_PORT` in your `.env` file.


## NSQ

NSQ is a realtime distributed messaging platform that is composed by 3 different services: `nsqlookupd`, `nsq` and `nsqadmin`. To start NSQ just run the following:

```bash
docker-compose up -d nsq nsqadmin
```

You can adjust which ports to use by modifying the following values in your `.env`:

```conf
NSQD_TCP_PORT=4150
NSQD_HTTP_PORT=4151
NSQD_HTTPS_PORT=4152
NSQLOOKUPD_TCP_PORT=4160
NSQLOOKUPD_HTTP_PORT=4161
NSQADMIN_PORT=4171
```

### nsqadmin

NSQAdmin provides an admin tool for monitoring all traffic that goes through NSQ. Once you start this container, `nsqadmin` will start running on `http://localhost:4171`.

## Tools

While it's good to be able to run everything on Docker, you'll also need to provide tools to the host system to be able to diagnose any future issues you may experience. This list provides instructions for installing these tools on Debian systems.

### mysql-client

Get MySQL config package.

```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.13-1_all.deb
```

Install.

```bash
sudo dpkg -i mysql-apt-config*
```

A dialog will appear. This option only determines which packages will be available after updating. Select 'Ok'.

Reload packages.

```bash
sudo apt-get update
```

Install.

```bash
sudo apt-get install mysql-client
```

### mongoimport

Import the public key.

```bash
wget -qO - https://www.mongodb.org/static/pgp/server-4.2.asc | sudo apt-key add -
```

Create a /etc/apt/sources.list.d/mongodb-org-4.2.list file.

```bash
echo "deb http://repo.mongodb.org/apt/debian stretch/mongodb-org/4.2 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.2.list
```

Reload packages.

```bash
sudo apt-get update
```

Install.

```bash
sudo apt-get install -y mongodb-org-tools
```

## Troubleshooting

### Permissions on nginx SSL keys (development)

If you have any problems when initializing the `nginx` container try changing the permissions on the `default.key` file located inside the `nginx/ssl` folder.

```bash
sudo chmod 644 nginx/ssl/default.key
```

### Logs

After you started all the containers you can see the status of all of them by doing:

```
docker-compose ps
```

Check their status to see if they're all up and running. If you notice something wrong, you can check the logs of a particular container by using the `logs` command. For example, to listen to `nginx` container logs you can do the following:

```
docker-compose logs -f nginx
```

Keep in mind that all server logs will be only available on their corresponding container.

## License

[MIT License](https://github.com/laradock/laradock/blob/master/LICENSE)
