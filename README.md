# Service Mesh Demos

Demo application for upcoming events.


### Application Setup

* Create an instance of Azure CosmosDB
    ```
    export RGNAME=''
    export COSMOSNAME=''

    az cosmosdb create --name $COSMOSNAME --resource-group $RGNAME --kind MongoDB
    ```

* Create images
    ```
    export VERSION=v2

    docker build -t chzbrgr71/data-api:$VERSION -f ./app/data-api/Dockerfile ./app/data-api
    docker push chzbrgr71/data-api:$VERSION

    docker build -t chzbrgr71/flights-api:$VERSION -f ./app/flights-api/Dockerfile ./app/flights-api
    docker push chzbrgr71/flights-api:$VERSION

    docker build -t chzbrgr71/geo-dashboard:$VERSION -f ./app/geo-dashboard/Dockerfile ./app/geo-dashboard
    docker push chzbrgr71/geo-dashboard:$VERSION
    ```

* Create kubernetes secret with cosmos credentials
    ```
    export MONGODB_URI=''
    export MONGODB_USER=''
    export MONGODB_PASSWORD=''

    kubectl create secret generic cosmos-db-secret --from-literal=uri=$MONGODB_URI --from-literal=user=$MONGODB_USER --from-literal=pwd=$MONGODB_PASSWORD
    ```

* Deploy app
    ```
    kubectl apply -f deploy-app.yaml
    ```

* Load data into Cosmos
    ```
    kubectl port-forward flights-api-5d4886f788-rwbdq 3003:3003
    http://localhost:3003/refresh
    ```