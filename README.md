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
    export VERSION=v3
    export ACRNAME=briaracr

    docker build -t chzbrgr71/data-api:$VERSION -f ./app/data-api/Dockerfile ./app/data-api
    docker push chzbrgr71/data-api:$VERSION
    az acr build -t chzbrgr71/data-api:$VERSION -r $ACRNAME ./app/data-api
    
    docker build -t chzbrgr71/flights-api:$VERSION -f ./app/flights-api/Dockerfile ./app/flights-api
    docker push chzbrgr71/flights-api:$VERSION
    az acr build -t chzbrgr71/flights-api:$VERSION -r $ACRNAME ./app/flights-api

    docker build -t chzbrgr71/service-tracker-ui:$VERSION -f ./app/service-tracker-ui/Dockerfile ./app/service-tracker-ui
    docker push chzbrgr71/geo-dashboard:$VERSION
    az acr build -t chzbrgr71/service-tracker-ui:$VERSION -r $ACRNAME ./app/service-tracker-ui
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
    kubectl apply -f ./k8s/deploy-app.yaml
    ```

* Load data into Cosmos
    ```
    kubectl port-forward flights-api-5d4886f788-rwbdq 3003:3003
    http://localhost:3003/refresh
    ```

### Linkerd 2.0

https://linkerd.io/2/getting-started 

* Install the CLI

* Install control plane: ```linkerd install | kubectl apply -f -```

* Add the app
    ```
    linkerd inject ./k8s/deploy-app.yaml | kubectl apply -f -
    ```

### Istio

Using istio release 1.0.1

https://istio.io/docs/setup/kubernetes/helm-install

* Install via Helm

kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml

helm install install/kubernetes/helm/istio --name istio --namespace istio-system --set grafana.enabled=true --set servicegraph.enabled=true --set tracing.enabled=true

kubectl label namespace default istio-injection=enabled
kubectl get namespace -L istio-injection

* Add Egress rules. 
    ```
    kubectl apply -f ./k8s/istio-egress.yaml
    ```


### Do Stuff

* Load test
    ```
    export APP_URL=http://137.135.113.90:8080/#/flights
    export APP_URL=http://40.117.123.212:3003/current
    export APP_URL=http://40.117.121.21:3009/get/flights/201808272052
    
    export APP_URL=http://137.135.118.92:8080/#/flights
    export APP_URL=http://40.121.7.125:3003/current

    while true; do curl -o /dev/null -s -w "%{http_code}\n" $APP_URL; sleep 1; done
    ```