# Hello-world-from-Django-Gunicorn-Docker-and-NGINX

Introduction
In app development, a critical, make-or-break stage is pushing to production or making an app production-ready. Certain configurations need to be done to ensure there are no breakages, such as security breaches or exposing sensitive configurations such as secret keys. Django web development is no different. How the app runs in development is very different from how it runs in production.

To serve in production, the Django application needs to have:

A stable and reliable web server gateway interface(WSGI) for receiving HTTP requests
A proxy server that can also act as a load balancer in the event the Django app receives heavy HTTP traffic
This guide will explore setting up a Django app with Gunicorn as the WSGI and NGINX as the proxy server. For ease of setup, the guide will package all these using Docker. It therefore assumes that you have at least intermediate level experience with Docker and Docker Compose and at least beginner level skills in Django.

Creating a Sample Django App
The app will be a simple Django app that displays a "hello world" message using an HTTPResponse. The starter code for the app can be found at this github link.

Download and use it for the rest of the guide.

Packaging the Django App Using Docker
For a multi-container application, this activity is done in two stages: 1) developing the Docker file for the main application, and 2) stitching everything up with the rest of the containers using Docker Compose.

App Docker File
The Docker file is simple. It sets up the Django app within its own image.

FROM python:3.8.3-alpine

ENV MICRO_SERVICE=/home/app/microservice
RUN addgroup -S $APP_USER && adduser -S $APP_USER -G $APP_USER
# set work directory


RUN mkdir -p $MICRO_SERVICE
RUN mkdir -p $MICRO_SERVICE/static

# where the code lives
WORKDIR $MICRO_SERVICE

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN apk update \
    && apk add --virtual build-deps gcc python3-dev musl-dev \
    && apk add postgresql-dev gcc python3-dev musl-dev \
    && apk del build-deps \
    && apk --no-cache add musl-dev linux-headers g++
# install dependencies
RUN pip install --upgrade pip
# copy project
COPY . $MICRO_SERVICE
RUN pip install -r requirements.txt
COPY ./entrypoint.sh $MICRO_SERVICE

CMD ["/bin/bash", "/home/app/microservice/entrypoint.sh"]
dockerfile
Project Docker Compose File
Docker Compose will achieve the following:

Spin up the three images: Nginx, Postgres, and Django app image

Define the order of running. The Postgres container will run first, followed by Django container and finally the Nginx container.

To fully build the Nginx container, you need special Docker and conf files for it. Within your sampleApp folder, create a folder named nginx. Within the nginx directory, create a dockerfile and copy the codeblock below:

FROM nginx:1.19.0-alpine

RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d
dockerfile
In the same folder, create a file named nginx.conf and copy the code block below. This is the code that is responsible for setting up nginx.

upstream sampleapp {
    server web:8000;
}

server {

    listen 80;

    location / {
        proxy_pass http://sampleapp;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }
    location /static/ {
        alias /home/app/microservice/static/;
    }

}
conf
After this is done, create the main docker-compose.yml file. This will be the file responsible for running the whole project. In the main project folder, sampleApp, create a file named docker-compose.yml and copy the code block below.

version: '3.7'

services:
  nginx:
    build: ./nginx
    ports:
      - 1300:80
    volumes:
      - static_volume:/home/app/microservice/static
    depends_on:
      - web
    restart: "on-failure"
  web:
    build: . #build the image for the web service from the dockerfile in parent directory
    command: sh -c "python manage.py makemigrations &&
                    python manage.py migrate &&
                    python manage.py initiate_admin &&
                    python manage.py collectstatic &&
                    gunicorn sampleApp.wsgi:application --bind 0.0.0.0:${APP_PORT}"
    volumes:
      - .:/microservice:rw # map data and files from parent directory in host to microservice directory in docker containe
      - static_volume:/home/app/microservice/static
    env_file:
      - .env
    image: sampleapp

    expose:
      - ${APP_PORT}
    restart: "on-failure"
    depends_on:
      - db
  db:
    image: postgres:11-alpine
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
      - PGPORT=${DB_PORT}
      - POSTGRES_USER=${POSTGRES_USER}
    restart: "on-failure"


volumes:
  postgres_data:
  static_volume:
dockerfile
Testing the Live Dockerized App
The whole project is set up and all that remains is to run it. Run the below Docker Compose command to spin up the containers.

docker-compose up --build
bash
To test if the whole project works, from the database, application, and nginx containers, access the app's home page and admin page. The home page, URL 0.0.0.0:1300, should display a simple "hello world" message.

The admin page URL is 0.0.0.0:1300/admin. Use the test credentials:

Username: admin

password: mypass123

You should see a screen like the one below. App hello world

This is what the admin page looks like. Admin page

Conclusion
Making a Django app production-ready just from declarative scripts in Docker is a big advantage for team leads and project managers. It gives them the ability to do configurations beforehand, allowing developers to focus on app development. Not only does this practice create a sense of uniformity in deployment practices, but it also saves vital time by allowing developers to focus on app development and business logic rather than setup and deployment.

Knowing how to package a Django app into an application-ready for deployment in a production environment is vital for developer roles such as DevOps engineers and full-stack with interest in Python for the web.

To further build upon these skills, take up the challenge of deploying the packaged application to a live server. Choose a suitable provider. Options include Linode, GCP, Digital Ocean, and AWS, among other cloud service providers.
