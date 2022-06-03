# Docker & Phoenix

This is the fastest (but not necessarily the best) way to setup elixir, phoenix, docker

## Create Phoenix Project

This part is a little ugly because you have to install several things you won't really need now, but will regret not having there in later steps. On the upside, you only have to do this one time.

1. Run a docker container for elixir
  - `docker run -it --rm -v $(pwd):/usr/src/app elixir:1.10.3 bash`
     [KH] Launch docker, run the bash command interactively, with a pseudo-terminal so that you can respond to prompts, and mount current directory in the docker container as /usr/src/app. Also remove the container process when you exist bash. Oh, also... do this all from a starting point of the prebuilt elixir version 1.10.3 docker image.

2. Within the docker cli
2.1 `cd /usr/src/app`
     [KH] cd to the local directory you mounted
2.2 install hex and phoenix
  - `mix do local.hex --force, archive.install hex phx_new 1.5.3`
    [KH] we're installing all the needed tools within the docker container so that we have elixir handy (?)
2.3 install nodejs and inotify-tools
  - `curl -sL https://deb.nodesource.com/setup_10.x | bash -`
  - `apt-get update -yqq && apt-get install -yqq --no-install-recommends nodejs`
2.4 `mix phx.new hello`
    [KH] Create a brand new phoenix app called hello!
2.5 Exit the docker cli
    

## Setup the Project

Outside of docker...

1. `cd hello`
2. create Dockerfile
```
FROM elixir:1.10.3
  [KH] start with this prebuilt image
LABEL maintainer="david.madouros@gmail.com"
  [KH] set metadata on the image, indicating the author
  
RUN curl -sL https://deb.nodesource.com/setup_10.x | bash -
  [KH] download and run the node.js repo setup script
RUN apt-get update -yqq && apt-get install -yqq --no-install-recommends \
  nodejs \
  inotify-tools
  [KH] download and install nodejs and inotify-tools; don't consider recommended packages as dependencies & answer 'y' to all prompts & be quiet about it
  [KH] I thought there was an extra "q" there but it means "level 2 quiet", or VERY quiet. (Can also be written as "-q=2"

WORKDIR /usr/src/app
  [KH] Starting from this directory...
RUN mix do local.hex --force, local.rebar --force, archive.install hex phx_new
  tell mix to execute this list of commands: 
     [KH] local.hex with force - install hex locally without prompts
     [KH] local.rebar with force - download rebar without prompts
     [KH] archive.install - install the phoenix project generator locally using hex 

COPY . /usr/src/app/
    [KH] copy the current directory to /usr/src/app
    [KH] why? Isn't it already there? does it copy it to the path outside the container?

CMD  ["mix", "phx.server"]
     [KH] start up the phoenix server
```

3. create docker-compose.yml
```
version: "3.8"
  [KH] use version 3.8 of docker


  [KH] create two services: web and database
services:
  web:
    build: .
    ports:
      - "4000:4000"
    [KH] export port 4000 through container

    volumes:
      - .:/usr/src/app
    [KH] expose the current directory to the docker container as /usr/src/app

    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: hello_dev
      DATABASE_HOST: database
   [KH] set these env vars

  database:
    image: postgres
    [KH] start with the prebuilt postgres image

    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: hello_dev
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:

  [KH] declare db_data as a volume variable?
```

4. update `config/dev.exs`
Change `hostname: "localhost",` to `hostname: "database",`
  [KH] tell the database to find the database service as the host

## Create Database

1. `docker-compose up -d database`
2. `docker-compose run web mix ecto.create`
3. `docker-compose up -d web`

## Visit page

1. http://localhost:4000/
