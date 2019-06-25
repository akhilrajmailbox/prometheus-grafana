# creating admin privileges for the kube-system namespace
```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'      
helm init --service-account tiller --upgrade
```

*prometheus and grafana in eks (macos will not work for array [0] use ubuntu helm) ::*

[link 1](https://eksworkshop.com/monitoring/cleanup/)
[link 2](https://docs.aws.amazon.com/eks/latest/userguide/prometheus.html)


# Raw Metrics
Make sure that your kubernetes cluster giving you the metrics, if not you have to use some other services in order to achieve it...
```
kubectl get --raw /metrics
```

# make sure that helm installed in your system:
```
helm ls
```

# Create namespace and prometheus with persistentVolume:
*Create the namespace*

```
kubectl create namespace prometheus
```

*Create prometheus with persistentVolume, by default it will use 2 GB and 8 GB for each services*
```
helm install stable/prometheus \
    --name prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"
```

# check all pods and services:
```
kubectl get all -n prometheus
```

# test your prometheus from your local system with port forward:
```
kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090
```

[check 1](http://127.0.0.1:8080/)
[check 2](http://127.0.0.1:8080/targets)



# Create namespace and grafana with persistentVolume:
*Create the namespace*
```
kubectl create namespace grafana
```

## without ssl configuration
*Configuring grafana without ssl configuration*

```
helm install stable/grafana \
    --name grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set adminPassword="MyPassAlwaysSecure" \
    --set datasources."datasources\.yaml".apiVersion=1 \
    --set datasources."datasources\.yaml".datasources[0].name=Prometheus \
    --set datasources."datasources\.yaml".datasources[0].type=prometheus \
    --set datasources."datasources\.yaml".datasources[0].url=http://prometheus-server.prometheus.svc.cluster.local \
    --set datasources."datasources\.yaml".datasources[0].access=proxy \
    --set datasources."datasources\.yaml".datasources[0].isDefault=true \
    --set service.type=LoadBalancer
```

## with ssl configuration
*Configuring grafana with ssl certificates*

```
helm install stable/grafana \
    --name grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set adminPassword="XBLEKSGraf" \
    --set datasources."datasources\.yaml".apiVersion=1 \
    --set datasources."datasources\.yaml".datasources[0].name=Prometheus \
    --set datasources."datasources\.yaml".datasources[0].type=prometheus \
    --set datasources."datasources\.yaml".datasources[0].url=http://prometheus-server.prometheus.svc.cluster.local \
    --set datasources."datasources\.yaml".datasources[0].access=proxy \
    --set datasources."datasources\.yaml".datasources[0].isDefault=true \
    --set service.type=LoadBalancer \
    --set service.port=443 \
    --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-backend-protocol"=http \
    --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-extra-security-groups"=sg-25135300356456469 \
    --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-ssl-cert"=arn:aws:acm:us-east-1:5414653165468143:certificate/09c53c254523-5423-5-14251-dcaf7a9e5b \
    --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-ssl-ports"=https
```

*edit the service manually in order to accept ssl ceertificates*


from >>
```
  ports:
  - name: service
```

to >>
```
  ports:
  - name: https
```


# check all pods and services
```
kubectl get all -n grafana
```

# to get the loadbalancer url
```
export ELB=$(kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "http://$ELB"
```

# to get the password for admin user
```
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```


```
Login into Grafana dashboard using credentials supplied during configuration
You will notice that ‘Install Grafana’ & ‘create your first data source’ are already completed. We will import community created dashboard for this tutorial
Click ‘+’ button on left panel and select ‘Import’
Enter 3131 dashboard id under Grafana.com Dashboard & click ‘Load’.
Leave the defaults, select ‘Prometheus’ as the endpoint under prometheus data sources drop down, click ‘Import’.
This will show monitoring dashboard for all cluster nodes
For creating dashboard to monitor all pods, repeat same process as above and enter 3146 for dashboard id
```


# Delete Prometheus and grafana

```
helm delete prometheus
helm del --purge prometheus
helm delete grafana
helm del --purge grafana
```