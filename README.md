# Elasticsearch on Kubernetes

This repository contains explanations,instructions and code to create an EFK Stack on Kubernetes on a minikube cluster for
purposes of demo and testing that the manifests work.
## EFK
![My Image](EFK.jpg)
## Definition
 **EFK** is an abbreviation for **Elasticsearch**,**Fluentd** and **Kibana**.**Elasticsearch** is a a search engine used to sift
 through large volumes of logs of data. it stores and retrieve logs from Fluentd.
 **Fluentd** is an open source log collection that supports multiple
 data sources and output formats. **Kibana** is used to query Elasticsearch and build vizualization for events log.

  ## StorageClass
 **storageclass/storage-class.yaml:**
 This file creates a storage class that will be used
 by VolumeClaimTemplates. it defines the storage class
 gp2 since the cluster is an aws EKS, It speecify the
 availability zones based on where the nodes region
 and availability zones in order to prevent node affinity
 problems where by a PVC might be created into a different
 availability zone than the pod.
 ## Elasticsearch

  ### 1. ConfigMap 
  **elasticsearch/elasticsearch-config.yaml:**
  The configMap has the necessary settings to configure the elasticsearch nodes.the settings included in this file define the cluster name,the node name,the node roles,the seed hosts and initial master nodes.It also enables xpack security for basic authentication when attempting to reach the elasticsearch cluter.
  ### 2. ES Load balancer 
  **elasticsearch/elasticsearch_svc.yaml:**
  * Creates a service that will define a DNS domain for the ES pods.
    when associated with ES StatefulSet,the service will return DNS an A records that point to ES pods with app: elasticsearch label.
  * The service will select pods that have the label app:elasticsearch.
  * The service is set to type LoadBalancer in order to create an external IP Address.
  * Two Ports are created 9200 and 9300 to interact with the
   rest API and inter-node communication respectively.
 
  ### 2. ES Stateful Set 
  **elasticsearch/elasticsearch_statefulset.yaml:**
  * create a Stateful set in order to have high availability 
    Instruct k8s to create 3 pods in order to maintain high availabilty
  * k8s will maintain 3 pods at all time 
  * Tell k8s which image to use and where to find it
  * Specify that the container needs at least 0.1vCPU
  * Open ports for REST API nad Inter node
     communication respectiveley.
  * can burst up to 1vCPU when needed
  * Specify a volumeMount called data and set it's path
     at /usr/share/elasticsearch/data.
  * create a cluster named k8s-logs
  * node name is set to metadata.name
     meaning pods have domains of the form 
     es-cluster-[0,1,2].
  * Tell the JVM to use a Minimum and Maximum of 512MB Heap size
     and define 3 Init Containers
  * The fix-permissions container container runs a chown to change   the owner and the group of ES Data dirctory ES UID
  because k8s mounts data directory as root which makes it inacessible
  * The increase-vm-max-map container
    runs a command to increase
    OS System limits on mmap counts 
    so that there are no out of memory errors
  * The increase-fdu-limit container
    increases fd--ulimit
    and increases the maximum number of
    open file descriptors
  * Create a volumeClaimTemplate
    that is used by k8s to create a 
    Persistent volume for pods and uses the storage class
    created earlier.
  ## Generating user passwords
  * because I enabled using xpack, a password is needed in order to connect to the cluste.run the following command:
    ```kubectl exec -it $(kubectl get pods -n kube-logging |grep es-cluster-0 | sed -n 1p | awk '{print $1}') -n kube-logging -- bin/elasticsearch-setup-passwords auto -b ```
    * This command generates passwords for different users including the elastic search user..refer to the commands.txt file to find the elastic user password.. 
  * Next  I add the generated password to the k8s secret as follows:
  ```kubectl create secret generic elasticsearch-pw-elastic -n kube-logging --from-literal password=```
  * The password appended on the command above is the password generated in the previous step..check out the commands.txt file to see the password.
  * Now we can check the health status of the elasticsearch cluster
  * To do that start by getting the elasticsearch IP Address
   * To get the elasticsearch service run ```Kubectl -n kube-logging get svc/elasticsearch```
  * A service named elasticsearch will appear with an external IP Address.
  * To check the health of the elasticsearch cluster, copy the IP address obtained with the previous command and append **:9200/_cluster/health/?pretty** to see the status of the cluster
  or append **:9200/_cluster/health/?pretty** 
  * You will be prompted for a username and password to authenticate.
  * Use the following credentials ie the password generated earlier, username: elastic and password:
  * You should see the status of the cluster marked as green in the yaml output you get meaning everything is working.
  ## Kibana Service and Deployment
     ## 1. Kibana ConfigMap
     **kibana/kibana-config.yaml**
  * This sets up environment variables for the elasticsearch
    host so that kibana knows where to connect ie the hosts 
    plus the elasticsearch username and password.
    ## kibana service and deployment
    **kibana/kibana-service.yaml**
  * Create a kibana service of Loadbalancer type
  * this service allows to access
    the kiban UI over node IP ADDRESS
  * This service should be accessible on port 5601
  * The kibana deloyment has a replica of one 
  * The elasticsearch values configured earlier via configMap
     are specified here.
  * Note that the secret key generated earlier is passed
    to kibana to allow it to authenticate to the cluster
    the password generate earlier is used as the password
    for login..
  * use the following command to get check the kibana service
    ```Kubectl -n kube-logging get svc/kibana```
  * append :5601 to the ip address to and you should
    see the Kibana UI with a form for login
    username is **elastic** and password password genarated earlier
  
 

## 4. Fluentd Service Account, Role, RBAC and Daemon set
   * Create service account that Fluentd 
     uses to access the k8s API
   * Define a ClusterRole called fluentd to and grant it the get,     list, and watch permissions on the pods
   * Then create a ClusterRoleBinding called fluentd which binds the fluentd ClusterRole to the fluentd Service Account.
   * Then create a Deamonset meaning it will run on each pod 
   in the cluster.
   * Fluentd uses these Elasticsearch values to ship the collected logs.
   * Finally There's a test pod to create logs
   to see if this setup works.
   # Testing the Complete setup
   * To test this setup, go to the kibana uI
   * go to console management 
   * go to index patterns, then click on index patterns
   * create an index pattern using "logstash-*"
   * and select "@timestamp" in the timestamp option
   then create index now go to discover console
   and you will see logs generated by the test pod.
## 5. Production Considerations
  #### a. Security
  This setup does not use TLS for communication
  but uses basic authentication
  ideally  certificates from a certificate authority
  should be used to increase security.
  generated and used for communication to increase security.
  #### b. Best Practices
  * Creating multiple nodes like master, data and client and ingest nodes with dedicated functionalities is good for high availability and fault tolerance
  * archiving indices to low cost cloud storage such as AW S3 and restore when you need data from those indices.
  * Its important to clean up old indices. a k8s cron job 
  can be setup to perfom this task.
  * Having data replicated across in different availability zones can help in disaster recovery and also improve query performance.
  ### d. credentials for the running cluster
  **username:elastic**
  **password is the password genarated earlier**
  ### N.B:
  
  * It takes a few moments to load and now you are inside the
    kibana dashboard. it sometimes requires refreshing the page
    when the passowrd fails as the kibana pod might not yet have
    connected to the elastic cluster yet..so refresh and try gain.
  * in order to deploy a yam file run ``` Kubectl apply -f ``` followed by the name of the file from the directory it resides in.
 ### sidecar
 ![image](https://github.com/ajin009/EFK-K8S-Sidecar/assets/105640840/a7dad79c-0ebc-43b6-9027-196db393d753)
 




