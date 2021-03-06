# Application Containerization : Made Easy

## Local Deployment
#### For Local Deployment the following prerequisites must be met
```
For windows Users: Visual Studio Code with codewind+git extensions and docker installed
For Linux CLI: Docker CE and appsody should be installed 
```
#### for reference: 
```
 1) https://appsody.dev/docs/getting-started/installation
 2) https://phoenixnap.com/kb/how-to-install-docker-centos-7
 3) https://code.visualstudio.com/download
 4) https://docs.docker.com/docker-for-windows/
```
### Create a new appsody application

```
mkdir appsody-app
cd appsody-app
appsody init nodejs-express
```
### Testing application Locally
```
appsody run
```
#### Add logging to the application
```
npm install morgan
//add below contents in app.js to enable logging
const morgan = require('morgan')
app.use(morgan('combined'))
```

#### Default Endpoints

```
Health endpoint: http://localhost:3000/health
Liveness endpoint: http://localhost:3000/live
Readiness endpoint: http://localhost:3000/ready
Metrics endpoint: http://localhost:3000/metrics
Dashboard endpoint: http://localhost:3000/appmetrics-dash (development only)
```
#### Local application edits and live updates
```
app.get('/foo', (req, res) => {
  res.json({msg: 'bar'});
});
```
## Cluster Deployment

### We are going to use OpenShift cluter 4.2 and 3.11 for this demonstration

### Login to Openshift 4.2 local registry with user credentials

```
export INTERNAL_IMAGE_REGISTRY=image-registry.openshift-image-registry.svc:5000
export IMAGE_REGISTRY=default-route-openshift-image-registry.apps.ocp.tmrnd.com.my
docker login -u $(oc whoami) -p $(oc whoami -t) $IMAGE_REGISTRY
```

### Login to Openshift 3.11 local registry with user credentials

```
export INTERNAL_IMAGE_REGISTRY=docker-registry.default.svc:5000
export IMAGE_REGISTRY=docker-registry-default.apps.cp.tmrnd.com.my
docker login -u $(oc whoami) -p $(oc whoami -t) $IMAGE_REGISTRY
```

### If using another openshift cluster and get an error about https/ssl., then add openshift URL to the Docker as insecure-registry
```
 vi /etc/docker/daemon.json
{
"insecure-registries" : ["docker-registry-default.apps.192.168.80.130.nip.io","docker-registry.default.svc:5000"]
}
systemctl daemon-reload
systemctl restart docker
```

## Create Application

### Create a new appsody application or initialize and existing

```
mkdir appsody-app
cd appsody-app
appsody init incubator/python-flask
```
### Initialize existing application with appsody
```
git clone https://github.com/mdn/express-locallibrary-tutorial.git
cd express-locallibrary-tutorial
appsody init nodejs-express none
```
## Setup Application manifest

### Create a namespace to deploy the application for personal dev, for example my-project

```
oc new-project my-project
export NAMESPACE=$(oc project -q)
```
### Generate an appsody deploy yaml and rename it to app-deploy-remote.yaml

```
appsody deploy --generate-only -n $NAMESPACE
```
## Deploy the Application

### Build and push the application image to the remote registry

### If not using knative

```
export APP_NAME=$(yq r app-deploy.yaml metadata.name)
export IMAGE_NAME=${APP_NAME}
export IMAGE_TAG=${RANDOM}
appsody deploy \
--tag "${NAMESPACE}/${IMAGE_NAME}:${IMAGE_TAG}" \
--push-url "${IMAGE_REGISTRY}" \
--pull-url "${INTERNAL_IMAGE_REGISTRY}" \
-n ${NAMESPACE}
```

### If using knative

```
export APP_NAME=$(yq r app-deploy.yaml metadata.name)
export IMAGE_NAME=${APP_NAME}
export IMAGE_TAG=${RANDOM}
appsody deploy \
--tag "${NAMESPACE}/${IMAGE_NAME}:${IMAGE_TAG}" \
--push-url "${IMAGE_REGISTRY}" \
--pull-url "${INTERNAL_IMAGE_REGISTRY}" \
-n ${NAMESPACE} --knative
```

### Open with a browser or use curl

```
export APP_URL=http://$(oc get route ${APP_NAME} -n ${NAMESPACE} -o jsonpath="{.spec.host}")
export APP_URL=$(oc get ksvc ${APP_NAME} -n ${NAMESPACE} -o jsonpath="{.status.url}")
```

```
open ${APP_URL}
curl ${APP_URL}
```

## Automate appsody deploy

### Create a .env environment variable and a script deploy.sh. Use ./deploy.sh everytime you want to deploy to the
```
cat <<EOF >.env
INTERNAL_IMAGE_REGISTRY=image-registry.openshift-image-registry.svc:5000
IMAGE_REGISTRY=default-route-openshift-image-registry.apps.ocp.tmrnd.com.my
NAMESPACE=${NAMESPACE}
APP_NAME=$(yq r app-deploy.yaml metadata.name)
IMAGE_NAME=\${APP_NAME}
APP_KNATIVE=true
EOF
```
### Generate deploy.sh

```
cat <<EOF >deploy.sh
#!/bin/bash
source .env
IMAGE_TAG=\${RANDOM}

if ! oc get project \${NAMESPACE}; then
  echo project \${NAMESPACE} not found, creating new project \${NAMESPACE}
  oc new-project \${NAMESPACE}
fi

if [ "\$APP_KNATIVE" = "true" ]; then
  echo Deploying Serverless Service
  APP_KNATIVE_FLAG="--knative"
fi

appsody deploy \
  --tag \${NAMESPACE}/\${IMAGE_NAME}:\${IMAGE_TAG} \
  --push-url \${IMAGE_REGISTRY} \
  --pull-url \${INTERNAL_IMAGE_REGISTRY} \
  -n \${NAMESPACE} \${APP_KNATIVE_FLAG}

if [ "\$APP_KNATIVE" = "true" ]; then
  echo Getting Serveless Application URL...
  APP_URL=\$(oc get ksvc \${APP_NAME} -n \${NAMESPACE} -o jsonpath="{.status.url}")
else
  echo Getting Application URL...
  APP_URL=http://\$(oc get route \${APP_NAME} -n \${NAMESPACE} -o jsonpath="{.spec.host}")
fi

echo App deployed: \${APP_URL}
EOF
chmod +x deploy.sh
```

### Cluster Role for Openshift users
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: user-read-only-access
rules:
- apiGroups: ["", "apps", "autoscaling", "extensions"]
  resources: ["*"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apiextensions.k8s.io"]
  resources: ["customresourcedefinitions"]
  verbs: ["*"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["rolebindings"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```
### Cluster Role Binding
```
apiVersion: authorization.openshift.io/v1
groupNames: null
kind: ClusterRoleBinding
metadata:
   name: user-readonly-access
roleRef:
  name: user-readonly-access
subjects:
- kind: User
  name: okdmigrate
userNames:
- okdmigrate
```
### Serverless setup
#### Apply Patch
```
oc patch ksvc express-locallibrary-tutorial --type merge -p '{"spec":{"template":{"metadata":{"annotations":{"autoscaling.knative.dev/maxScale": "5"}}}}}'
```
```
oc patch ksvc express-locallibrary-tutorial --type merge -p '{"spec":{"template":{"metadata":{"annotations":{"autoscaling.knative.dev/target": "5"}}}}}'

```
#### Or edit manually
```
oc get ksvc
oc edit ksvc express-locallibrary-tutorial

spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "5"
        autoscaling.knative.dev/target: "5"
```
#### Apply the load benchmarking
```
hey -c 100 -z 10s "${SVC_URL}/?sleep=3&upto=10000&memload=100"
```
### Git Key Commands
```
Git Commands:
git init
git remote set-url origin https://github.com/karimbzu/express-library
git add .
git status
git commit -m "first commit"  /  git commit -a
git pull (if you want to merge with existing branch)
git push origin master
```
### Remove Docker Images
```
docker rmi $(docker images -a -q) --force
```
## CI/CD Pipeline
#### In this demo we will demonstrate how to deploy applications using CI/CD Pipeline in Openshift 4.2
### Notes for Red Hat OpenShift Installations

#### Using buildah in your pipelines

If you plan to use `buildah` in your Pipelines, you will need to set an additional permission on any service account that will be used to run a pipeline by using the following command:

      ```
      oc adm policy add-scc-to-user privileged -z [service_account_name] -n [namespace]
      ```


#### Pushing to the OpenShift registry using webhooks

Let's assume you wish to create a webhook such that created PipelineRuns will use the provided service account `tekton-webhooks-extension`.

Run the following command first:

`oc adm policy add-role-to-user edit -z tekton-webhooks-extension`

You should specify the following registry location if your namespace is `kabanero`:

`image-registry.openshift-image-registry.svc:5000/kabanero` (for OpenShift 4.2x)

or

`docker-registry.default.svc:5000/kabanero` (for OpehShift 3.11)

If using a self-signed certificate for the internal RedHat Docker registry, you will need to use a `buildah` task that skips self-signed certificate verifications too, for example by using the Tekton catalog's `buildah` task and setting TLS_VERIFY to default to `false`


#### Defined SCC

This project has been tested with the following scc (from `oc get scc`):

```
NAME               PRIV      CAPS      SELINUX     RUNASUSER          FSGROUP     SUPGROUP    PRIORITY   READONLYROOTFS   VOLUMES
anyuid             false     []        MustRunAs   RunAsAny           RunAsAny    RunAsAny    10         false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
hostaccess         false     []        MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <none>     false            [configMap downwardAPI emptyDir hostPath persistentVolumeClaim projected secret]
hostmount-anyuid   false     []        MustRunAs   RunAsAny           RunAsAny    RunAsAny    <none>     false            [configMap downwardAPI emptyDir hostPath nfs persistentVolumeClaim projected secret]
hostnetwork        false     []        MustRunAs   MustRunAsRange     MustRunAs   MustRunAs   <none>     false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
node-exporter      false     []        RunAsAny    RunAsAny           RunAsAny    RunAsAny    <none>     false            [*]
nonroot            false     []        MustRunAs   MustRunAsNonRoot   RunAsAny    RunAsAny    <none>     false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
privileged         true      [*]       RunAsAny    RunAsAny           RunAsAny    RunAsAny    <none>     false            [*]
restricted         false     []        MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <none>     false            [configMap downwardAPI emptyDir persistentVolumeClaim projected secret]
```
