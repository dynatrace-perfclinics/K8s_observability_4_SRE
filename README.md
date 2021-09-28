# Peformance Clinic : Kubernetes observavility for SREs with Dynatrace
This repository contains the files utilized during the demo of the Performance clinic on K8s observability for SREs with Dynatrace

This repository showcase the usage of the Prometheus OpenMetrics Ingest by using GKE with :
* the HipsterShop
* the dynatrace sockshop
* Litmus Chaos
* OpenTelemetry Collector
* Istio
* Kspan and Event exporter


## Prerequisite 
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create onlineboutique \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone Github repo
```
git clone https://github.com/dynatrace-perfclinics/K8s_observability_4_SRE
cd K8s_observability_4_SRE
```
### Deploy the sample Application

#### 0. Istio
0. Create the various namespaces
For the hipsterShop :
```
   kubectl create namespace hipster-shop
   kubectl -n hipster-shop create rolebinding default-view --clusterrole=view --serviceaccount=hipster-shop:default
```

1. Download Istioctl
```
curl -L https://istio.io/downloadIstio | sh -
```
This command download the latest version of istio ( in our case iostio 1.10.2) compatible with our operating system.
2. Add istioctl to you PATH
```
cd istio-1.10.3
```
this directory contains samples with addons . We will refer to it later.
```
export PATH=$PWD/bin:$PATH
```
#### 1. Install Istio
##### 1. Configure Istio To generate OpenTelemetry Traces & Metrics
###### a. Istio Traces
To enable Istio and take advantage of the tracing capabilities of Istio, you need to install istio with the following settings
 ```
istioctl install --set meshConfig.defaultConfig.tracing.zipkin.address=otlp-collector.default.svc.cluster.local:9411 --set meshConfig.enableTracing=true --set profile=demo -y
 ```
After this command Istio will configure the Envoy Proxy to generate traces and export it to Otlp-collector service. ( it will be installed on the next steps)

###### b. Update the OpenTelemetry collector deployment file
Because Isio generates traces in a Zipkin traces, we need to convert the generated traces in an OpenTelemetry Format.
OpenTelemetry colletor would be the component that will help us converting our traces and export it to Dynatrace Open Trace Ingest.

Before deploying our OpenTelemetry collector, we need to generate the api token in dynatrace.
Follow the instruction described in [dynatrace's documentation](https://www.dynatrace.com/support/help/shortlink/api-authentication#generate-a-token)
Make sure that the scope Ingest OpenTelemetry traces and metrics v2 is enabled.
<p align="center"><img src="/image/dt_api.PNG" width="60%" alt="dt api scope" /></p>

We need to update the opentelemetry collector deployment file by referring to our dynatrace tenant
```
EXPORT DT_API_TOKEN=<YOUR DT TOKEN>
EXPORT DT_API_URL="https://{your-environment-id}.live.dynatrace.com"
sed -i "s,TENANTURL_TOREPLACE,$DT_API_URL," istio/otel-collector-deployment.yaml
sed -i "s,DT_API_TOKEN_TO_REPLACE,$DT_API_TOKEN," istio/otel-collector-deployment.yaml
```
Before deploying the OpenTelemetry Collector , you need to get the dynatrace cluster id registered in Dynatrace.
To extact , you will need to go to Infrastructure/Kubernetes, select the cluster you want to monitor.
select the Hexadecimal value of the cluster id in the URL :
<p align="center"><img src="/image/cluster_config_id.PNG" width="60%" alt="dt api scope" /></p>
The hexadecimal value needs to be converted in to decimal.
once you have the decimal value you can run the following command :

```
EXPORT DT_CLUSTER_ID=<YOUR DT CLUSTER ID>
sed -i "s,DYNATRACE_CLUSTER_ID,$DT_CLUSTER_ID," istio/otel-collector-deployment.yaml
```
###### c. Deploy the OpenTelemetry Collector
We can now deploy the openTelemetry collector :
```
kubectl apply -f istio/otel-collector-deployment.yaml
```

Then we want to instruct istio to automatically inject the envoy Proxy to all the pods of our Hipster-shop application
so we will label the namesapce : hipster-shoo
```
kubectl label namespace hipster-shop istio-injection=enabled
```
### 2. Kubernetes events : Deploy Kspan and Event exporter
#### Kspan
Kspan is a solution build by weaveworks generating OpenTelementry traces based on K8s event.
To deploy kspan , we need to create a serice account having a clusterRole to be able to get, list, watch events from the cluster.
Therefore we need to deploy kspan in the following order:
```
kubectl apply -f kspan/rbac.yaml
kubectl apply -f kspan/kspan_deployment.yaml
```
#### Event exporter
```
kubectl apply -f Event-exporter/deploy.yaml
```
The event exporter will expose 2 Prometheus Counter:
* kube_event_count
* kube_event_unique_events_total

### 3.HipsterShop
```
cd hipstershop
./setup.sh
```

##### Update the ingressgateway to expose ports for sockshop
```
kubectl edit svc istio-ingressgateway -n istio-system
```
Add the following ports :
```
- name: web
  nodePort: 31770
  port: 8080
  protocol: TCP
  targetPort: 8182
```

#####  Expose the HipsterShop out of the cluster
```
kubectl apply -f istio/hipstershop_gateway.yaml
```
### 5. Deploy Sockshop Production
```
cd ../sockshop
kubectl create -f ./manifests/k8s-namespaces.yml

kubectl -n sockshop-production create rolebinding default-view --clusterrole=view --serviceaccount=sockshop-production:default


kubectl apply -f ./manifests/backend-services/user-db/sockshop-production/


kubectl apply -f ./manifests/backend-services/shipping-rabbitmq/sockshop-production/

kubectl apply -f ./manifests/backend-services/carts-db/
kubectl apply -f ./manifests/backend-services/catalogue-db/

kubectl apply -f ./manifests/backend-services/orders-db/

kubectl apply -f ./manifests/sockshop-app/sockshop-production/
```
#####  Expose the Sockshop out of the cluster
```
cd ..
kubectl apply -f istio/sockshop_gateway.yaml
```
### 8. Install Litmus Chaoas
```
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
kubectl create ns litmus
kubectl get pods -n litmus
kubectl apply -f /litmus_choas/scheduler.yaml -n litmus
```
Apply the experiments in the hipster-shop Namespace
```
kubectl apply -f /litmus_choas/experiment.yaml -n hipster-shop
```
Apply the Service account authorize to run the various experiments
```
kubectl apply -f /litmus_choas/rbac.yaml -n hipster-shop
```
Deploy the scheduled experiments :
```
kubectl apply -k /litmus_choas/schedule/ -n hipster-shop
```
### 5. Run a neoload test

#### uppdate the definition of your hipstershop url
TO get the ip adress of the istio gateway run the following command :
```
kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
Update the following file loadtest/hipstershop/team/servers/34#2E89#2E214#2E38.xml
update the hostname value by replacing with your Ip adress.

Download The latest version of [NeoLoad](https://www.neotys.com/support/download-neoload) 

Launch NeoLoad and opent the project : loadtest/hipstershop/hipstershop.nlp

Click on the Scenario Tab and click on run the predefined test.


