## Arranco app manualmente
Supongo que tengo un redis funcionando en el puerto 6739 (docker run -p 6379:6379 redis)
> python app.py

## Arranco app con docker
creo la imagen docker a partir del Dockerfile
> docker build -t front .

Lo ejecuto Ejecuto
> docker run -p 5000:5000 front python app.py

No funciona. No puede acceder a Redis en localhost. Lo arranco pasándole el host donde está Redis

> docker run -p 5000:5000 --env REDIS_HOST=192.168.1.76 front python app.py

También lo puedo arrancar con gunicorn
> docker run -p 5000:5000 --env REDIS_HOST=192.168.1.76 front gunicorn -w 1 app:app -b 0.0.0.0:5000

## Arranco la app con docker-compose

> docker-compose up

## Despliego la app en k8s

Arranco minikube
> minikube start

Configuro kubectl para que se conecte al cluster k8s
> eval $(minikube docker-env)

Hago build de la imagen 

> docker build -t front .

> docker tag front front:v1

Despliego la aplicación
> kubectl apply -f .k8s/

Obtengo la url de la aplicación en el cluster
> minikube service front-api --url

> curl $(minikube service front-api --url)

La borro del cluster
> kubectl delete -f .k8s/ 

```
minikube service list
minikube dashboard
```