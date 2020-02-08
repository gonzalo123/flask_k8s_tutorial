## Deploying Python Application using Docker and Kubernetes

I've learning how to deploy one Python application to Kubernetes. Here you can see my notes:

Let's start from a dummy Python application. It's a basic Flask web API. Each time we perform a GET request to "/" we
 increase one counter and see the number of hits. The persistence layer will be a Redis database. The script is very
  simple:

```python
from flask import Flask
import os
from redis import Redis

redis = Redis(host=os.getenv('REDIS_HOST', 'localhost'),
              port=os.getenv('REDIS_PORT', 6379))
app = Flask(__name__)


@app.route('/')
def hello():
    redis.incr('hits')
    hits = int(redis.get('hits'))
    return f"Hits: {hits}"


if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

First of all we create a virtual environment to ensure that we're going to install our dependencies isolately:

> python -m venv venv

We enter in the virtualenv

> source venv/bin/activate

And we install our dependencies:

> pip install -r requirements.txt

To be able to run our application we must ensure that we've a Redis database ready. We can run one with Docker:

> docker run -p 6379:6379 redis

Now we can start our application:

> python app.py

We open our browser with the url: http://localhost:5000 and it works.

Now we're going to run our applwwication within a Docker container. First of of all we need to create one Docker image from a docker file:
 
```dockerfile
FROM python:alpine3.8
ADD . /code
WORKDIR /code
RUN pip install -r requirements.txt

EXPOSE 5000
```

Now we can build or image:

> docker build -t front .

And now we can run our front image:

> docker run -p 5000:5000 front python app.py

If we open now our browser with the url http://localhost:5000 we'll get a 500 error. That's because our Docker container is trying to use one Redis host within localhost. It worked before, when our application and our Redis were within the same host. Now our API's localhost isn't the same than our host's one. 
  
Our script the Redis host is localhost by default but it can be passed from an environment variable,
  
```python
redis = Redis(host=os.getenv('REDIS_HOST', 'localhost'),
              port=os.getenv('REDIS_PORT', 6379))
```

we can pass to our our Docker container the real host where our Redis resides (suposing my IP address is 192.168.1.100):

> docker run -p 5000:5000 --env REDIS_HOST=192.168.1.100 front python app.py

If dont' want the development server we also can start our API using gunicorn

> docker run -p 5000:5000 --env REDIS_HOST=192.168.1.100 front gunicorn -w 1 app:app -b 0.0.0.0:5000

And that works. We can start our app manually using Docker. But it's a bit complicated. We need to run two containers (API and Redis), setting up the env variables.
Docker helps us with docker-compose. We can create a docker-compose.yaml file configuring our all application:

```yaml
version: '2'

services:
  front:
    image: front
    build:
      context: ./front
      dockerfile: Dockerfile
    container_name: front
    command: gunicorn -w 1 app:app -b 0.0.0.0:5000
    depends_on:
      - redis
    ports:
      - "5000:5000"
    restart: always
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379
  redis:
    image: redis
    ports:
      - "6379:6379"
```

Docker compose is pretty straightfordward. But, what happens if our production environment is a cluster? docker-compose works fine in a single host. But it our production environment is a cluster, we´ll face problems (we need to esure manually things like hight avaiavility and things like that). Docker people tried to answer to this question with Docker Swarm. Basically Swarm is docker-compose within a cluster. It uses almost the same syntax than docker-compose in a single host. Looks good, ins't it? OK. Nobody uses it. Since Google created their Docker conainer orchestator (Kubernetes, aka K8s) it becames into the de-facto standard. The good thing about K8s is that it's much better than Swarm (more configurable and more powerfull), but the bad part is that it isn't as simple and easy to understand as docker-compose.

Let's try to execute our proyect in K8s:

First I start minikube

> minikube start

and I configure kubectl to connect to my minikube k8s cluster

> eval $(minikube docker-env)

The API:

First we create one service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: front-api
spec:
  # types:
  # - ClusterIP: (default) only accessible from within the Kubernetes cluster
  # - NodePort: accessible on a static port on each Node in the cluster
  # - LoadBalancer: accessible externally through a cloud provider's load balancer
  type: LoadBalancer
  # When the node receives a request on the static port (30163)
  # "select pods with the label 'app' set to 'front-api'"
  # and forward the request to one of them
  selector:
    app: front-api
  ports:
    - protocol: TCP
      port: 5000 # port exposed internally in the cluster
      targetPort: 5000 # the container port to send requests to
      nodePort: 30164 # a static port assigned on each the node

```

And one deploymente:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front-api
spec:
  # How many copies of each pod do we want?
  replicas: 1

  selector:
    matchLabels:
      # This must match the labels we set on the pod!
      app: front-api

  # This template field is a regular pod configuration
  # nested inside the deployment spec
  template:
    metadata:
      # Set labels on the pod.
      # This is used in the deployment selector.
      labels:
        app: front-api
    spec:
      containers:
        - name: front-api
          image: front:v1
          args: ["gunicorn", "-w 1", "app:app", "-b 0.0.0.0:5000"]
          ports:
            - containerPort: 5000
          env:
            - name: REDIS_HOST
              valueFrom:
                configMapKeyRef:
                  name: api-config
                  key: redis.host

```

In order to learn a little bit of K8s I'm using a config map called 'api-config' where I put some information (such as the Redis host that I'm going to pass as a env variable)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  redis.host: "back-api"
```

The Backend: Our Redis database:

First the service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: back-api
spec:
  type: ClusterIP
  ports:
    - port: 6379
      targetPort: 6379
  selector:
    app: back-api
```

And finally the deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: back-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: back-api
  template:
    metadata:
      labels:
        app: back-api
    spec:
      containers:
        - name: back-api
          image: redis
          ports:
            - containerPort: 6379
              name: redis
```

Before deploying my application to the cluster I need to build my docker image and tag it

> docker build -t front .

> docker tag front front:v1

Now I can deploy my application to my K8s cluster:

> kubectl apply -f .k8s/

If want to know what's the external url of my application in the cluster I can use this command

> minikube service front-api --url

Then I can see it running using the browser or with curl

> curl $(minikube service front-api --url)

And that's all. I can delete all application alos

> kubectl delete -f .k8s/ 