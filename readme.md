# Symfony Doxec

Symfony Doxec is a tool to create and develop a Symfony project with docker.

It comes with a doxec script that will help you to create a new project, start it, stop it etc.

Why *Doxec* ? Because it's a mix of *Docker* and *Exec*. The main goal being to run commands in a docker container.

## Requirements

- Docker
- Git

That's all :tada:

*Tested only on linux for instance. Won't work on Windows and may not work with Windows WSL or Mac OS.*

## Installation

Clone this repository and copy files in an empty directory of your choice.

We are going to use the `doxec` script to help us create everyting. But it's not mandatory. You can do everything manually and get rid of the script. Manual instructions are available below on the **Manual** section.

--- 
### Alias for the doxec script
Add an alias for the doxec script in your .bashrc or .zshrc file: It's not mandatory but it will be easier to use.

```bash
alias doxec="./doxec"
```

### Starting containers

If you have not defined an alias, you need to run the script with `./doxec`.

If doxec is not executable, you need to make it executable with `(sudo chmod +x doxec`.

From the folder where you copied files, run `doxec up` to start docker containers. All services are defined in the docker-compose.yml file.

Services are:
- `db` : MySQL database
- `web` : The container running the Symfony application
- `phpmyadmin` : PhpMyAdmin to manage the database
- `mailhog` : MailHog to catch emails sent by the application

This is the strict minimum for a nice development environment. You can add more services if you want, like *meilisearch*, *redis*, *elasticsearch* etc.

At this stage, nothing is installed in the `web` container. So the next step is to create a new Symfony project.

### Creating a new Symfony project

Run `doxec sf install`.

It will install a Symfony `webapp` in the `web` container. It will also : 
- install the symfony cli in case you need it 
- install the doctrine fixtures bundle.
- create a `.env.local` file with the database credentials so you don't have to take care of it.
- init a git repository

You are now ready to start developing your Symfony application.

Access:
- application: http://localhost:8080
- phpmyadmin: http://localhost:8081
- mailhog: http://localhost:8025

The command `doxec sf install` will create a new Symfony project only if there is not `symfony.lock` file. If there is one, it will install dependencies, migrate database and load fixtures.

**To sum up, all you need to do is running `doxec up && doxec sf install` to start a new project.**

## Usage

Basically, `doxec` is a shortcut to run commands in the `web` container. It's a wrapper around `docker compose exec -it web`.

`doxec <command>`
Run any command in the `web` container.

Except for all command listed below :

`doxec up` 
Start all containers.

`doxec stop`
Stop all containers.

`doxec down`
Stop and remove all containers.

`doxec clean` 
Stop and remove all containers and volumes.

`doxec sf`
Shortcut to run `php bin/console` in the `web` container.

`doxec sf install`
Install a new Symfony project or load an existing one in the `web` container.

## Manual

### Start containers
  
```bash
docker-compose up -d
```

### Install project in the web container

```bash
docker-compose exec -it web bash

# Inside the container create the project in a tmp folder
composer create-project symfony/website-skeleton tmp --no-interaction

# It create docker files, we don't want them
rm -rf tmp/docker-compose.yml tmp/docker-compose.override.yml

# Move everyting back to /var/www
mv -f tmp/{.,}* .

# Install fixture bundle
composer require --dev doctrine/doctrine-fixtures-bundle
```

Think at creating a .env.local file with the database credentials. 
  
  ```bash
  DATABASE_URL=mysql://user:password@db/database?serverVersion=5.7&charset=utf8mb4
  ```

### Usage

You need to run every command in the web container. So go inside it with:
``` bash
docker-compose exec -it web bash
```

then use it like you would normally

```bash
php bin/console make:entity
```

## More services

If you want to add more services, you can add them in the docker-compose.yml file. For example for meilisearch :
```yaml
  meilisearch:
    image: getmeili/meilisearch:latest
    container_name: meilisearch
    restart: always
    ports:
      - 7700:7700
    volumes:
      - meilisearch:/data.ms
  [...]
  volumes:
    [...]
    meilisearch:
```

