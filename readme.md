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

There is two more services in this docker-compose file: meilisearch and a caddy server with a mercure hub.

Change JWT Secret in docker compose : 
```bash
JWT_KEY: '!TheSecretToChange!'
MERCURE_PUBLISHER_JWT_KEY: '!TheSecretToChange!'
MERCURE_SUBSCRIBER_JWT_KEY: '!TheSecretToChange!'
```

In the .env.local file, add the following lines:
```bash
MERCURE_URL=http://mercure/.well-known/mercure
MERCURE_PUBLIC_URL=http://mercure/.well-known/mercure
MERCURE_JWT_SECRET=!TheSecretToChange!
MERCURE_CORS_ALLOWED_ORIGINS=*
```

Create a token here : https://jwt.io/ with the same secret as you defined earlier. You can change the secret at the bottom right of the form.
This token will be used to publish or subscribe.

The payload could be this (but should be more secured in production): 
```json
{
  "mercure": {
    "publish": ["*"],
    "subscribe": ["*"]
  }
}
```

Access the interface of the hub here : http://localhost:8082/.well-known/mercure/ui
You can test if your token works by setting it in the settings section. Then try to subscribe to a topic and publish to a topic.

Topics must be in the form of `http://example.com/{topic}`

(doc and good implementation is not finished but here's a POC)

In a Symfony controller, define theses 

```php
define('HUB_URL', 'http://mercure/.well-known/mercure');
define('JWT', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXJjdXJlIjp7InB1Ymxpc2giOlsiKiJdfX0.d-OMAinT8QctdPgZX_A74Cmw0NnEKLk-eGSN0vDPSJc');

use Symfony\Component\Mercure\Jwt\StaticTokenProvider;
use Symfony\Component\Mercure\Publisher;
use Symfony\Component\Mercure\Update;
```

Then in a method, you can publish to a topic like this : 

```php
$topic = 'http://example.com/topic_name';
$data = ['foo' => 'bar'];

$tokenProvider = new StaticTokenProvider(JWT);
$publisher = new Publisher(HUB_URL, $tokenProvider);

$update = new Update($topic, json_encode($data));
$publisher($update);
```

On your frontend subscribe to a topic like this : 

```js
var source = new EventSource("http://localhost:8082/.well-known/mercure?topic=http://example.com/topic_name");
source.onmessage = function(event) {
  console.log(event.data);
  
  document.getElementById("newPublishs").innerHTML += event.data + "<br>";
};

