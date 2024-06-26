# PHP development environment with namoshek/php-mssql:7.4-fpm, Nginx and MySQL to run Laravel applications using Docker and Docker Compose

You need to have Docker and Docker Compose installed on your server to proceed using this PHP environment.

This is a PHP development environment used to run Laravel applications. The following three separate service containers will be used:

- An `app` service running PHP7.4-FPM.
- A `db` service running MySQL 5.7.
- An `nginx` service that uses the `app` service to parse PHP code before serving the Laravel application to the final user.

## Running the application

- To get started, set up your application in the root directory.

- Set the environment variables creating a `.env` file. See the MySQL service section below for more informations.

- Build the app image with the following command:

```bash
docker-compose build app
```

- When the build is finished, you can run the environment in background mode with:

```bash
docker-compose up -d
```

- To show information about the state of your active services, run:

```bash
docker-compose ps
```

The environment is now up and running, but you still need to execute a couple commands to finish setting up the Laravel application. You can use the `docker-compose exec` command to execute commands in the service containers, such as an `ls -l` to show detailed information about files in the application directory:

```bash
docker-compose exec app ls -l
```

- Now run `composer install` to install the application dependencies:

```bash
docker-compose exec app composer install
```

- Generate a unique application key with the Artisan Laravel command-line tool. This key is used to encrypt user sessions and other sensitive data:

```bash
docker-compose exec app php artisan key:generate
```

- Now go to your browser and access your server’s domain name or IP address on port `8000`: `http://server_domain_or_IP:8000`. In case you are running this demo on your local machine, use `http://localhost:8000` to access the application from your browser.

- You can use the logs command to check the logs generated by your services:

```bash
docker-compose logs nginx
```

- If you want to pause your Docker Compose environment while keeping the state of all its services, run:

```bash
docker-compose pause
```

- You can then resume your services with:

```bash
docker-compose unpause
```

- To shut down your Docker Compose environment and remove all of its containers, networks, and volumes, run:

```bash
docker-compose down
```

## Services description

### Dockerfile

Although both `db` service and `nginx` service, will be based on default images obtained from the Docker Hub, the `app` service will be based on a custom image created by the `Dockerfile`.

The `Dockerfile` starts by defining the base image `php:7.4-fpm`.

After installing system packages and PHP extensions, the Composer will be installed by copying the composer executable from its latest official image.

A new system user is then created and set up using the `user` and `uid` arguments that were declared at the beginning of the `Dockerfile`. These values will be injected by Docker Compose at build time.

> This new system user is necessary to execute Laravel Artisan and Composer commands while developing the application. The `uid` setting ensures that the user inside the container has the same `uid` as your system user on your host machine. This way, any files created by these commands are replicated in the host with the correct permissions. This also means that you’ll be able to use your code editor of choice in the host machine to develop the application that is running inside containers.

Finally, the default working dir as `/var/www` and the newly created user are set. This will make sure you’re connecting as a regular user, and that you’re on the right directory, when running Laravel Artisan and Composer commands on the application container.

### PHP service

The `app` service will build an image called `laravel-image`, based on the `Dockerfile` previously created. The container defined by this service will run a namoshek/php-mssql:7.4-fpm server to parse PHP code and send the results back to the nginx service, which will be running on a separate container. The mysql service defines a container running a MySQL 5.7 server. All these services will share a bridge network named `app-network`.

The application files will be synchronized on both the `app` and the `nginx` services via bind mounts. Bind mounts are useful in development environments because they allow for a performant two-way sync between host machine and containers.

Inside the `app` container you will be able to execute command line tasks with the Laravel Artisan and Composer.

The `app` service will set up a container named `laravel-app`. It builds a new Docker image based on a `Dockerfile` located in the same path as the `docker-compose.yml` file. The new image will be saved locally under the name `laravel-image`.

The `volumes` setting creates a shared volume that will synchronize contents from the current directory to `/var/www` inside the container. Notice that this is not your document root, since that will live in the nginx container.

Another file which will be synchronized is the `local.ini` file from the directory `./php/local.ini` to `/usr/local/etc/php/conf.d/local.ini` inside the container.

The `local.ini` is the configuration file (php.ini) that is read when PHP starts up.

### Nginx service

The `nginx` service uses a pre-built Nginx image on top of Alpine, a lightweight Linux distribution. It creates a container named `laravel-nginx`, and it uses the ports definition to create a redirection from port `8000` on the host system to port `80` inside the container.

The `volumes` setting creates two shared volumes. The first one will synchronize contents from the current directory to `/var/www` inside the container. This way, when you make local changes to the application files, they will be quickly reflected in the application being served by Nginx inside the container. The second volume will make sure the Nginx configuration file, located at `./nginx/conf.d/app.conf`, is copied to the container’s Nginx configuration folder. This configuration file will configure Nginx to listen on port `80` and use `index.php` as default index page. It will set the document root to `/var/www/public`, and then configure Nginx to use the `app` service on port `9000` to process all the php files.

### MySQL service

The `db` service uses a pre-built MySQL 5.7 image from Docker Hub. Because Docker Compose automatically loads `.env` variable files located in the same directory as the `docker-compose.yml` file, you can obtain the database settings from the Laravel `.env` file.

The `volumes` setting creates two shared volumes. The first one will make sure the MySQL configuration file, located at `./mysql/my.cnf`, is copied to the container’s MySQL configuration folder. The second volume will share a `.sql` database dump that will be used to initialize the application database. The MySQL image will automatically import `.sql` files placed in the `/docker-entrypoint-initdb.d` directory inside the container.

The `environment` setting defines environment variables in the new container. You can use values obtained from the Laravel `.env` file to set up the MySQL service, which will automatically create a new database and user based on the provided environment variables:

```bash
DB_HOST=db
DB_DATABASE=laravelapp
DB_USERNAME=laravelapp_user
DB_PASSWORD=password
```

## References

- https://www.digitalocean.com/community/tutorials/how-to-install-and-set-up-laravel-with-docker-compose-on-ubuntu-20-04
- https://docs.docker.com/
- https://docs.docker.com/compose/
