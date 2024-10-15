# Boilerplate example for Rails-React-Docker appication
A boilerplate combining Ruby on Rails, React, and Docker for quick setup and deployment of full-stack application projects.

Based on and many thanks to [the Wild Code for Coffee](https://github.com/willcodeforcoffee/docker-rails-react/blob/master/README.md).

------



This guide provides a step-by-step approach to setting up a <ins>**Docker**</ins>-based development environment for running a <ins>**React**</ins> frontend alongside a <ins>**Ruby on Rails**</ins> backend. Designed specifically for ***development purposes***, this setup simplifies dependency management and eliminates the need to install programming languages directly on a local machine, as everything is handled within Docker containers.

The guide outlines how to configure a ***Docker Compose*** file, enabling the generation and execution of React and Rails projects entirely within Docker. This approach removes the necessity for local installations of Ruby or node.js, streamlining the development process and keeping the system clean. While optimized for development, this setup is not intended for production use; further information on production-ready guidelines can be found in the FAQ.


## Table of Contents
[Requirements](#requirements)

[Step 1: Set up Docker Compose](#step-1-create-docker-composeyml)
    
[Step 2: Build the Frontend](#step-2-build-the-react-frontend)

[Step 3: Build the Backend](#step-3-build-the-rails-backend)
        
- [Postgres and Redis Setup](#step-3a-add-postgres-and-redis-services)
- [Rais Setup](#step-3b-add-the-rails-backend-service)
- [Rails Integration with Other Services](#step-3c-configure-rails-to-use-other-services)
- [Fixing Potential Problems](#fixing-the-serverpid-problem)

[Step 4: Reverse Proxy Setup](#step-4-reverse-proxy-setup)

[CLI commands](#how-can-rails-cli-commands-be-run)

# Requirements

To proceed with the instructions, you need to have the following software installed:

- [Docker Desktop](https://www.docker.com/products/docker-desktop/), there won’t be provided installation guidance here; please refer to the [the Docker website](https://www.docker.com/get-started/) for instructions specific to your operating system.

Subtle differences exist in how Docker operates on Windows, macOS, and Linux. Extra steps may be necessary depending on the specific operating system version, which will be noted as required. Docker facilitates the creation of standardized developer environments independent of the host operating system. 
# Getting Started

Creating and running Docker containers from the command line is straightforward, but this guide focuses on running an entire environment. To manage multiple containers for a single application, the Docker feature known as [Docker Compose](https://docs.docker.com/compose/) will be utilized. Docker Compose simplifies the management of multiple containers, folder mappings, and networking between containers.

Docker Compose operates using a [YAML](https://yaml.org/) named docker-compose.yml. Let’s start by setting that up.

## Step 1: Set up Docker Compose

### Create a docker-compose.yml file

In the working directory, create a file named `docker-compose.yml` and populate it with the following content:

```yml
version: "3.8"
services:
  frontend:
    image: node:18.8.0-bullseye
    restart: "no"
    volumes:
      - ./frontend:/app
```

In this file, we have defined a container named `frontend` that will run Node v18 on Debian Bullseye. This container is configured not to restart automatically. It establishes a volume mapping from the `/app` folder within the container to the `./frontend` directory in the current working folder on the host machine.

To generate the frontend React application, the `create-react-app` command can be executed **inside the container**. The volume mapping will ensure that the generated files are saved directly to the host machine:

```sh
docker compose run frontend yarn create react-app app --template typescript
```

**A few things are going to happen now!**

1. Docker will download the Node 18 image layers from Docker Hub to your host computer.
2. Docker Compose will start a container named "frontend" using the image it just downloaded with the folder mapping `/frontend` to `/app`
3. Docker will *run* on "frontend" the `yarn create react-app` command into the `/app` folder on the container using the Typescript template.
4. Because Docker mapped `/app` to `./frontend`, Docker will create the `frontend` folder and when `create-react-app` finishes the React application will also be in that folder on the host machine.
5. When `create-react-app` completes Docker will stop the "frontend" container.

All of these steps took a little over 10 minutes on my computer so don't be worried if it takes a while. This did not install Node on your host machine, it ran it from inside a Docker container.

## Step 2: Build the React Frontend
### Create a Docker Image for Frontend

At this stage, executing the following command from the working directory (or from within the `./frontend` directory) is necessary, as the folder may have been created with `root` permissions and requires changing ownership:

```sh
sudo chown -R $USER .
```

In the newly created `frontend` folder, it is necessary to create a `Dockerfile` that enables the repeated creation of the React frontend and facilitates the app's execution upon starting.

Create a file named `Dockerfile` in the `frontend` folder. Include the following contents:


```Dockerfile
FROM node:18.8.0-bullseye

RUN mkdir /app
WORKDIR /app

EXPOSE 3000
```

The `Dockerfile` will utilize the same `image` specified in the `docker-compose` file (node:18.8.0-bullseye) as a base, setting `/app` as the directory for executing commands. Additionally, it will `EXPOSE` port 3000, which is the default for the `create-react-app` server.

Next, update the  `docker-compose.yml` for creating the app image instead of the base Node:18-bullseye image:


```yml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    command: yarn install && yarn start
    restart: "no"
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"

```


By substituting the `image` section with a `build` section, "frontend" has been switched from using an existing `image` to a built Dockerfile.

Additionally, a `command` has been added to execute `yarn install`.

The React server can now be started using the following command:

```sh
docker compose up
```

Before proceeding to the next step, ensure that the current Docker services are stopped (`CTRL+X` and run `docker-compose down`).

Next, edit the `docker-compose.yml` file to set it up to start the React development server by using the command `yarn start`, which will initiate the React development server:

```yml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    command: yarn start
    restart: "no"
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"

```
After the React server is running you will be able to use your web browser to go to http://localhost:3000/ and you should see the Create React App default page.

At this point we have demonstrated how to use Docker to create a Node/React application without having to install any programming languages. How exciting! 🥳 You can fire up your code editor to edit directly from the host filesystem or open in the container using VS Code devcontainers or WebStorm.

Once the React server is operational, access it via a web browser by navigating to [http://localhost:3000/](http://localhost:3000/), where the default Create React App page should be displayed.

This process illustrates how to utilize Docker to set up a Node/React application without requiring any installation of programming languages. This is an exciting development! 🥳 Code can be edited directly from the host filesystem or within the container.


Before proceeding to the next step, ensure that the current Docker services are stopped: `CTRL+X` and then run `docker-compose down`

## Step 3: Build the Rails Backend

The next step involves enhancing the existing frontend by integrating a backend service.

To increase the complexity slightly, a Rails application will be created to utilize both Postgres and Redis. This approach simulates a more realistic production environment while demonstrating the simplicity of adding services through Docker Compose.

The initial focus will be on setting up the Postgres and Redis services. As database services, they require persistent storage to retain data, even after restarts or system reboots. Therefore, persistent volumes will be necessary for these services.


### Step 3a: Add Postgres and Redis services

Let's add the following snippet to our `docker-compose.yml` file:

```yml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    command: yarn start
    restart: "no"
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"
  postgres:
    image: postgres:12.11-alpine
    environment:
      POSTGRES_PASSWORD: pg
    volumes:
      - postgres-data:/var/lib/postgresql/data:rw
  redis:
    image: redis:7.0-alpine
    volumes:
      - redis-data:/data:rw
volumes:
  postgres-data:
  redis-data:

```

The `volume` mappings for the databases have been included. Docker uses the `host:container` notation for this purpose. In this scenario, two *Docker managed volumes* named `postgres-data` and `redis-data` are created in the `volumes` section. For the frontend, the source code was mapped to a specific folder, but for the database services, the focus on data location is less critical, allowing Docker to manage the volumes independently.

Additionally, the `environment` section for the `postgres` service is noteworthy. In alignment with the [12 Factor App](https://12factor.net/config) principles, both Docker and Postgres facilitate the configuration of settings via environment variables. The [Postgres Docker image mandates the creation of a database with an associated password](https://hub.docker.com/_/postgres/), which is achieved through the `POSTGRES_PASSWORD` environment variable.

> Currently, `POSTGRES_PASSWORD` is not a secure option, even for development environments. 
> Instructions on how to configure it more securely will be provided  later (in step 3c) when setting up the contents of the `.env` file.\
\* For the moment, nothing is pushed to the remote branch. Before any potential push to the remote branch at this stage, ensure that `docker-compose.yml` is added <ins>only temporarily</ins> to the `.gitignore` file (`echo 'docker.compose.yml' >> .gitignore`).
> 


To verify the configuration, start Docker once more:

```sh
docker compose up
```
This process might take some time as Docker downloads the Postgres and Redis images from Docker Hub. You'll know it's ready when you see the following in the console output:


```
docker-rails-react-postgres-1  | 2022-01-01 00:00:00.000 UTC [1] LOG:  database system is ready to accept connections
```

and

```
docker-rails-react-redis-1     | 1:M 01 Jan 2022 00:00:00.000 * Ready to accept connections
```

If both services display "ready to accept connections," we can proceed with setting up Rails. Use `CTRL+C` to stop Docker, then run `docker compose down` to stop the services again

### Step 3b: Add the Rails Backend service

The initial step is to set up a container for our backend Rails service. We'll replicate the process used for the `frontend` service by utilizing a `Dockerfile` and specifying a build context.

Execute the following commands to create an `api` folder and a `Dockerfile`:

```sh
mkdir api
touch api/Dockerfile
```

Next, let's modify the `api/Dockerfile` to utilize the latest Ruby 3.1 image:

```Dockerfile
FROM ruby:3.1-bullseye

RUN gem install bundler rails rake

RUN mkdir /app
ENV RAILS_ROOT /app
WORKDIR /app

EXPOSE 3000
```

Now, include a new service in your `docker-compose.yml` file:

```yml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    command: yarn start
    restart: "no"
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"
  postgres:
    image: postgres:12.11-alpine
    environment:
      POSTGRES_PASSWORD: pg
    volumes:
      - postgres-data:/var/lib/postgresql/data:rw
  redis:
    image: redis:7.0-alpine
    volumes:
      - redis-data:/data:rw
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    command: sh -c "/wait && bin/start_rails.sh"
    restart: "no"
    volumes:
      - ./api:/app
volumes:
  postgres-data:
  redis-data:
```

We've established a volume mapping from `api` to `app` in the container. Now, let's utilize this container to create a new Rails app:

```sh
docker compose run api rails new . --database=postgresql --api --skip-git --force

```

At this point, it's necessary to run the following command again from the working directory (or from within the `./api` directory) to change `root` ownership and permissions to `user`:

```sh
sudo chown -R $USER .
```

This will initiate the following steps:
1. Docker will download the latest Ruby 3.1 image (consider the latest one but in a consistent way).
1. Docker Compose will start the services.
1. Once all services are running, it will create a new Rails app within the container.

Due to the volume mapping, your new Rails app will be generated in the `api` folder. Take a look.

### Step 3c: Configure Rails to Use Other Services

### Environment File 

Rails requires some setup to integrate with the services we've created. The configurations can easily be added using [environment variables](https://docs.docker.com/compose/environment-variables/). We already defined one environment variable, `POSTGRES_PASSWORD`, above. To simplify configuration maintenance and enable sharing among containers, let's move our settings into a [separate file named `.env`](https://docs.docker.com/compose/environment-variables/#the-env-file).

So, create a `.env` file at the same level as your `docker-compose.yml` file and add the following to it:


```env
## Server Configuration
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=docker_rails_react
DATABASE_URL=postgres://postgres:postgres@postgres:5432/docker_rails_react

REDIS_HOST=redis
REDIS_PORT=6379
REDIS_CHANNEL_PREFIX=docker_rails_react
REDIS_URL=redis://redis:6379/0
```


It is now paramount as an essential security measure to prevent the exposure of sensitive personal information to add both the `.env` file and the Rails Μaster Κey (`api/config/master.key`) to the `.gitignore` file:
```bash
echo '.env' >> .gitignore
echo 'api/config/master.key' >> .gitignore
```


###  - Edit database.yml

Let's configure Rails to use the database. Edit `api/config/database.yml` to include the following contents:

```yml
default: &default
  adapter: postgresql
  encoding: unicode
  # For details on connection pooling, see Rails configuration guide
  # https://guides.rubyonrails.org/configuring.html#database-pooling
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: <%= ENV['POSTGRES_USER'] %>
  password: <%= ENV['POSTGRES_PASSWORD'] %>
  host: <%= ENV['POSTGRES_HOST'] %>
  port: <%= ENV['POSTGRES_PORT'] %>

development:
  <<: *default
  database: <%= ENV['POSTGRES_DB'] %>

test:
  <<: *default
  database: <%= ENV['POSTGRES_DB'] %>_test

production:
  <<: *default
  database: <%= ENV['POSTGRES_DB'] %>
```

###  - Edit cable.yml

We can configure ActionCable to utilize our Docker Redis instance in development as well. Edit `api/config/cable.yml` to include the following contents:

```yml
development:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://redis:6379/0" } %>
  channel_prefix: <%= ENV.fetch("REDIS_CHANNEL_PREFIX") { "" } %>

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://redis:6379/0" } %>
  channel_prefix: <%= ENV.fetch("REDIS_CHANNEL_PREFIX") { "" } %>
```

### Edit Docker Compose

Now to use it we can add `env_file` to all of the services and remove the `environment` section in our `docker-compose.yml` file:

```yml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    command: yarn start
    restart: "no"
    env_file:
      - ".env"
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    command: sh -c "bundle install && bin/rails server"
    restart: "no"
    env_file:
      - ".env"
    volumes:
      - ./api:/app
  postgres:
    image: postgres:12.11-alpine
    env_file:
      - ".env"
    volumes:
      - postgres-data:/var/lib/postgresql/data:rw
  redis:
    image: redis:7.0-alpine
    env_file:
      - ".env"
    volumes:
      - redis-data:/data:rw
volumes:
  postgres-data:
  redis-data:
```

### Fixing the server.pid Problem

When running `docker compose up` a second time, the following output may be received:

```
docker-rails-react-api-1       | => Booting Puma
docker-rails-react-api-1       | => Rails 7.0.4 application starting in development
docker-rails-react-api-1       | => Run `bin/rails server --help` for more startup options
docker-rails-react-api-1       | Exiting
docker-rails-react-api-1       | A server is already running. Check /app/tmp/pids/server.pid.
```

An unintended consequence of running the Rails server through `sh -c`, as done in the `docker-compose.yml` file, is that `sh` will catch but not forward interrupt signals to Rails. Without that signal, the [Puma](https://puma.io/) web server cannot shut down cleanly and remove the `server.pid` file.

The [explanation and solution found on Stack Overflow](https://stackoverflow.com/a/38732187) suggests creating a Docker [entrypoint](https://docs.docker.com/engine/reference/builder/#entrypoint).

Using the `entrypoint` method involves three steps:
1. Create an entrypoint shell file (`api/bin/docker-entrypoint.sh`).
1. Add an `ENTRYPOINT` instruction in `api/Dockerfile` that points to the shell file.
1. Modify the command in `docker-compose.yml` to run only `rails server`.


#### Step 1: api/bin/docker-entrypoint.sh

This file checks for and removes `server.pid`, then executes other environment setup commands like `bundle install`. The final line will `exec` the `command` specified in `docker-compose.yml`.

It is placed in the `bin` subfolder as a convention, but there is no requirement for it to be located there.


```sh
#!/bin/sh
set -e

if [ -f tmp/pids/server.pid ]; then
  rm tmp/pids/server.pid
fi
bundle install
exec bundle exec "$@"
```

#### Step 2: api/Dockerfile

The primary changes to the Dockerfile occur at the bottom, where the `ENTRYPOINT` is defined.

```Dockerfile
FROM ruby:3.1.3-bullseye

RUN gem install bundler rails rake

RUN mkdir /app
ENV RAILS_ROOT /app
WORKDIR /app

EXPOSE 3000

ENTRYPOINT ["/app/bin/docker-entrypoint.sh"]
CMD ["rails", "server"]
```

While it's uncertain if the `CMD` instruction is necessary, it's beneficial to include it as a default option regardless.

#### Step 3. docker-compose.yml

For the sake of simplicity, the rest of the file is omitted so that the change to be made is clear:

```yml
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    command: ["rails", "server"]
```

The entire `docker-compose.yml` file should now look like this:

```yml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    command: yarn start
    restart: "no"
    env_file:
      - ".env"
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    command: sh -c "bundle install && bin/rails server"
    restart: "no"
    env_file:
      - ".env"
    volumes:
      - ./api:/app
  postgres:
    image: postgres:12.11-alpine
    env_file:
      - ".env"
    volumes:
      - postgres-data:/var/lib/postgresql/data:rw
  redis:
    image: redis:7.0-alpine
    env_file:
      - ".env"
    volumes:
      - redis-data:/data:rw
volumes:
  postgres-data:
  redis-data:
```

## Step 4: Reverse Proxy Setup

The reverse proxy will sit in front of the React and Rails containers, making them accessible through a single URL (host and port) instead of two.

We will use [nginx](https://nginx.org/en/) as our reverse proxy because it is easy to configure and provision (it's also a commonly used web server and proxy in production environments).

We will proxy the React `frontend` service to `/`, the root of our site, and the Rails `api` to `/api` of the site. Additionally, we need to consider that `create-react-app` provides hot reloading via a websocket at `/ws`.

The port mapping for `frontend` and `api` will be removed so that they can only be accessed through `proxy` afterward.

We will use the [`nginx`](https://hub.docker.com/_/nginx/) Docker image directly without needing another `Dockerfile`.

**nginx** is configured using a file called `nginx.conf`. The image allows for easy overriding of the default configuration by simply mapping a file volume over the existing file in the image. The following steps will demonstrate how to do that:

### Add `nginx.conf`

A `proxy` folder will be created at the root level, alongside `api` and `frontend`. The `nginx.conf` file will be placed there.

Create `proxy/nginx.conf` and add the following content to the file:


```nginx
worker_processes auto;
worker_rlimit_nofile 2048;

events {
  worker_connections 1024;
}

error_log /dev/stdout info;

http {
  charset                utf-8;

  sendfile               on;
  tcp_nopush             on;
  tcp_nodelay            on;

  server_tokens          off;

  types_hash_max_size    2048;
  types_hash_bucket_size 64;

  client_max_body_size   64M;

  include /etc/nginx/mime.types;
  default_type           application/octet-stream;

  server {
      listen 80;

      location / {
        proxy_pass http://frontend:3000/;
        # To proxy websockets
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
      }

      # Websocket for automatic reloading
      location /ws {
        proxy_pass http://frontend:3000/ws;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
      }


      location /api/ {
        proxy_pass http://api:3000/;
        proxy_set_header Upgrade \$http_upgrade;
      }
  }
}
```

Much of the file consists of boilerplate copied from another file, but the critical changes are found in the `server` and `location` directives. Each `location` has a `proxy_pass` directing traffic to each service specified in our `docker-config.yml` file. Additionally, headers are included for connection upgrades to manage websocket delegation behind the scenes.

### docker-compose.yml changes

A new service named `proxy` has been created:
1. Utilizing the `nginx` image
2. Mapping `nginx.conf` using the `volumes` command
3. Setting the listening port to `3000`
4. Establishing dependencies on the downstream `frontend` and `api` containers to prevent premature startup

```yml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    command: yarn start
    restart: "no"
    env_file:
      - ".env"
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    command: sh -c "bundle install && bin/rails server"
    restart: "no"
    env_file:
      - ".env"
    volumes:
      - ./api:/app
  postgres:
    image: postgres:12.11-alpine
    env_file:
      - ".env"
    volumes:
      - postgres-data:/var/lib/postgresql/data:rw
  redis:
    image: redis:7.0-alpine
    env_file:
      - ".env"
    volumes:
      - redis-data:/data:rw
  proxy:
    image: nginx:1.23.3-alpine
    volumes:
      - ./proxy/nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "3000:80"
    depends_on:
      - frontend
      - api
volumes:
  postgres-data:
  redis-data:
```

---

## How can Rails CLI commands be run?

```sh
docker compose exec api rails help
```

Simply replace `help` with the desired command!

---
