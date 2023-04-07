# Confluent Cloud Monitoring

Monitoring for Confluent Cloud using Prometheus-Grafana in Kubernetes

Prometheus is a time series database that scrapes metrics from data sources, including Confluent Cloud Metrics API.

Grafana is a platform that takes telemetry data from different sources and visualizes it in dashboards using graphs and charts.

To deploy `Prometheus & Grafana` in the AKS. 


Step 1.
	
Verify access to your Kubernetes cluster from the monitoring team.


Step 2. 

[Install the Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)

Then Install the Azure Kubernetes Service CLI by running the following command:

`sudo az aks install-cli`

Step 3.
Verify your access. 

Set the subscription.

In order to authenticate into the Azure PROD account, run the following:
`az account set --subscription 2a71f9dc-3c1b-48dd-bb6b-abcde123456`

In order to authenticate into the Azure TE account, run the following:
`az account set --subscription 2a71f9dc-3c1b-48dd-bb6b-abcde123456`

To add the TE Kubernetes cluster/context to your Kube Config, run the following:
`az aks get-credentials --resource-group <resource-group> --name <cluster-name>`

To add the PD Kubernetes cluster/context to your Kube Config, run the following:
`az aks get-credentials --resource-group <resource-group> --name <cluster-name>`


Step 4.	

Set the kubectl login flow for Azure Kubernetes Service

`export KUBECONFIG=/path/to/kubeconfig`

`kubelogin convert-kubeconfig -l interactive`

Set the grafana namespace as the default namespace

`kubectl config set-context --current --namespace=grafana`

Step 5.	

Retrieve a Global [Confluent Cloud API key](https://docs.confluent.io/cloud/current/access-management/authenticate/api-keys/api-keys.html#cloud-cloud-api-keys) 


Step 6.
Create or update the prometheus-config config map using in `prometheus/configmap.yml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |-
    global:
      scrape_interval:     60s
      evaluation_interval: 60s
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager:9093"
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
        - targets: ['localhost:9090']
      
      - job_name: Confluent Cloud
        scrape_interval: 1m
        scrape_timeout: 1m
        honor_timestamps: true
        static_configs:
          - targets:
            - api.telemetry.confluent.cloud
        scheme: https
        basic_auth:
          username: '<api-key>'
          password: '<api-password>'
        metrics_path: /v2/metrics/cloud/export
        params:
          "resource.kafka.id":
            - '<resource-id-test-cluster>’
            - '<resource-id-stage-cluster>’
          "resource.ksql.id":
            - '<resource-id>’
          "resource.schema_registry.id":
            - '<resource-id>’


```

Step 7.
Create or Update the prometheus `prometheus/deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  labels:
    app: prometheus
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus
        args:
          - '--storage.tsdb.retention=6h'
          - '--storage.tsdb.path=/prometheus'
          - '--config.file=/etc/prometheus/prometheus.yml'
        ports:
        - name: web
          containerPort: 9090
        volumeMounts:
        - name: prometheus-config-volume
          mountPath: /etc/prometheus
      restartPolicy: Always
      volumes:
      - name: prometheus-config-volume
        configMap:
            defaultMode: 420
            name: prometheus-config
```

Step 8. Apply Kubernetes resources

`kubectl apply -f prometheus/`

Applies the Kubernetes yaml files in the prometheus directory.


Step 9. 
Create or Update three Grafana configmaps the in `grafana/datasource-configmap.yaml` directory.

`datasource-configmap.yml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
data:
  prometheus.yml: |-
    {
        "apiVersion": 1,
        "datasources": [
            {
              "access":"proxy",
              "editable": true,
              "name": "Prometheus",
              "orgId": 1,
              "type": "prometheus",
              "url": "http://<Promtheus-Pod-IP>:9090",
              "version": 1
            }
        ]
    }
```


The `grafana-configmap.yml` specifies the `grafana.ini` configuration settings file used by the Grafana deployment.

The `dasboard-configmap.yml` specifies the `ccloud.json` dashboard template used for the Confluent Cloud dashboard.

Step 10.
Create or Update Grafana Deployment file in `grafana/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      name: grafana
      labels:
        app: grafana
    spec:
      containers:
        - env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: password
            - name: GF_SECURITY_ADMIN_USER
              value: admin
            - name: GF_USERS_ALLOW_SIGN_UP
              value: "false"
          image: grafana/grafana:8.1.3
          name: grafana
          ports:
          - name: grafana
            containerPort: 3000
          volumeMounts:
            - mountPath: /etc/grafana/provisioning/datasources
              name: grafana-datasources
              readOnly: false
            - mountPath: /etc/grafana
              name: grafana-config
            - mountPath: /etc/grafana/provisioning
              name: grafana-dashboard
      volumes:
        - name: grafana-datasources
          configMap:
            defaultMode: 420
            name: grafana-datasources
        - name: grafana-config
          configMap:
            name: grafana-config
        - name: grafana-dashboard
          configMap:
            name: grafana-dashboard
```

Please note that the deployment file must mount the data inside the configmaps to the Kubernetes pod.


`kubectl apply -f grafana/`

Applies the Kubernetes yaml files in the grafana directory.

Step 11. Get the latest cluster IP of the pods

`kubectl describe pods`

Scroll up within the command's output to find the IP for both Prometheus and Grafana.

Make sure that the ClusterIP address of Prometheus is specfied in the `grafana/configmap.yml` file.

```yaml

data:
  prometheus.yml: |-
    {
        "apiVersion": 1,
        "datasources": [
            {
              "access":"proxy",
              "editable": true,
              "name": "Prometheus",
              "orgId": 1,
              "type": "prometheus",
              "url": "http://10.226.97.69:9090",
              "version": 1
            }
        ]
    }
```

Go into the TE VDI in Remote Connection and visit Prometheus and Grafana at the IPs stated in the pod.

Prometheus --> http://10.226.97.69:9090
Grafana --> http://10.226.96.188:3000

If these IPs are not loading, run `kubectl describe pods` to get the latest IP addresses.