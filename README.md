# kubernetes-deployment
Step by step to deploy simple laravel app on top of k8s.

## Requirements
- minikube (local k8s) - https://minikube.sigs.k8s.io/docs/
- kubectl (CLI to manage pods, etc) - https://kubernetes.io/docs/tasks/tools/
- docker (Container registry) - https://docs.docker.com/get-docker/
- Working laravel app (Fresh installation of laravel app will do)

## Create Dockerfile and build the container

Create docker file at the root of your project. `vim Dockerfile` and paste following code:

```
# Manage composer dependencies
FROM composer:1.6.5 as build
WORKDIR /app
COPY . /app
RUN composer install

# Configure apache web server with minimal setup
FROM php:7.1.8-apache
EXPOSE 80
COPY --from=build /app /app #copy over from first container build
COPY vhost.conf /etc/apache2/sites-available/000-default.conf
RUN chown -R www-data:www-data /app a2enmod rewrite
```

After that, build the container so we can run it later.

```
docker build -t laravel-app .

-t laravel-app define the name the container called laravel-app
. is the location of the Dockerfile and application code, in this case, it's the current directory
```

To test it, we need to run the container

```
docker run -ti \
  -p 8080:80 \
  -e APP_KEY=base64:cUPmwHx4LXa4Z25HhzFiWCf7TlQmSqnt98pnuiHmzgY= \
  laravel-app
```
