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

In Kubernetes, an app deployed in the cluster is called a **Pod**. To check for Pod successfully created use following:

```
kubectl get pods
```

`minikube` also provide dashboard to visualize created pods, etc. Open dashboard:

```
minikube dashboard
```

This will open the dashboard and from there you can see vizualize information.

## Exposing the application with Services

Bear in mind, the deployed app(POD) has dynamics IP, means everytime we scale and deployed, system will assigned dynamic IP to pod makes hard to route the traffic. To overcome this, we can use Service(load balancer which has fixed IP address) in front of Pods. Worry not, k8s provide services load balancer. We can create service with following code:

```
kubectl expose pods laravel-app --type=NodePort --port=80
```

Verify if service was successfully created:

```
kubectl get services
```

We also can view services that has been created using minikube dashboard. To obtain service's application URL, issued following command:

```
minikube service --url=true laravel-app
```

With above command, you will get URL something like this `http://127.0.0.1:51766`, if you open up this URL, it should display your working laravel app. To launch directly into browser without get the url, issue following command:

```
minikube service laravel-app
```

At this point, we should have a local k8s cluster with:

- A single Pod running
- A service that route traffic to a Pod (load balance with NodePort type)

But, having a single Pod running is not enough, what if Pod accidentally deleted?. We can try it out by deleting existing Pod:

```
kubectl delete pod laravel-app
```

If we browsing again service's URL previously, we couldnt see our working laravel app anymore. It'd be better if there could be a mechanism to watch Pods and restart them when they are deleted, or they crash. Let's create a deployment definition, create this file and put inside `k8s` folder (create this folder):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-kubernetes-demo
spec:
  selector:
    matchLabels:
      run: laravel-kubernetes-demo
  template:
    metadata:
      labels:
        run: laravel-kubernetes-demo
    spec:
      containers:
        - name: demo
          image: <your-username>/laravel-kubernetes-demo
          ports:
            - containerPort: 80
          env:
            - name: APP_KEY
              value: base64:cUPmwHx4LXa4Z25HhzFiWCf7TlQmSqnt98pnuiHmzgY=
```              

Once done, submit this deployment into running cluster. 

```
kubectl apply -f k8s/deployment.yaml
```

Try open up again URL, you should see it is working. Try delete the pods and by now, Pod will automatically respawn/created. Hooray!

## Scaling the application

It is time to scale our application(running on multiple Pods instead of single Pods) into more instances. This way, we could have a backup running application, if one crashed, we got it another covered. Use following command to scale:

```
kubectl scale --replicas=3 deployment/laravel-app
```

We can verify it and by that time, we can see 3 pods is running (can see directly via dashboard):

```
kubectl get deployment,pods
```

### Using Nginx Ingress to expose the app

To use a URL in Kubernetes like we would normally access website via domain, we need an Ingress. An Ingress is a set of rules to allow inbound connections to reach a Kubernetes cluster. Old way, we might have used Nginx or Apache as a reverse proxy to reroute traffic to minikube's cluster IP address. See, from here we can say that `The Ingress is the equivalent of a reverse proxy in Kubernetes.`

Let create one, put under `k8s` folder with name `ingres.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: laravel-app-ingress
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: laravel-app
                port:
                  number: 80
```

Among the basic content you would expect from a Kubernetes resource file, this file defines a set of rules to follow when routing inbound traffic. The Ingress resource is useless without an Ingress controller so we will need to create a new controller or use an existing one.

Minikube comes with the Nginx as Ingress controller, and you can enable it with:

```
minikube addons enable ingress
```
