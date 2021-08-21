# kubernetes-deployment
Step by step to deploy simple laravel app on top of k8s.

## Requirements
- minikube (local k8s, run minikube start to configure local cluster) - https://minikube.sigs.k8s.io/docs/
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
Now you can access laravel app from browser using this url http://localhost:8080.

To make it available for k8s cluster later, we need to upload the above container into docker hub. Run `docker login` and follow on screen instructions. Once success, upload to docker hub:

```
# names the container
docker tag laravel-app <your-username>/laravel-app

# upload to docker hub
docker push <your-username>/laravel-app

# this will make container publicly accessible to other users. Use paid account to make it private :)
```

Now those image is publicly available as `<your-username>/laravel-app` on Docker Hub and anyoone can download it. Previously we run the container using local container, to test the new image that we upload run again following command:

```
docker run -ti \
  -p 8080:80 \
  -e APP_KEY=base64:cUPmwHx4LXa4Z25HhzFiWCf7TlQmSqnt98pnuiHmzgY= \
  <your-username>/laravel-app
```

## Deploying app with k8s

Now the docker image is built & available, we can deploy container image with following code:

```
kubectl run laravel-app \
  --restart=Never \
  --image=<your-username>/laravel-app \
  --port=80 \
  --env=APP_KEY=base64:cUPmwHx4LXa4Z25HhzFiWCf7TlQmSqnt98pnuiHmzgY=
  
  
* `kubectl run laravel-app` deploys an app in the cluster and gives it the name laravel-app.
* --restart=Never is used not to restart the app when it crashes.
* --image=<your-username>/laravel-app and --port=80 are the name of the image and the port exposed on the container.
```
