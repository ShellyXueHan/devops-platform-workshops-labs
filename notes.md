# My personal notes message

1. start with oc design/structure map

2. `oc export` replaced with `oc get --export`

3. use labels for all components

4. use `oc apply` for updating and creating templates if not exist, not `oc create`. then `oc process` to run the template

5. deployment vs deploymentConfig
    deployment -> k8s
    deploymentConfig -> OCP, has a Raplication Controller (for each deployment version)

6. `oc new-build` recognize a jenkinsfile/docker/s2i from the context, and decide a strategy and build the thing


-----------creating deployment templates----------------

<!-- delete all the normal components after generating the templates -->
oc get all,configmap -l app=grafana
oc delete all,configmap -l app=grafana
oc get all,configmap -l app=shellyxuehan-loki
oc delete all,configmap -l app=shellyxuehan-loki
oc get all,configmap -l app=shellyxuehan-prometheus
oc delete all,configmap -l app=shellyxuehan-prometheus

<!-- create oc templates -->
oc apply -f xxx.yaml

<!-- use the template -->
oc process prometheus-template \
    -p PROMETHEUS_SERVICE_NAME=shellyxuehan-prometheus \
    -p ROUTE_SUBDOMAIN=pathfinder.gov.bc.ca \
    -p ROUTE_ENV=dev \
    | oc apply -f -

oc process loki-template \
    -p LOKI_SERVICE_NAME=shellyxuehan-loki \
    -p ROUTE_SUBDOMAIN=pathfinder.gov.bc.ca \
    -p ROUTE_ENV=dev \
    | oc apply -f -

oc process grafana-template \
    -p GRAFANA_SERVICE_NAME=shellyxuehan-grafana \
    -p LOKI_SERVICE_NAME=shellyxuehan-loki \
    -p PROMETHEUS_SERVICE_NAME=shellyxuehan-prometheus \
    -p ROUTE_SUBDOMAIN=pathfinder.gov.bc.ca \
    -p ROUTE_ENV=dev \
    | oc apply -f -


<!-- clean up the namespace -->
oc delete all,configmaps,templates,pvc,secrets -l app=shellyxuehan-grafana
oc delete all,configmaps,templates,pvc,secrets -l app=shellyxuehan-loki
oc delete all,configmaps,templates,pvc,secrets -l app=shellyxuehan-prometheus

-----------------setup jenkins--------------------
Start the build:
oc new-build https://github.com/ShellyXueHan/devops-platform-workshops-labs#ShellyXueHan-201 --context-dir=openshift201


--------------patroni-------------
<!-- tag the image -->
oc tag patroni:v10-latest patroni:v10-dev \
-n s4g19x-shellyxuehan-openshift201-may2019-tools

<!-- deploy ha db, ignore unused params from env var -->
oc process -f openshift/deployment.yaml \
  --param-file=./dev.env --ignore-unknown-parameters=true \
  | oc apply -f - -n s4g19x-shellyxuehan-openshift201-may2019-dev

<!-- image pulling for patroni-->
1. the deployment creates a service account for patroni use
2. add image puller role to it from tools:
    oc policy add-role-to-user system:image-puller system:serviceaccount:s4g19x-shellyxuehan-openshift201-may2019-dev:patroni -n s4g19x-shellyxuehan-openshift201-may2019-tools


--------------DB backup-------------
<!-- running the templates -->
1. build the image
oc process -f openshift/backup-build.json | oc apply -f -
2. deploy with two volumes attached, for backing up and restoring db
oc process -f openshift/simple-deploy.json | oc apply -f -

<!-- to trigger build -->
oc start-build xxxx
#HINT: avoid unecessary commits and build from current git clone/checkout directory.
oc start-build xxxx "--from-dir=$(git rev-parse --show-toplevel)" --wait