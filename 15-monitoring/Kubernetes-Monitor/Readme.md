# Kubernetes Monitoring Using Prometheus Stack

> The Stack inludes the following

- Promethues
- Grafana
- Alert Manager

### Installing the Prometheus stack using Helm charts

*** Ref : https://github.com/prometheus-community/helm-charts/ ***

#### Add the repo in your workstation

> Pre-requites :
- Kubernetes Cluster
- helm cli installed
- smtp server details if you need alert Manager configured


```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

```

#### Install the prometheus stack

> Since all the services are Cluster IP by default, modify the services to NodePort if you wanted to access the services using the cluster values file


```