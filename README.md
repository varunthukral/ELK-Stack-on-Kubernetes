
# DigitalOcean-k8-log monitoring system
* Deploying ELK stack alongside fluentd to a DigitalOcean Managed Kubernetes (DOKS) cluster.
* Set Up an Elasticsearch, Fluentd,Logstash and Kibana  Logging Stack on Managed Kubernetes cluster .

## Prerequisites
* The kubectl command-line interface installed on your local machine and configured to connect to your cluster. You can read more about installing and configuring kubectl in its official [documentation](https://kubernetes.io/docs/tasks/tools/).
     * We’ll be deploying a 3-Pod Elasticsearch cluster, as well as a single Kibana and logstash Pod. Every worker node will also run a Fluentd Pod. The cluster in this guide consists of 3  nodes and a managed control plane
           ![](/images/github.PNG)
           ![](/images/pods.PNG)
            ![](/images/overview_do.PNG)
* A Kubernetes 1.10+ cluster with role-based access control (RBAC) enabled.
* Account in Digital Ocean Cloud Enviornment.
## Steps
1. Creating and setting up a Kubernetes Cluster in Digital Ocean
2. Creating a Namespace
3. Creating the Elasticsearch StatefulSet
4. Creating the Kibana Deployment and Service
5. Deploy  Filebeat and Logstash
6. Creating the Fluentd DaemonSet

### Note
All Yaml files are in manifest folder.

##  Creating and setting up a Kubernetes Cluster in Digital Ocean
* Follow the [Getting Started with kubernetes](https://docs.digitalocean.com/products/kubernetes/quickstart/) to set up our cluster. set up min of 3 node cluster
* Download the kubeconfig file and keep the file in your .kube/config folder
* Run the following command to check if connection with the cluster is successful and that you have a minimum of three clusters

 ![](/images/nodes.PNG)
 
 ![](/images/node_do.PNG)
 
 ![](/images/K8s-DO.PNG)


##  Creating a Namespace
~~~
kubectl create ns elk
~~~
![](/images/namespace_command.PNG)

![](/images/ns_do.PNG)


## Creating the Elasticsearch StatefulSet

In this Project , we use 3 Elasticsearch Pods and to start, we’ll create a headless Kubernetes service called elasticsearch that will define a DNS domain for the 3 Pods


We define a Service called elasticsearch in the elk Namespace, and give it the app:elasticsearch label. We then set the .spec.selector to app: elasticsearch so that the Service selects Pods with the app: elasticsearch label. When we associate our Elasticsearch StatefulSet with this Service, the Service will return DNS A records that point to Elasticsearch Pods with the app: elasticsearch label
we define a StatefulSet called elk-cluster in the elk namespace. We then associate it with our previously created elasticsearch Service using the serviceName field. This ensures that each Pod in the StatefulSet will be accessible using the following below DNS address
~~~
elk-cluster-0.elasticsearch,elk-cluster-1.elasticsearch,elk-cluster-2.elasticsearch
~~~



Creating the StatefulSet
Elasticsearch requires stable storage to persist data across Pod rescheduling and restarts.We specify 3 replicas (Pods) and set the matchLabels selector to app: elasticseach
We name the containers elasticsearch and choose the docker.elastic.co/elasticsearch/elasticsearch:7.2.0 Docker image.
 
 We then open and name ports 9200 and 9300 for REST API and inter-node communication, respectively. We specify a volumeMount called data that will mount the PersistentVolume named data to the container at the path /usr/share/elasticsearch/data
 we set some environment variables in the container:
 cluster.name and node.name,discovery.seed_hosts,cluster.initial_master_nodes,ES_JAVA_OPTS
 
 
 Kubernetes will use this to create PersistentVolumes for the Pods. In the block above, we name it data (which is the name we refer to in the volumeMounts defined previously), and give it the same app: elasticsearch label as our StatefulSet.
We then specify its access mode as ReadWriteOnce, which means that it can only be mounted as read-write by a single node. We define the storage class as do-block-storage.

Finally, we specify that we’d like each PersistentVolume to be 50GiB in size. You should adjust this value depending on your cluster.


Run the elk_stateful.yaml file.

~~~
kubectl create -f elk_statefulset.yaml
~~~

 ![](/images/elk_statefulset.PNG)
 
  ![](/images/elk_stateful_do.PNG)
 

monitor the StatefulSet as it is rolled out using kubectl rollout status:

~~~
kubectl rollout status sts/elk-cluster --namespace=elk
~~~

Once all the Pods have been deployed, we  can check that your Elasticsearch cluster is functioning correctly by performing a request against the REST API.


To do so, first forward the local port 9200 to the port 9200 on one of the Elasticsearch nodes (elk-cluster-0) using kubectl port-forward:
~~~
kubectl port-forward elk-cluster-0 9200:9200 --namespace=elk
~~~
![](/images/portforward_elk.PNG)

![](/images/restapi_elk.PNG)

This indicates that our Elasticsearch cluster k8s-logs has successfully been created with 3 nodes: elk-cluster-0, elk-cluster-1, and elk-cluster-2

### Creating the Kibana Deployment and Service

Kibana on Kubernetes, we’ll create a Service called kibana, and a Deployment consisting of one Pod replica

![](/images/services_do.PNG)

In the Deployment spec, we define a Deployment called kibana and specify that we’d like 1 Pod replica.  We use the docker.elastic.co/kibana/kibana:7.2.0 image.
Next, we use the ELASTICSEARCH_URL environment variable to set the endpoint and port for the Elasticsearch cluster. Using Kubernetes DNS, this endpoint corresponds to its Service name elasticsearch.

we set Kibana’s container port to 5601, to which the kibana Service will forward requests

~~~
kubectl create -f elk_kibana.yaml
~~~

![](/images/createelk_kibana.PNG)

we can check the rollout by below command

~~~
kubectl rollout status deployment/kibana -n elk
~~~
![](/images/rollout_elk.PNG)

![](/images/Deployments_DO.PNG)


To access the Kibana interface, we’ll once again forward a local port to the Kubernetes node running Kibana.

~~~
kubectl port-forward kibana-84cf7f59c-skn9d 5601:5601 --namespace=elk
~~~
![](/images/kibana_portforward.PNG)


Now, hit  the web browser, with  the below following URL:
~~~
http://localhost:5601
~~~
![](/images/Kibana.PNG)



### Deploy Filebeat and Logstash

use filebeat as a log shipper for our containers, we need to create separate filebeat pod for each running k8s node by using DaemonSet.
filebeat is going to be deployed to our rbac enabled cluster, we should first create a dedicated ServiceAccount.filebeat is deployed on all three nodes

we are going to use container input plugin which collects container logs under the given path. Also, to send the events directly to the Logstash, we will use logstash output plugin

After creating  ServiceAccount and ConfigMap, we can provide them to our DaemonSet

Run the filebeat-ds.yaml 

![](/images/filebeat_ds.PNG)

After deploying filebeat daemonset , it will deploy one filebeat pod for each node in your cluster.we can check the logs for the filebeat pod.

![](/images/loggingfilebeat.PNG)

~~~
kubectl  logs filebeat-6vw4j -n elk
~~~

we will use a config-map logstash-config will use elasticsearch to send events to elasticsearch under pre-defined indexes. Since container logs are in json format, we can use the json filter plugin to decode them

After creating the ConfigMap, we can bind it to our single Logstash pod. create a deployment object and its corresponding service which will interact with filebeat instances.

![](/images/cm.png)


After deployment,we check the logstash pod.
~~~
kubectl logs -f logstash-85b7fdd746-vzvzf -n elk
~~~
![](/images/logstashpodlogs.PNG)

![](/images/elk_timestamp.PNG)

### Creating the Fluentd DaemonSet

we’ll set up Fluentd as a DaemonSet, which is a Kubernetes workload type that runs a copy of a given Pod on each Node in the Kubernetes cluster. Using this DaemonSet controller, we’ll roll out a Fluentd logging agent Pod on every node in our cluster

we create a Service Account called fluentd that the Fluentd Pods will use to access the Kubernetes API. We create it in the elk Namespace and we have given the label app: fluentd

![](/images/elk-sa.PNG)


we define a ClusterRole called fluentd to which we grant the get, list, and watch permissions on the pods and namespaces objects

![](/images/clusterrole.PNG)

we define a DaemonSet called fluentd in the elk Namespace and give it the app: fluentd label

~~~
kubectl create -f fluentd.yaml
~~~
![](/images/fluentds.PNG)

![](/images/ds_do.PNG)


To demonstrate a basic Kibana use case of exploring the latest logs for a given Pod, we’ll deploy a minimal counter Pod that prints sequential numbers to stdout.




This is a minimal Pod called counter that runs a while loop, printing numbers sequentially.

Deploy the counter Pod using kubectl

~~~
kubectl create -f counter.yaml
~~~
From the Discover page, in the search bar enter kubernetes.pod_name:counter. This filters the log data for Pods named counter

![](/images/Counter.PNG)

