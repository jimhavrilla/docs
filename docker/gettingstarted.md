Let's start with making a small Flask app, as a really simple use case of `docker`. Let's clone a [really simplistic Flask app](https://github.com/shekhargulati/python-flask-docker-hello-world). You can use this to have something to run continuously out of the docker container.

Now you have an app that is really small and just prints "Flask inside Docker!!" Let's break down a `Dockerfile`. I prefer something like this:

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
