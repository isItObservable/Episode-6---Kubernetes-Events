# How to collect Kubernetes events

This repository is here to guide you through the GitHub tutorial that goes hand-in-hand with a video available on YouTube and a detailed blog post on my website. 
Together, these resources are designed to give you a complete understanding of the topic.


Here are the links to the related assets:
- YouTube Video: [The power of Kubernetes Events](https://www.youtube.com/watch?v=K-F-p-ekSsM)
- Blog Post: [How to collect Kubernetes events](https://isitobservable.io/observability/kubernetes/how-to-collect-kubernetes-events)


Feel free to explore the materials, star the repository, and follow along at your own pace.


## Kubernetes Events - Kspan and Event exporter
<p align="center"><img src="/image/kubernetes.png" width="40%" alt="Loki Logo" /></p>

This repository showcases the usage of the Kspan and Event Exporter with Dynatrace by using GKE with the HipsterShop


## Prerequisite
The following tools need to be installed on your machine :
- jq
- kubectl
- git
- gcloud (if you're using GKE)
- Helm
  
### 1. Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2. Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create isitobservable \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone  the GitHub repo
```
git clone https://github.com/isItObservable/Episode-6---Kubernetes-Events
cd Episode-6---Kubernetes-Events
```
### 4. Deploy Prometheus
#### HipsterShop
```
cd hipstershop
./setup.sh
```
#### Prometheus (as done in [Episode 1](https://github.com/isItObservable/Episode1---Kubernetes-Prometheus/) )
```
helm install prometheus stable/prometheus-operator
```
### 5. Deploy the Dynatrace K8s operator
##### 1. Start a Dynatrace trial
Go to the following URL to start a [Dynatrace trial](https://www.dynatrace.com/trial/)

##### 2. Install the Dynatrace K8s operator
Follow the steps described on the [Dynatrace documentation](https://www.dynatrace.com/support/help/setup-and-configuration/setup-on-container-platforms/kubernetes/monitor-kubernetes-environments/#anchor_classic)

### 6. Deploy the Event Exporter
#### 1. Install Event exporter
```
kubectl apply -f Event-exporter/deploy.yaml
```
The event exporter will expose 2 Prometheus Counter:
* kube_event_count
* kube_event_unique_events_total

The deployment file has the dynatrace annotation that allow dynatrace to scrap automatically the metrics exposed by our exporter
```
metrics.dynatrace.com/port: '9102'
metrics.dynatrace.com/scrape: 'true'
metrics.dynatrace.com/path: '/metrics'
```
#### 2. Explore the Prometheus metrics in Dynatrace
In Dynatrace, click on metrics and search for kube_event_unique_events
Once we see the metric in Dynatrace, we can now use the metric explorer to build a graph.
<p align="center"><img src="/image/dt_metric.png" width="60%" alt="dynatrace metrics" /></p>

#### 3. Let's build a Graph
In Dynatrace, click on the Explore Data on the left menu
Search for the metric kube_event_unique_events_total_count
Use :
    * the aggregator : Count
    * split by : involved_object_kind,type,reason
    * select the visualization : Pie
<p align="center"><img src="/image/dt_explore_data.png" width="60%" alt="dt metric explorer" /></p>


### 6. Deploy Kspan

#### Update the OpenTelemetry collector deployment file
Before creating the secret, you'll need to generate the api token in Dynatrace.
Follow the instructions described in the [Dynatrace documentation](https://www.dynatrace.com/support/help/shortlink/api-authentication#generate-a-token)
Make sure that the scope Ingest OpenTelemetry traces is enabled.
<p align="center"><img src="/image/dt_api.png" width="60%" alt="dt api scope" /></p>

We need to update the OpenTelemetry collector deployment file by referring to our Dynatrace tenant
```
EXPORT DT_API_TOKEN=<YOUR DT TOKEN>
EXPORT DT_API_URL="https://{your-environment-id}.live.dynatrace.com"
sed -i "s,TENANTURL_TOREPLACE,$DT_API_URL," kspan/otel-collector-deployment.yaml
sed -i "s,DT_API_TOKEN_TO_REPLACE,$DT_API_TOKEN," kspan/otel-collector-deployment.yaml
```
#### Configure Dynatrace
Before deploying kspan, let's configure Dynatrace to store the various spans and resource attributes sent by kspan.
This configuration is important to be able to store the Kubernetes information in the trace
Here are the span attributes that need to be created in Dynatrace ( Settings/service-side monitoring/ span attributes )
* eventID
* kind
* message
* name
* namespace
* reason
* type
<p align="center"><img src="/image/dt_span_attribute.png" width="60%" alt="dt api scope" /></p>

Here are the resource attributes that need to be created ( Settings/service-side monitoring/Resource attributes )
* service.instance.id
* service.name
<p align="center"><img src="/image/dt_ressource_attribute.png" width="60%" alt="dt ressource attribute" /></p>

#### Deploy kspan
To deploy kspan, we need to create a service account having a clusterRole to be able to get, list, and watch events from the cluster.
Therefore, we need to deploy KSPAN in the following order:
```
kubectl apply -f kspan/rbac.yaml
kubectl apply -f kspan/otel-collector-deployment.yaml
kubectl apply -f kspan/kspan_deployment.yaml
```
#### Visualize the traces in Dynatrace

To visualize the traces in Dynatrace, let's update our k8s workload a bit by deploying another testing application: the sockshop
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
To visualize the traces in Dynatrace, click on the menu (Application & Microservices / Distributes traces)
Filter on "Ingested traces"
<p align="center"><img src="image/dt_distributed traces.PNG" width="60%" alt="dt distributed traces" /></p>

