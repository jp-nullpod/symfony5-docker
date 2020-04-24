# Purpose

We will be trying to host a simple Symfony 5 app with a MySql Database on docker images and run them.

# Creating our Symfony 5 app

1. Create the symfony 5 basic web application project, go to your terminal and type: (I am assuming here that symfony is already installed on your machine)

```
symfony new symfony5-docker --full
```

go to the project folder that you have just created,
```
cd symfony5-docker
```
then start the project to test it:
```
symfony server:start --no-tls
```

2. Open your browser, and go to the URL provided which is typically:
```
http://127.0.0.1:8000/
```

3. Create your environment variables in the *.env* file based on your environment parameters. Fill them with the relevant values.

```
MYSQL_ROOT_PASSWORD=...
MYSQL_DATABASE=...
MYSQL_USER=...
MYSQL_PASSWORD=...
```

4. Create a folder ```docker``` in the project, under docker (see picture below):
- Create a folder ```nginx``` and a file ```default.conf``` inside it
- Create a folder and ```php``` and a file ```Dockerfile``` inside it

![Files Hierarchy](./resources/docker_folder.png)

# Creating our Docker Images

To create a simple app project, assuming there is a MySql database in it and that the app is hosted on a web server e.g. nginx. We will need 3 images: mysql, nginx and a php runtime

## Configuration (docker-compose.yml)

The Docker Images needed are:
- mysql, Container name: mysql
    - I used MySql 8 for a stable MySql docker image. Note that MySql 8 uses a different password strategy for hashing and storing passwords. 
    To use the default method of MySql 5.7, ```--default-authentication-plugin=mysql_native_password``` command has been added.
    
- php, Container name: phpbin
    - This image contains the php runtime and is governed by its own Dockerfile stored in the project at ```docker/php/Dockerfile``` as explained 
    below. The php version and image used is specified in the Dockerfile.
     
- nginx, Container name: nginx
    - The web server used for the development environment. Its config is stored in the project itself at ```./docker/nginx/default.conf```.

Network:
- All images are by default on the same network, but their IP address cannot be predicted in advance. To make sure, that they are on the 
same network. The ```networks:``` section has been added per image with a common name. 
(```symfony``` in this case). You can use any name you want as long as they are the same on each image. When the docker project is up and running
```docker
    networks:
      - symfony
```

## The *docker-compose.yaml* file

Open [here](docker-compose.yaml) for the full file.

1. Create the docker-compose.yaml in the root folder of the app. It can be outside as well if you are working with several apps and technologies, but it would mean that you would have to change all the relative paths below.

In our mysql configuration below, the ```volumes:``` section means that the folder ```.docker/data/db``` is mapped to ```/var/lib/mysql``` on the docker image. 
The same applies to ```ports:```. Our mysql database is accessible on the host machine on ```127.0.0.1:3307``` once the docker images are up and running.
The ```${  }``` syntax suggests that the value of those variables have been defined in the .env and will be replaced here. ```${MYSQL_ROOT_PASSWORD}``` will be replaced by the value defined in ```.env```.

```docker
  mysql:
    image: mysql:8
    container_name: mysql
    volumes:
      - '.docker/data/db:/var/lib/mysql'
    ports:
      - 127.0.0.1:3307:3306
    restart: on-failure
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    networks:
      - symfony
```

For the ```nginx``` image, a few important notes:
- our nginx server depends on our php image.
- our port has been defined in ```.env```, I use port 8000 mapped to 80.
- our ```default.conf``` is found in our project and contains basic configurations ```nginx``` configurations for a php symfony project to work properly 

```docker
    volumes:
      - './public/:/usr/src/app'
      - './docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro'
    ports:
      - '${NGINX_PORT}:80'
    depends_on:
      - php
    networks:
      - symfony
```

The php runtime most important section is the path of the Dockerfile which is explained below in the tutorial.

```docker
    build:
      context: .
      dockerfile: docker/php/Dockerfile
```
3. Modifying our Environment Variables

In our .env file, we usually use the name of the host or IP address

```
DATABASE_URL=mysql://test_user:test_password@localhost:3306/database_name
```

Since we are now running all of our apps inside our Docker images, what is pretty cool here is that we can use the name of the Docker image instead of the IP address in our .env file!

```
DATABASE_URL=mysql://test_user:test_password@mysql:3306/database_name
```

## The Dockerfile

Open [here](./docker/php/Dockerfile) for the full file.

Our Dockerfile determines how the image will look like and which packages need to be installed for it to work properly.

We use Php 7.4 on our server, but any version of PHP7 would work in that case

```dockerfile
FROM php:7.4-fpm
```

This section installs network tools for ping, database drivers, zip/unzip for the images, etc..

```dockerfile
RUN docker-php-ext-install pdo_mysql

RUN pecl install apcu

RUN apt-get update && \
apt-get install -y \
zlib1g-dev \
iputils-ping \
unzip \
zip

RUN docker-php-ext-install zip
RUN docker-php-ext-enable apcu
```

For the *composer* install, it is important to update the hash from the *composer* website for the composer component to be installed properly
on the docker images

```dockerfile
php -r "if (hash_file('sha384', 'composer-setup.php') === 'e0012edf3e80b6978849f5eff0d4b4e4c79ff1609dd1e613307e16318854d24ae64f26d17af3ef0bf7cfb710ca74755a') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;" \
```

The var folder for logs, cache and sessions need to have full user rights so that your app can write to them. 
Here we give full permissions to this directory for the dev environment.

```dockerfile
RUN chmod -R 777 /usr/src/app/var/
```


## Setup and Commands

To setup the project, you will need the latest versions of:
- docker
- docker-compose

To build the project:
```bash
docker-compose build
```
Hmmm... this seemed to have worked:

![command line output](./resources/docker_creation.png)

To run the project in 'detached' mode:
```bash
docker-compose up -d
```

### php image setup and config

To access the *phpbin* docker image:
```bash
docker exec -it phpbin /bin/bash
```
Cool, we're in:

![command line output](./resources/docker_web.png)

Check that php has been installed properly on the image by running:
```
php -v
```
Oooh! Nice:

![command line output](./resources/docker_phpv.png)

You can even ping the other images from the php image:

```bash
ping mysql
ping nginx
```

![command line output](./resources/we_can_ping.png)

Now, run the project, type:
```bash
composer install
```

- Congrats, your app should be available at http://localhost:8000 (or the port number that you have specified in your .env file)

```
http://localhost:8000/
```

or should I say 'Yay! It's awesome!':

![website output](./resources/page_running.png)


### mysql image setup and config
- To access the *mysql* docker image:
```bash
docker exec -it mysql /bin/bash
```
- Access the mysql console by typing:
```bash
mysql -u root -p 
```
- Type the passwword provided in the *docker-compose.yml* file
- Create a user to access the database from the network
```sql
CREATE USER 'test_user'@'%'IDENTIFIED WITH mysql_native_password BY 'test_password';
GRANT ALL PRIVILEGES ON *.* TO 'test_password'@'%';
```
- Note the use '%', so that the user can access the database from a different machine on the network


### Misc. Notes.

To exit the image, type:
```
exit
```

And to finally stop the docker, type:
```
docker-compose stop
```






