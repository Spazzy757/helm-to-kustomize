# How to convert Helm Chart into Kustomize Base

## Step 1: Helm Fetch

Fetch the chart as helm template needs it locally in order to template out the yaml

```
helm fetch \
--untar \
--untardir charts \
stable/nginx-ingress
```


## Step 2: Helm Template

Template out the yaml into a file, this is the step where you add the values to the chart

```
helm template \
--name ingress-controller \
--output-dir base \
--values values.yaml \
charts/nginx-ingress
```

This should give you a folder with a whole bunch of Kubernetes resources:

```
ls -l base/nginx-ingress/templates/

-rw-r--r--  1 spazzy  staff   1.0K Sep 24 08:44 clusterrole.yaml
-rw-r--r--  1 spazzy  staff   510B Sep 24 08:44 clusterrolebinding.yaml
-rw-r--r--  1 spazzy  staff   2.5K Sep 24 08:44 controller-deployment.yaml
-rw-r--r--  1 spazzy  staff   690B Sep 24 08:44 controller-hpa.yaml
-rw-r--r--  1 spazzy  staff   1.4K Sep 24 08:44 controller-role.yaml
-rw-r--r--  1 spazzy  staff   500B Sep 24 08:44 controller-rolebinding.yaml
-rw-r--r--  1 spazzy  staff   602B Sep 24 08:44 controller-service.yaml
-rw-r--r--  1 spazzy  staff   273B Sep 24 08:44 controller-serviceaccount.yaml
-rw-r--r--  1 spazzy  staff   1.6K Sep 24 08:44 default-backend-deployment.yaml
-rw-r--r--  1 spazzy  staff   541B Sep 24 08:44 default-backend-service.yaml
-rw-r--r--  1 spazzy  staff   288B Sep 24 08:44 default-backend-serviceaccount.yaml
```

just to neaten things up lets move these up a dir and delete the template dir:

```
mv base/nginx-ingress/templates/* base/nginx-ingress && rm -rf base/nginx-ingress/templates
```

## Step 3: create the Kustomization config

One thing that is not very well known is that helm does not handle namespaces very well, when you define `--namespace` while running `helm install` tiller does all the namespace work at runtime, meaning that in order to be more decalartive you will need to create you namespace config manually:

```
cat <<EOF > base/nginx-ingress/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress
EOF
```

You will need to create a `kustomization.yaml` that lists all your resources, this is probably the most time consuming part as this 
will require you to actually go through the resources, you can also add some common labels etc.

```
 cat <<EOF > base/nginx-ingress/kustomization.yaml
 namespace: "ingress"
 commonLabels:
     roles: routing
 resources:
 - namespace.yaml
 - clusterrole.yaml
 - clusterrolebinding.yaml
 - controller-deployment.yaml
 - controller-hpa.yaml
 - controller-role.yaml
 - controller-rolebinding.yaml
 - controller-service.yaml
 - controller-serviceaccount.yaml
 - default-backend-deployment.yaml
 - default-backend-service.yaml
 - default-backend-serviceaccount.yaml
EOF
```


## Step 4: Apply your new base to cluster

as of kubectl 1.14 kustomize is intergrated therfore you can simply run:

```
kubectl apply -k base/nginx-ingress
```
