# Creating an app with `docker-compose` and `elasticsearch`

## Setting up 

### Getting started

So to add a layer of complexity, let's talk about `docker-compose`.  `docker-compose` allows you to use more than one docker container at once for a joint task.  The case I'll discuss here is setting up a search engine using `elasticsearch` (I won't go over how to set up indices, that's very case-by-case).

The `Dockerfile` set up is identical to how it was done in the [first tutorial](https://github.com/jimhavrilla/docs/edit/master/docker/generaluse.md), so you can pretty much reuse it.  Don't forget to clone the [simpleflaskdocker](https://github.com/jimhavrilla/simpleflaskdocker) repo: `git clone https://github.com/jimhavrilla/simpleflaskdocker.git` or `git clone git@github.com:jimhavrilla/simpleflaskdocker.git` depending on your setup.

Before running anything make sure to run (and have `Python 3.7+` or `miniconda3` installed):

```
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.8.1
pip install elasticsearch
```

It took `~33 seconds` on my Macbook Pro 2015 on Wi-Fi.  Hopefully it's as fast for you!

### Running the first commands

First, if on Mac or Linux, run, in the `simpleflaskdocker` folder:

```
mkdir media/database/esdata
```

Windows:

```
md /s media\database\esdata
```

## Understanding the `.yml` file for `docker-compose`

Here's what the `docker-compose.yml` file (with comments, please read) looks like for using the `elasticsearch` docker image with our flask app image. 

```
version: '3.5' # docker-compose version (actually important here)

services:
    elasticsearch: # Elasticsearch Instance
        container_name: elastic-search
        image: docker.elastic.co/elasticsearch/elasticsearch:7.8.1
        volumes: # Persist ES data in separate "esdata" volume, as it can get quite large
            - ./media/database/esdata:/usr/share/elasticsearch/data
            - ./elastic/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
        environment:
            - bootstrap.memory_lock=true
            - "ES_JAVA_OPTS=-Xms500m -Xmx500m" # good amount of set aside memory for ES for a small web server
            - discovery.type=single-node
        ports: # Expose Elasticsearch ports
            - "9200:9200" # make sure these are closed to outside unless you don't want them to be (can also do localhost:9200:9200 or 127.0.0.1:9200:9200)
    app: # flask app
        container_name: website_dev
        build: # builds in current folder as per usual
            context: .
        ports:
            - "5000:5004" # Expose flask port, always machine port:container port in that order.
        environment: # Set ENV vars
            - FLASK_APP=app.py
            - FLASK_DEBUG=1
            - NODE_ENV=local
            - ES_HOST=elasticsearch
        volumes: # Attach local book data directory
            - ./media/database:/media/database # again, depends on some big data
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
            - "5010:5004" # Expose flask port
        environment: # Set ENV vars
            - FLASK_APP=app.py
            - FLASK_DEBUG=1
            - NODE_ENV=local
            - ES_HOST=elasticsearch
        volumes: # Attach local book data directory
            - ./media/database:/media/database # again, depends on some big data
        depends_on: # makes sure that es is running first
            - elasticsearch
        ulimits: # so there are no crazy core dumps on the server from the app (optional)
            core:
                hard: 0
                soft: 0
```

## Basic commands to get started with docker

Some simple commands that will help you here are as follows.  Instead of running the usual `docker build` or `docker run`, you can now replace it with:

```
docker-compose build app #(builds the dev app above, and any time you edit code in the folder, results instantly appear!)
docker-compose build prod #(builds the production app when you're ready for production stage)
```

You want to be sure you have the `elasticsearch` image, so you should pull it manually with `docker pull docker.elastic.co/elasticsearch/elasticsearch:7.8.1`.

To run your new builds, simply do (this will also build them, if no images are present):

```
docker-compose up -d app # -d detaches the container to run in background
docker-compose up -d prod
```

In this case the `depends-on` part of the `docker-compose.yml` file will allow you to run those containers and it will check that `elasticsearch` is running (and if not, start it) first.

Now you have a working production app and a dev app for test that updates instantly!

## Bonus: Indexing with elasticsearch and querying it

Unix users can just run:

```
cd elastic
bash setup.sh
bash run.sh
```

And Windows users can run:

```
chdir elastic
py index.py
py query.py
```

This will create a Lucene index with the data in the file `sampledb.txt` which are some MeSH terms I grabbed from the MeSH database from the NLM.  Then it will query using a somewhat complex fuzzy + exact hybrid search for "Abdominal" and return the top ranked results.

If you go to the ports in your browser, `localhost:5000` and `localhost:5010`, you will see that the "Hello world!" page has changed to display the `elasticsearch` results.
