## Download sample repo

Let's start with making a small Flask app, as a really simple use case of `docker`. Let's clone a [really simplistic Flask app](https://github.com/jimhavrilla/simpleflaskdocker). You can use this to have something to run continuously out of the docker container.

## Breaking down the Dockerfile 

Now you have an app that is really small and just prints "Hello world!" Let's break down a `Dockerfile`. I prefer something like this:

```
# I prefer using a CentOS image as most servers use CentOS or RHEL, and currently 3.8 is the most stable, up-to-date Python version, this pulls from dockerhub
FROM centos/python-38-centos7
# Often times, you will have run things with "sudo" so it is just easier to make yourself root when creating the image so installing with yum is easier
USER root
# yum is the package manager for CentOS, most things you need will require these things installed for C code compilation (and git and curl)
RUN yum install musl-dev linux-headers g++ gcc git curl
RUN yum clean all
# If using Python it is smaller and faster to use pip to build images than conda, and requirements.txt is a standard way to store package names and versions for this
# make sure pip is up-to-date, and wheel is installed (sometimes it isn't)
COPY requirements.txt requirements.txt
RUN python -m pip install --upgrade pip
RUN python -m pip install wheel
RUN pip install -r requirements.txt
# you can make the Workdir whatever you want (/app, /code, etc), but it is where everything will run from
WORKDIR /code
# not always necessary, but useful for docker-compose, uWSGI
ENV FLASK_APP app.py
ENV FLASK_RUN_HOST 0.0.0.0
# copies the current directory to our Workdir
COPY . /code
# runs a very small script that contains something like "python app.py"
RUN chmod +x /code/app_init.sh
ENTRYPOINT sh /code/app_init.sh
```

It can be a lot to digest at first, but knowing what to put in your Dockerfile is essential to how you build your image.  One piece of advice is to have all the code you use to actually make the image in a separate folder if you're planning on putting several files in the image.  Best practices tell you to mount with `-v` or `--mount`, but sometimes you may need to internalize the data in the image if you are less familiar with the code and how to mount data into it.

## Building a Docker image

So after cloning my sample repo above, to build the docker, run:

```
docker build -t simpledocker .
```

This will build an image using the tag name "simpledocker" (`-t` flag) and use the current folder `.` to do it.  Most of the time this is the command you will run for `docker build`, with the tag name changed of course.

This should take no time at all to run.

## Running our Docker image

So, the `docker run` command is one that is not broken down enough.  To create a working container that talks to your localhost port 5000, you can run:

```
docker run -d --rm -p 5000:5004 simpledocker
```

Now, let's break down the important flags:
- `-d` flag detaches the image
- `it` lets you go inside the image so you can interact with it (kind of the opposite of `-d`)
- `--rm` removes the container automatically when it is killed, fails, or stops
- `-p` sets the port, this confuses many people.  It goes `p externalport:internalport` so in my command you will be able to see the "Hello World!" message in your `localhost:5000`, and uses port `5004` inside the docker container. So you can tell flask to run in any port.  My `app.py` sample script uses port `5004` for no particular reason.  You can change that to `4001`, `7777`, whatever you like and if you still want to view it at `localhost:5000` the `-p` command is `5004:4001`, `5004:7777` etc.
- `-v` you can add this to mount a volume to docker like `-v /somewhereIhaveHUGEdata:/code` if you want a folder to be accessible inside the container's workdir for example, but not built into the image (which is good practice)
- lastly, you type the name of the image you wish to run as a container, in this case, `simpledocker`

## Examining your containers, stopping them, and cleaning up

If you want to see the containers running, just run:

```
docker ps
```

You will see output that looks like this:

```
CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS              PORTS                                            NAMES
f5047dced54a        simpledocker                                          "/bin/sh -c 'sh /cod…"   3 hours ago         Up 3 hours          8080/tcp, 0.0.0.0:5000->5004/tcp                 priceless_hermann
```

Now, let's kill the poor, innocent container that did nothing wrong by either copying the container ID (first 5 characters is usually fine) `f5047dced54a` or the name `priceless_hermann`:

```
docker kill f5047dced54a # or docker kill priceless_hermann
```

If you run the container without `--rm`:

```
docker run -d -p 5000:5004 simpledocker
```

you can see it be killed with:

```
docker ps -a # SHOWS ALL CONTAINERS, EVEN KILLED
```

Which shows you:

```
CONTAINER ID        IMAGE                                                 COMMAND                  CREATED              STATUS                       
5b81582e3749        simpledocker                                          "/bin/sh -c 'sh /cod…"   About a minute ago   Exited (137) 4 seconds ago 
```

And if you wanted to look at the logs of a stopped or running container, you can use:

```
docker logs 5b81582e3749
```

Which will look like:

```
docker logs 5b81582e3749
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
 * Running on http://0.0.0.0:5004/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 255-331-439
```

You can look inside the container as well by doing (h/t to [Thorsten Hans](https://thorsten-hans.com/how-to-run-commands-in-stopped-docker-containers)):

```
docker commit 5b81582e3749 test
docker images # should show test as a new image now
docker run -it --rm --entrypoint sh test
```

And then you can run commands inside of the stopped container and read internal logs, use `ls` etc and debug what could have gone wrong.

Lastly, you can clean it up with:

`docker container prune`

And enter `y`.

Also to get rid of this image, find it with:

```
docker images
```

Gives you something like:

```
REPOSITORY                                      TAG                 IMAGE ID            CREATED             SIZE
simpledocker                                    latest              a510b6807d75        3 hours ago         818MB
```

Then take the ID or the tag name: 

```
docker rmi simpledocker # or docker rmi a510b6807d75 or docker rmi simpledocker:latest if you made different versions of the same image
```

This is really useful when you have a bunch of errors in your builds (which you will) so you don't eat up all your memory or disk space.

Lastly, if you know what you're doing, you can just do most of the pruning automatically with:

```
docker system prune
```

Which will remove all stopped containers and untagged images.
