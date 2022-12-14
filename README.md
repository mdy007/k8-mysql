Initially create a namepsace called "siteassignment" and create the blow workloads,

1. Storage class - We are creating a storage class to allocate persistant volumes
> kubectl create -f storage-class.yaml
2. Configmap - Which specifies a few variables to set in the MySQL
configuration files. These different configurations are selected by the
pods themselves, but they give us a handy way to manage potential
configuration variables
> kubectl create -f mysql-configmap.yaml
3. Services - MySQL pods can talk to one another and our
App pods can talk to MySQL
> kubectl create -f mysql-services.yaml
4. Statefulset - Create 1 master MYSQL pod and 2 worker MYSQL pod ( with xtrabackup container - MySQL’s replication handles
master-to-slave replication but xtrabackup handles slave-to-master backward replication )
> kubectl create -f mysql-statefulset.yaml
5. NFS - Images will be mounted in NFS service with replication for HA
> kubectl create -f nfs.yaml
6. APP - We will create a HA application with HA available MYSQL cluster 
> kubectl create -f app.yaml

Resilience Testing
> $ kubectl scale --replicas=3 deployment/wordpress.

We’ll again see that data is preserved across all three instances. To test the MySQL StatefulSet, we can scale down the number of replicas using the following
> $ kubectl scale statefulsets mysql --replicas=1

We’ll see a loss of both slaves in this instance and, in the event of a loss of the master in this moment, the data it has will be preserved on the GCE
Persistent Disk. However, we’ll have to manually recover the data from the disk. If a master node goes down, a new master will be spun up and via xtrabackup, it will repopulate with the data from a slave. Therefore, I don’t recommend ever running with a replication factor of less than three when running production databases.

HA - Enable HPA based on CPU > 30% for the site pod set.
> kubectl autoscale deployment wordpress --cpu-percent=30 --min=1 --max=10

7. node-exporter - Deploy node exporter on all the Kubernetes nodes as a daemonset. Daemonset makes sure one instance of node-exporter is running in all the nodes. It exposes all the node metrics on port 9100 on the /metrics endpoint
> kubectl create -f daemonset.yaml
> kubectl create -f service.yaml

8. mysql-exporter - installing mysql exporter using helm charts with custom conf as values.yaml
> helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm repo update
  helm install --name <release name> -f values.yaml
  
by using ingress endpoint we can able to reach the metrics or else we can download the kubeconfig of our stack in local and just port forward to our laptop 
> kubectl port-forward --namespace=siteassignment service/ingress 9100:9100 & \
  kubectl port-forward --namespace=siteassignment service/ingress 9104:9104
  
9. Pod restart - you can deploy the kube-state-metrics container that publishes the restart metric for pods: https://github.com/kubernetes/kube-state-metrics 
more on this - https://github.com/kubernetes/kube-state-metrics/blob/master/docs/pod-metrics.md
