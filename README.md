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
    export VERSION=v5.1
    export ACRNAME=briaracr

    docker build -t hackfest/data-api:$VERSION -f ./app/data-api/Dockerfile ./app/data-api
    docker build -t hackfest/flights-api:$VERSION -f ./app/flights-api/Dockerfile ./app/flights-api
    docker build -t hackfest/quakes-api:$VERSION -f ./app/flights-api/Dockerfile ./app/quakes-api
    docker build -t hackfest/weather-api:$VERSION -f ./app/flights-api/Dockerfile ./app/weather-api
    docker build -t hackfest/service-tracker-ui:$VERSION -f ./app/service-tracker-ui/Dockerfile ./app/service-tracker-ui

    az acr build -t hackfest/data-api:$VERSION -r $ACRNAME ./app/data-api
    az acr build -t hackfest/flights-api:$VERSION -r $ACRNAME ./app/flights-api
    az acr build -t hackfest/quakes-api:$VERSION -r $ACRNAME ./app/quakes-api
    az acr build -t hackfest/weather-api:$VERSION -r $ACRNAME ./app/weather-api
    az acr build -t hackfest/service-tracker-ui:$VERSION -r $ACRNAME ./app/service-tracker-ui
    ```

* Create kubernetes secret with cosmos credentials
    ```
    export MONGODB_URI=''
    export MONGODB_USER=''
    export MONGODB_PASSWORD=''

    kubectl create secret generic cosmos-db-secret --from-literal=uri=$MONGODB_URI --from-literal=user=$MONGODB_USER --from-literal=pwd=$MONGODB_PASSWORD
    ```

* Deploy app (without Service Mesh)
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

    via bash

    ```
    export APP_URL=http://23.96.62.115:8080/#/flights
    export APP_URL=http://137.135.101.232:3003/latest
    while true; do curl -o /dev/null -s -w "%{http_code}\n" $APP_URL; sleep 1; done
    ```

    via container

    ```
    docker run -d --name load-test1 -e "load_duration=-1" -e "load_rate=1" -e "load_url=137.135.101.232:3003/latest" chzbrgr71/loadtest

    az container create --name load-test1 --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=5 load_url=137.135.101.232:3003/latest

    az container delete --yes --resource-group aci --name load-test1
    ```

    ```
    for i in 1 2 3 4 5; do
        az container create --name flights-load-test${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=1 load_url=40.121.107.138:3003/latest
        az container create --name quakes-load-test${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=1 load_url=40.114.68.113:3012/latest
    done

    for i in 1 2 3 4 5; do
        az container delete --yes --resource-group aci --name flights-load-test${i}
        az container delete --yes --resource-group aci --name quakes-load-test${i}
    done

    # single instance
    az container create --name flights-load-test1 -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=40.121.107.138:3003/latest
    az container create --name quakes-load-test1 -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=40.114.68.113:3012/latest

    az container delete --yes --resource-group aci --name flights-load-test1
    az container delete --yes --resource-group aci --name quakes-load-test1
    ```
    Web UI: http://40.76.10.96:8080 


    Change deployment image tag:
    ```
    kubectl set image deployment/quakes-api quakes-api=briaracr.azurecr.io/hackfest/quakes-api:v5

    kubectl set image deployment/quakes-api quakes-api=briaracr.azurecr.io/hackfest/quakes-api:v5.1
    kubectl set image deployment/flights-api flights-api=briaracr.azurecr.io/hackfest/flights-api:v5.1
    ```