# Service Mesh Demos

Demo application for upcoming events.


### Application Setup

* Create an Azure resource group.

    ```
    RG_NAME=service-mesh-demo
    LOCATION=westeurope

    az group create -n $RG_NAME -l $LOCATION
    ```

* Create an Azure Kubernetes Service cluster. I used k8s version 1.10.6.

    > Note: I used my own pre-created service principal. You can choose to leave this out and the CLI will create one for you.

    ```
    RG_NAME=service-mesh-demo
    CLUSTER_NAME=aks-service-mesh
    LOCATION=westeurope
    SP_ID=<replace>
    SP_SECRET=<replace>
    K8S_VERSION=1.10.6
    VM_SIZE=Standard_D2_v2

    az aks create --node-vm-size $VM_SIZE -n $CLUSTER_NAME -g $RG_NAME --kubernetes-version $K8S_VERSION --node-count 5 --dns-name-prefix $CLUSTER_NAME --service-principal $SP_ID --client-secret $SP_SECRET --no-wait
    ```

* Access AKS

    ```
    az aks get-credentials -n $CLUSTER_NAME -g $RG_NAME

    kubectl get nodes
    ```


* Install Helm (only needed for Istio) http://helm.sh 

* Create an instance of Azure CosmosDB
    ```
    export RG_NAME=''
    export COSMOSNAME=''

    az cosmosdb create --name $COSMOSNAME --resource-group $RG_NAME --kind MongoDB
    ```

* Create kubernetes secret with cosmos credentials

    > Note: the MONGODB_URI should be of the format below **(Ensure you add the `/hackfest?ssl=true`)** at the end.

    mongodb://cosmosbrian11199:ctumHIz1jC4Mh1hZgWGEcLwlCLjDSCfFekVFHHhuqQxIoJGiQXrIT1TZTllqyB4G0VuI4fb0qESeuHCRJHA==@acrhcosmosbrian11122.documents.azure.com:10255<span style="color:blue">/hackfest</span>?ssl=true

    ```
    export MONGODB_URI=''
    export MONGODB_USER=''
    export MONGODB_PASSWORD=''

    kubectl create secret generic cosmos-db-secret --from-literal=uri=$MONGODB_URI --from-literal=user=$MONGODB_USER --from-literal=pwd=$MONGODB_PASSWORD
    ```

* Create Azure Container Registry

    ```
    export ACRNAME=briaracreu

    az acr create --resource-group $RG_NAME --name $ACRNAME --sku Basic
    ```

* Create images

    ```
    export ACRNAME=briaracreu
    export VERSION=2.5-istio
    export VERSION=weather-404
    export VERSION=quakes-slow

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
    
    browse to: http://<EXTERNAL-IP>:3003/refresh
    ```

* Remove app to re-deploy with service mesh
    ```
    kubectl delete -f ./k8s/deploy-app.yaml
    ```

### Linkerd2

https://linkerd.io/2/getting-started 

* Install the CLI

* Install control plane: 

    ```
    linkerd install | kubectl apply -f -
    ```

    * Or with TLS
        ```
        linkerd install --tls=optional | kubectl apply -f -
        ```

* Deploy the app with linkerd sidecar injected
    ```
    linkerd inject ./k8s/deploy-app.yaml | kubectl apply -f -

    # aks-linkerd-eu-100
    linkerd inject ./k8s/deploy-app-quakes-slow.yaml | kubectl apply -f -
    OR
    linkerd inject --tls=optional ./k8s/deploy-app-quakes-slow.yaml | kubectl apply -f -

    # aks-linkerd-eu-101
    linkerd inject ./k8s/deploy-app-weather-404.yaml | kubectl apply -f -
    OR
    linkerd inject --tls=optional ./k8s/deploy-app-weather-404.yaml | kubectl apply -f -
    ```

    to remove: 
        
    ```
    linkerd inject ./k8s/deploy-app-quakes-slow.yaml | kubectl delete -f -
    linkerd inject ./k8s/deploy-app-weather-404.yaml | kubectl delete -f -
    ```

### Istio (optional)

Using istio release 1.0.1

https://istio.io/docs/setup/kubernetes/helm-install

* Install via Helm

    ```
    # can potentially skip this step if using Helm below

    kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
    ```

    ```
    # run from the istio release root directory

    helm install install/kubernetes/helm/istio --name istio --namespace istio-system --set grafana.enabled=true --set servicegraph.enabled=true --set tracing.enabled=true
    ```

    ```
    helm install --name istio install/kubernetes/helm/istio --namespace istio-system \
    --set global.crds=false \
    --set global.controlPlaneSecurityEnabled=true \
    --set global.mtls.enabled=true \
    --set grafana.enabled=true \
    --set tracing.enabled=true \
    --set galley.enabled=false \
    --values install/kubernetes/helm/istio/values.yaml
    ```

    ```
    kubectl label namespace default istio-injection=enabled
    kubectl get namespace -L istio-injection
    ```

* Update istio egress rules in `deploy-app-istio.yaml` to match the external IP's for your services (mainly Cosmos)

* Install

    ```
    kubectl apply -f ./k8s/deploy-app-istio.yaml
    ```

### Do Stuff

* Load test

    via bash

    ```
    export APP_URL=http://13.68.196.193/latest
    
    while true; do curl -o /dev/null -s -w "%{http_code}\n" $APP_URL; sleep 1; done
    ```

    via container

    ```
    docker run -d --name load-test1 -e "load_duration=-1" -e "load_rate=1" -e "load_url=137.135.101.232:3003/latest" chzbrgr71/loadtest

    az container create --name load-test1 --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=5 load_url=13.68.196.193/latest

    az container delete --yes --resource-group aci --name load-test1
    ```

    ```
    # aks-linkerd-eu-100
    export FLIGHTS_IP=104.41.132.175
    export QUAKES_IP=40.114.86.117
    export WEATHER_IP=137.117.109.171
    
    for i in 1 2 3; do
        az container create --name flights-load-test${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$FLIGHTS_IP:3003/latest
        az container create --name quakes-load-test${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$QUAKES_IP:3012/latest
        az container create --name weather-load-test${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$WEATHER_IP:3015/latest
    done

    for i in 1 2 3; do
        az container delete --yes --resource-group aci --name flights-load-test${i}
        az container delete --yes --resource-group aci --name quakes-load-test${i}
        az container delete --yes --resource-group aci --name weather-load-test${i}
    done

    # aks-linkerd-eu-101
    export FLIGHTS_IP=23.96.16.35
    export QUAKES_IP=168.62.172.68
    export WEATHER_IP=23.96.18.179

    for i in 1 2 3; do
        az container create --name flights-load-testb${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$FLIGHTS_IP:3003/latest
        az container create --name quakes-load-testb${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$QUAKES_IP:3012/latest
        az container create --name weather-load-testb${i} -l eastus --image chzbrgr71/loadtest --resource-group aci -o tsv --cpu 1 --memory 1 --environment-variables load_duration=-1 load_rate=2 load_url=$WEATHER_IP:3015/latest
    done

    for i in 1 2 3; do
        az container delete --yes --resource-group aci --name flights-load-testb${i}
        az container delete --yes --resource-group aci --name quakes-load-testb${i}
        az container delete --yes --resource-group aci --name weather-load-testb${i}
    done
    ```

* Change deployment image tag:
    
    ```
    kubectl set image deployment/quakes-api quakes-api=briaracr.azurecr.io/hackfest/quakes-api:v5

    kubectl set image deployment/quakes-api quakes-api=briaracr.azurecr.io/hackfest/quakes-api:v5.1
    kubectl set image deployment/flights-api flights-api=briaracr.azurecr.io/hackfest/flights-api:v5.12
    ```