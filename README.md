# Openshift GitOps Demo

## Prereqs

Download the `argocd` cli: https://argoproj.github.io/argo-cd/cli_installation/

## Installation

Install Openshift GitOps from Operator Hub

Apply the following to allow Argo to manage resources in other projects, per [Red Hat solution](https://access.redhat.com/solutions/6012601):

```
oc apply -f ./argocd-workaround993.yaml
```

## Access the GitOps Server

Run the following to get the Web URL:

```
ARGO_URL=$(oc get route openshift-gitops-server --template='{{.spec.host}}' -n openshift-gitops)
```

Retrieve the `admin` password from the `openshift-gitops-cluster` secret:

```
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```

### Login from the CLI

```
argocd login $ARGO_URL

argocd cluster list
```


### Create the `Application` resource in the `openshift-gitops` namespace

Create an [Application resource](https://github.com/redhat-developer-demos/openshift-gitops-examples/raw/main/components/applications/bgd-app.yaml) that defines the source and destination for your application.

```
oc apply -f https://github.com/redhat-developer-demos/openshift-gitops-examples/raw/main/components/applications/bgd-app.yaml
```

Once created, Argo will automatically start syncing the application to the cluster.

You will see resources created in the `bgd` namespace:

```
oc get pods,svc,route -n bgd
```

You can verify the app is deployed with the following URL:

```
oc get route bgd --template='{{.spec.host}}' -n bgd
```

### Make some manual changes

Add a manual change to the deployment with a patch command:

```
oc -n bgd patch deploy/bgd --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/env/0/value", "value":"green"}]'

oc rollout status deploy/bgd -n bgd
```

This will change the color of the square in the application. The `Deployment` resource has now drifted from the configuration. Argo will do nothing because the `syncPolicy.selfHeal` is `false`.

```yaml
[...]
kind: Application
spec:
  [...]
  syncPolicy:
    automated:
      prune: true
      selfHeal: false  # <--- argo doesn't fix configuration drift
```

We can configure self healing to be true, and Argo will sync the application when it has an OutOfSync state:

```
oc patch application/bgd-app -n openshift-gitops --type=merge -p='{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
```

## App of Apps Pattern

https://argoproj.github.io/argo-cd/operator-manual/cluster-bootstrapping/#app-of-apps-pattern

## Resources

Argo docs: https://argoproj.github.io/argo-cd/