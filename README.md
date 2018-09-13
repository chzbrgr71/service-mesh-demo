# Service Mesh Demos

Demo application for upcoming events.


### Application Setup

* Create an Azure Kubernetes Service cluster (use the docs). I used k8s version 1.10.6

* Create an instance of Azure CosmosDB
    ```
    export RGNAME=''
    export COSMOSNAME=''

    az cosmosdb create --name $COSMOSNAME --resource-group $RGNAME --kind MongoDB
    ```

* Create kubernetes secret with cosmos credentials
    ```
    export MONGODB_URI=''
    export MONGODB_USER=''
    export MONGODB_PASSWORD=''

    kubectl create secret generic cosmos-db-secret --from-literal=uri=$MONGODB_URI --from-literal=user=$MONGODB_USER --from-literal=pwd=$MONGODB_PASSWORD
    ```

* Create images
    ```
    export VERSION=2.0
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


* Test deploy app without service mesh
    ```
    kubectl apply -f ./k8s/deploy-app.yaml
    ```

* Load data into Cosmos for each api
    ```
    kubectl get svc

    NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
    data-api             LoadBalancer   10.0.189.95    40.114.33.42     3009:30176/TCP   8m
    flights-api          LoadBalancer   10.0.204.253   40.76.219.242    3003:31739/TCP   8m
    kubernetes           ClusterIP      10.0.0.1       <none>           443/TCP          43m
    quakes-api           LoadBalancer   10.0.200.23    40.117.157.117   3012:30004/TCP   8m
    service-tracker-ui   LoadBalancer   10.0.82.227    40.76.210.156    8080:31798/TCP   8m
    weather-api          LoadBalancer   10.0.47.46     40.114.29.147    3015:32186/TCP   8m
    
    eg: http://40.76.219.242:3003/refresh
    ```

* Remove app to re-deploy with service mesh
    ```
    kubectl apply -f ./k8s/deploy-app.yaml
    ```

### Linkerd2

https://linkerd.io/2/getting-started 

* Install the CLI

* Install control plane: ```linkerd install | kubectl apply -f -```

* Add the app
    ```
    linkerd inject ./k8s/deploy-app.yaml | kubectl apply -f -
    ```

### Istio (optional)

Using istio release 1.0.1

https://istio.io/docs/setup/kubernetes/helm-install

* Install via Helm

kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml

helm install install/kubernetes/helm/istio --name istio --namespace istio-system --set grafana.enabled=true --set servicegraph.enabled=true --set tracing.enabled=true

kubectl label namespace default istio-injection=enabled
kubectl get namespace -L istio-injection

* Add Egress rule 
    ```
    kubectl apply -f ./k8s/istio-egress.yaml
    ```


### Do Stuff

* Load test

    via bash

    ```
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
        az container create --name flights-load-test${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=40.76.219.242:3003/latest
        az container create --name quakes-load-test${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=40.117.157.117:3012/latest
        az container create --name weather-load-test${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=40.114.29.147:3015/latest
    done

    for i in 1 2 3 4 5; do
        az container delete --yes --resource-group aci --name flights-load-test${i}
        az container delete --yes --resource-group aci --name quakes-load-test${i}
        az container delete --yes --resource-group aci --name weather-load-test${i}
    done

    # single instance
    az container create --name flights-load-test1 -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=40.76.219.242:3003/latest
    az container create --name quakes-load-test1 -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=40.117.157.117:3012/latest
    az container create --name weather-load-test1 -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=40.114.29.147:3015/latest

    az container delete --yes --resource-group aci --name flights-load-test1
    az container delete --yes --resource-group aci --name quakes-load-test1
    az container delete --yes --resource-group aci --name weather-load-test1
    ```
    Web UI: http://40.76.210.156:8080


    Change deployment image tag:
    ```
    kubectl set image deployment/quakes-api quakes-api=briaracr.azurecr.io/hackfest/quakes-api:v5

    kubectl set image deployment/quakes-api quakes-api=briaracr.azurecr.io/hackfest/quakes-api:v5.1
    kubectl set image deployment/flights-api flights-api=briaracr.azurecr.io/hackfest/flights-api:v5.12
    ```

### 404 Errors

```
router.get('/latest', (req, res, next) => {

    async.waterfall([
        (cb) => {
            // get latest timestamp from DB
            console.log('getting latest timestamp of flights')
            var path = 'get/latest/flights'
            getFromDataApi(path, (e, d) => {
                cb(null, d.payload[0].Timestamp)
            })
        },
        (timestamp, cb) => {
            // use latest timestamp for flights from DB
            console.log('getting latest flights based on timestamp')
            var path = 'get/flights/' + timestamp
            getFromDataApi(path, (e, d) => {
                cb(null, d.payload.FeatureCollection)
            })

        }
    ],(e,r) => {
        //jsonResponse.json( res, st.OK.msg, st.OK.code, r)
        //res.json({}).status(404) 
        res.sendStatus(404)
    })

})
```