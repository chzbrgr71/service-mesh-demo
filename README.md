# Service Mesh Demos

Demo application for upcoming events.


### Docker Setup

docker build -t chzbrgr71/data-api:v1 -f ./app/data-api/Dockerfile ./app/data-api
docker build -t chzbrgr71/flights-api:v1 -f ./app/flights-api/Dockerfile ./app/flights-api
docker build -t chzbrgr71/geo-dashboard:v1 -f ./app/geo-dashboard/Dockerfile ./app/geo-dashboard

kubectl apply -f deploy-app.yaml