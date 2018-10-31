## Accessing Google Cloud

- Ask someone to add you to the [Google Cloud Account](https://console.cloud.google.com/iam-admin/iam?authuser=0&project=xray2poc)
- After accepting the invite, make sure you have access to [Kubernetes project](https://console.cloud.google.com/kubernetes/list?authuser=0&organizationId=916896841130&project=xray2poc&angularJsUrl=%2Fkubernetes%2Flist%3Fauthuser%3D1%26organizationId%3D916896841130%26project%3Dxray2poc)

## Setting up kubectl

Follow the [Configuring Cluster Access for kubectl](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl).

tl;dr is 

- Install the [Cloud SDK](https://cloud.google.com/sdk/install)

- Select the GC project `xray2poc` and login

	```
	gcloud components update
	gcloud auth login
	gcloud config set project xray2poc
	gcloud container clusters get-credentials xray-dev --zone europe-west3-a
	```
	
- [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and make sure that you can list the Kubernetes pods

	```bash
	kubectl get pods

	NAME                                                   READY     STATUS              RESTARTS   AGE
	artifactory-artifactory-776bd78dc8-rcrvc               1/1       Running             0          74d
	artifactory-artifactory-nginx-c844d4fd8-rgkmj          1/1       Running             2          74d
	```

## Setting up Docker
Follow [Authenticate to the Docker registry ](https://cloud.google.com/container-registry/docs/advanced-authentication)

tl;dr is	

```bash
gcloud auth configure-docker

# If you get the error ` Invalid choice: 'configure-docker', make sure to run
gcloud components update
```

You should be able to login without password

```bash
docker login eu.gcr.io
Authenticating with existing credentials...
Login Succeeded
```

Make sure that your `~/.docker/config.json` does have the following `auth` part. If not, add it manually

```
{
  "auths": {
    "eu.gcr.io": { },
    "https://eu.gcr.io": { },
    "https://gcr.io": {},
    "gcr.io": {}
  },
  "credHelpers": {
    "marketplace.gcr.io": "gcloud",
    "asia.gcr.io": "gcloud",
    "us.gcr.io": "gcloud",
    "staging-k8s.gcr.io": "gcloud",
    "eu.gcr.io": "gcloud",
    "gcr.io": "gcloud"
  }
}
```




## Cluster setup

 * `kubectl apply -f resources/helm-service-account.yaml`
 * `helm init --service-account helm`
 * `gcloud container node-pools create infra2 --zone europe-west3-a --cluster xray-dev --machine-type n1-standard-2 --enable-autorepair --node-labels dedicated=infra --enable-autoscaling --min-nodes 1 --max-nodes 20 --num-nodes 1 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw,compute-rw"`
 * Secret with service-account JSON credentials

## Apps
 * `helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator`
 * `helm upgrade -i prometheus stable/prometheus -f apps/prometheus.yaml`
 * `helm upgrade -i grafana stable/grafana -f apps/grafana.yaml`
 * `helm upgrade -i vpn ./charts/openvpn -f apps/vpn.yaml`
 * `helm upgrade -i jenkins ./charts/jenkins -f apps/jenkins.yaml`
 * `helm upgrade -i nginx stable/nginx-ingress -f apps/nginx.yaml`
 * `kubectl apply -f resources/jenkins.yaml`
 * `helm upgrade -i artifactory ./charts/artifactory -f apps/artifactory.yaml`

## Kafka topics

* `kubectl run -i --rm --tty kafka-client --image=confluentinc/cp-kafka:4.0.1-1 --restart=Never -- /usr/bin/kafka-topics --zookeeper re-zookeeper:2181 --topic events --create --partitions 5 --replication-factor 1`

## Benchmarks

 * `helm upgrade -i re-bench1 ./charts/re_bench --wait --timeout 1200 && helm upgrade -i re-bench1 ./charts/re_bench --set tags.consumer=true`

## Usable commands

 * `kubectl run -i --rm --tty re-client --image=eu.gcr.io/xray2poc/re:latest --image-pull-policy=Always --restart=Never --env=KAFKA=re-kafka:9092 --env=KAFKA_GROUP=re --env=SCYLLA=re-scylla -- lein repl`
 * `kubectl run -i --rm --tty kafka-client --image=confluentinc/cp-kafka:4.0.1-1 --image-pull-policy=Always --restart=Never -- bash`
 * kafka-topics --zookeeper re-zookeeper:2181  --list
 * kafka-topics --zookeeper re-zookeeper:2181  --list --topic events --describe
 * kafka-consumer-groups --zookeeper re-zookeeper:2181 --list
 * kafka-console-consumer --bootstrap-server re-kafka:9092 --topic -events


## VPN Access

To get VPN key you should have access to our `xray-dev` Google's cluster and set it as current context for kubectl. Either it up setup (see abouve) or ask someone who already has the access then run

```bash
./tools/vpn-key.sh <key-name> default vpn
```
This script creates <key-name>.ovpn file in the current directory.

## Internal Services
Once connected via VPN, you can access

- [Jenkins](http://jenkins/)
- [Grafana](http://grafana/)
- [Httpbin](http://httpbin/)


## Deploying a docker image to k8s with Helm

A quick tutorial how to install a service to k8s, in this example the `httpbin`.

Create the chart templates to [charts directory](charts/):

```bash
> cd charts
> helm create httpbin
```

Add configuration to [apps](apps), in file `httpbin.yaml`:

```yaml
nodeSelector:
  dedicated: infra

resources:
 limits:
   cpu: 100m
   memory: 128Mi
 requests:
   cpu: 100m
   memory: 128Mi
```

Modify the [values.yaml](charts/httpbin/values.yaml) as needed, here we changed
the docker image repository to
[kennethreitz/httpbin](https://hub.docker.com/r/kennethreitz/httpbin/),
`replicaCount` to `4` and `tag` to `latest`.

Deploy the new service:

```bash
> helm upgrade -i httpbin charts/httpbin -f apps/httpbin.yaml
```

Monitor the deployment with kubectl.

```bash
> kubectl get deployments |grep httpbin
```

When done, the service is available with the name of the service, depending on
the `resolv.conf` httpbin answers from
(http://[httpbin|httpbin.default|httpbin.default.svc.cluster.local]).



## Custom Grafana dashboards

Grafana doesn't allow the user to save any changes to the dashboard. How to save
them for everybody:

 * Import the dashboard JSON from Grafana
 * Create a new file to the [dashboards directory](resources/grafana/dashboards), name it to `whatever-dashboard.yaml` including:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: "whatever-dashboard-name-you-want"
  labels:
    grafana_dashboard: "true"
data:
  also-whatever-but-still-having-a-meaning.json: |
    # Here paste your JSON. REMEMBER RIGHT INDENTATION. Four spaces here.
```

 * Apply the dashboard changes with `kubectl apply -f resources/grafana/dashboards/`.




