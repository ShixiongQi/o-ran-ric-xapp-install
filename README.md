## Reference
```
https://docs.o-ran-sc.org/projects/o-ran-sc-it-dep/en/latest/installation-guides.html#ric-platform
```

## Setting up Kubernetes Cluster
```
sudo -i && cd
git clone https://gerrit.o-ran-sc.org/r/it/dep

cd dep/tools/k8s/bin
./gen-cloud-init.sh

# ./k8s-1node-cloud-init.sh
```

## Onetime setup for Influxdb
```
kubectl get ns ricinfra

# If the namespace doesn’t exist, then create it using:
kubectl create ns ricinfra

helm install stable/nfs-server-provisioner --namespace ricinfra --name nfs-release-1
kubectl patch storageclass nfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
apt install nfs-common
```

## Setting up RIC Platform
```
cd && cd dep
git submodule update --init --recursive --remote

# 1. Modify the deployment recipe - "ricip" and "auxip" in "./RECIPE_EXAMPLE/PLATFORM/example_recipe.yaml."
# 2. Modify "ric-dep/helm/appmgr/values.yaml" and "ric-dep/helm/infrastructure/values.yaml", change the image repository of helm tiller to "fishead/gcr.io.kubernetes-helm.tiller:v2.12.3"
# Reference: https://hub.docker.com/layers/fishead/gcr.io.kubernetes-helm.tiller/v2.12.3/images/sha256-cab750b402d24dd7b24756858c31eae6a007cd0ee91ea802b3891e2e940d214d?context=explore
# 3. Modify "bin/deploy-ric-platform" -> "$ROOT_DIR/../ric-dep/bin/install -f $OVERRIDEYAML -c influxdb"

# Use dawn release by default
cd && cd dep/bin
./deploy-ric-platform ../RECIPE_EXAMPLE/PLATFORM/example_recipe_oran_dawn_release.yaml

# Checking Ingress Gateway Health
curl -v http://${INGRESS_IP}:32080/appmgr/ric/v1/health/ready
```

## Setting up xApp onboarding tools
```
# Create a local helm repository with a port other than 8080 on host
docker run --rm -u 0 -it -d -p 8090:8080 -e DEBUG=1 -e STORAGE=local -e STORAGE_LOCAL_ROOTDIR=/charts -v $(pwd)/charts:/charts chartmuseum/chartmuseum:latest

# Set CHART_REPO_URL env variable
export CHART_REPO_URL=http://0.0.0.0:8090

# Git clone appmgr
cd && git clone "https://gerrit.o-ran-sc.org/r/ric-plt/appmgr"

# Change dir to xapp_onboarder
cd appmgr/xapp_orchestrater/dev/xapp_onboarder

# If pip3 is not installed, install using the following command
apt install -y python3-pip

# In case dms_cli binary is already installed, it can be uninstalled using following command
pip3 uninstall xapp_onboarder

# Install xapp_onboarder using following command
pip3 install ./

# (OPTIONAL) List the helm charts from help repository
curl -X GET http://localhost:8090/api/charts | jq .
```

## Setting up Traffic Steering use case (TS, QP, AD)
```
cd && cd o-ran-ric-xapp-install
git submodule update --init --recursive --remote

# Onboard TS xApp
dms_cli onboard ric-app-ts/xapp-descriptor/config.json ric-app-ts/xapp-descriptor/schema.json

# Onboard AD xApp
dms_cli onboard ric-app-ad/xapp-descriptor/config.json ric-app-ts/xapp-descriptor/schema.json

# Onboard QP xApp
dms_cli onboard ric-app-qp/xapp-descriptor/config.json ric-app-ts/xapp-descriptor/schema.json

# Install xApps
dms_cli install trafficxapp 1.2.1 ricxapp
dms_cli install qp 0.0.4 ricxapp
dms_cli install ad 0.0.2 ricxapp
```

## Rebuiding Traffic Steering use case (TS, QP, AD)
```
# Remember to log in to docker first
docker login

# Rebuid TS xApp
docker build --network host --no-cache .
docker build -t shixiongqi/ric-app-ts:1.2.1 .
docker push shixiongqi/ric-app-ts:1.2.1

kubectl -n ricxapp rollout restart deployment ricxapp-trafficxapp

# Rebuid AD xApp
docker build --network host --no-cache .
docker build -t shixiongqi/ric-app-ad:0.0.2 .
docker push  shixiongqi/ric-app-ad:0.0.2

kubectl -n ricxapp rollout restart deployment ricxapp-ad

# Rebuid QP xApp
docker build --network host --no-cache .
docker build -t shixiongqi/ric-app-qp:0.0.4 .
docker push     shixiongqi/ric-app-qp:0.0.4

kubectl -n ricxapp rollout restart deployment ricxapp-qp
```
-----------
## Tips for InfluxDB
```
InfluxDB can be accessed via port 8086 on the following DNS name from within your cluster:

  http://ricplt-influxdb.ricplt:8086

You can connect to the remote instance with the influx CLI. To forward the API port to localhost:8086, run the following:

  kubectl port-forward --namespace ricplt $(kubectl get pods --namespace ricplt -l app=ricplt-influxdb -o jsonpath='{ .items[0].metadata.name }') 8086:8086

You can also connect to the influx CLI from inside the container. To open a shell session in the InfluxDB pod, run the following:

  kubectl exec -i -t --namespace ricplt $(kubectl get pods --namespace ricplt -l app=ricplt-influxdb -o jsonpath='{.items[0].metadata.name}') /bin/sh

To view the logs for the InfluxDB pod, run the following:

  kubectl logs -f --namespace ricplt $(kubectl get pods --namespace ricplt -l app=ricplt-influxdb -o jsonpath='{ .items[0].metadata.name }')
```
