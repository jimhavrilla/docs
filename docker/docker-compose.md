## Creating the `.yml` file for `docker-compose`

So to add a layer of complexity, let's talk about `docker-compose`.  `docker-compose` allows you to use more than one docker container at once for a joint task.  The case I'll discuss here is setting up a search engine using `elasticsearch` (I won't go over how to set up indices, that's very case-by-case).

The `Dockerfile` set up is identical to how it was done in the [first tutorial](https://github.com/jimhavrilla/docs/edit/master/docker/generaluse.md), so you can pretty much reuse it.  Don't forget to clone the [simpleflaskdocker](https://github.com/jimhavrilla/simpleflaskdocker) repo: `git clone https://github.com/jimhavrilla/simpleflaskdocker.git` or `git clone git@github.com:jimhavrilla/simpleflaskdocker.git` depending on your setup.

Before running anything make sure to run (and have `Python 3.7+` or `miniconda3` installed):

```
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.8.1
pip install elasticsearch
```

It took `~33 seconds` on my Macbook Pro 2015 on Wi-Fi.  Hopefully it's as fast for you!


For example, here's what I set up for a web server I did recently with `elasticsearch` in a `docker-compose.yml` file with comments. 

```
version: '3.5' # docker-compose version (actually important here)

services:
    elasticsearch: # Elasticsearch Instance
        container_name: elastic-search
        image: docker.elastic.co/elasticsearch/elasticsearch:7.8.1
        volumes: # Persist ES data in separate "esdata" volume, as it can get quite large
            - /media/database/esdata:/usr/share/elasticsearch/data
            - ./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
        environment:
            - bootstrap.memory_lock=true
            - "ES_JAVA_OPTS=-Xms2g -Xmx2g" # good amount of set aside memory for ES for a small web server
            - discovery.type=single-node
        ports: # Expose Elasticsearch ports
            - "9300:9300"
            - "9200:9200"
    app: # flask app
        container_name: website_dev
        build: # builds in current folder as per usual
            context: .
        ports:
            - "5005:5005" # Expose flask port
        environment: # Set ENV vars
            - FLASK_APP=app.py
            - FLASK_DEBUG=1
            - NODE_ENV=local
            - ES_HOST=elasticsearch
        volumes: # Attach local book data directory
            - /media/database:/media/database # again, depends on some big data
            - .:/code # this is a really neat hack, allows you to change the code in the folder, and have it directly affect the running container for quick dev changes
        depends_on: # makes sure that es is running first
            - elasticsearch
        ulimits: # so there are no crazy core dumps on the server from the app (optional)
            core:
                hard: 0
                soft: 0
    prod: # flask app
        container_name: website_production
        build: # builds in current folder as per usual
            context: .
        ports:
            - "5010:5005" # Expose flask port
        environment: # Set ENV vars
            - FLASK_APP=app.py
            - FLASK_DEBUG=1
            - NODE_ENV=local
            - ES_HOST=elasticsearch
        volumes: # Attach local book data directory
            - /media/database:/media/database # again, depends on some big data
        depends_on: # makes sure that es is running first
            - elasticsearch
        ulimits: # so there are no crazy core dumps on the server from the app (optional)
            core:
                hard: 0
                soft: 0
```

## Basic commands

Some simple commands that will help you here are as follows.  Instead of running the usual `docker build` or `docker run`, you can now replace it with:

```
docker-compose build app #(builds the dev app above)
docker-compose build prod #(builds the production app when you're ready for production stage)
```

You want to be sure you have the `elasticsearch` image, so you should pull it manually with `docker pull docker.elastic.co/elasticsearch/elasticsearch:7.8.1`.

To run your new builds, simply do:

```
docker-compose up -d app
docker-compose up -d prod
```

In this case the `depends-on` part of the `docker-compose.yml` file will allow you to run those containers and it will check that `elasticsearch` is running (and if not, start it) first.
