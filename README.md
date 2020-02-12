# What is Kustomize

"kustomize lets you customize raw, template-free YAML files for multiple purposes, leaving the original YAML untouched and usable as is.

kustomize targets kubernetes; it understands and can patch kubernetes style API objects. It's like make, in that what it does is declared in a file, and it's like sed, in that it emits edited text."

View repo of kustomize:

https://github.com/kubernetes-sigs/kustomize

# Practice of the kubernetes doc

https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/

Running through all the examples to get a feel for kustomize

To view Resources found in a directory containing a kustomization file, run the following command:

    $ kubectl kustomize <kustomization_directory>

To apply those Resources, run kubectl apply with --kustomize or -k flag:

    $ kubectl apply -k <kustomization_directory>


### Key words 

- configMapGenerator
- secretGenerator
- cross cutting fields
- composing (combining)
- customizing
    - patchesStrategicMerge 
    - patchesJson6902
    - vars
- bases & overlays

# configMapGenerator

## Example 1

`$ mkdir example-1`

/example-1

Create a application.properties file

```
cat <<EOF >./example-1/application.properties
FOO=Bar
EOF
```

```
cat <<EOF >./example-1/kustomization.yaml
configMapGenerator:
- name: example-configmap-1
  files:
  - application.properties
EOF
```

Run:

$ kubectl kustomize example-1/

```
apiVersion: v1
data:
application.properties: |
    FOO=Bar
kind: ConfigMap
metadata:
name: example-configmap-1-8mbdf7882g
```

`$ kubectl kustomize example-1/ > example-1/output.yaml`



## Example 2

`$ mkdir example-2`

/example-2


```
cat <<EOF >./example-2/kustomization.yaml
configMapGenerator:
- name: example-configmap-2
  literals:
  - FOO=Bar
EOF
```

`$ kubectl kustomize example-2/ > example-2/output.yaml`


# secretGenerator


## Example 3

`$ mkdir example-3`

/example-3

```
cat <<EOF >./example-3/password.txt
username=admin
password=secret
EOF
```

```
cat <<EOF >./example-3/kustomization.yaml
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
EOF
```

`$ kubectl kustomize ./example-3 > example-3/output.yaml`


## Example 4

`$ mkdir example-4`

/example-4

```
cat <<EOF >./example-4/kustomization.yaml
secretGenerator:
- name: example-secret-2
  literals:
  - username=admin
  - password=secret
EOF
```

`$ kubectl kustomize ./example-4 > example-4/output.yaml`



# generatorOptions

## Example 5

`$ mkdir example-5`

/example-5

The part to note here:

- **disableNameSuffixHash: true**

```
cat <<EOF >./example-5/kustomization.yaml
configMapGenerator:
- name: example-configmap-3
  literals:
  - FOO=Bar
generatorOptions:
  disableNameSuffixHash: true
  labels:
    type: generated
  annotations:
    note: generated
EOF
```

`$ kubectl kustomize example-5/ > example-5/output.yaml`

## Example 6

`$ mkdir example-6`

/example-6

Examples of cross-cutting fields:

- namespace
- prefix or suffix
- labels
- annotations
- customizing




```
cat <<EOF >./example-6/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
```

```
cat <<EOF >./example-6/kustomization.yaml
namespace: my-namespace
namePrefix: dev-
nameSuffix: "-001"
commonLabels:
  app: bingo
commonAnnotations:
  oncallPager: 800-555-1212
resources:
- deployment.yaml
EOF
```

`$ kubectl kustomize example-6/ > example-6/output.yaml`

# Composing and Customizing Resources

## Example 7

### Composing

`$ mkdir example-7`

/example-7


```
cat <<EOF > ./example-7/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```



```
cat <<EOF > ./example-7/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF
```



```
`cat <<EOF >./example-7/kustomization.yaml`
resources:
- deployment.yaml
- service.yaml
EOF
```


`$ kubectl kustomize ./example-7 > ./example-7/output.yaml`

___

### Customizing

Patches - summary

apply different customizations by applying patches

Support for different patching mechanisms:
- patchesStrategicMerge
    - Is a list of file paths
    - The names inside the patches must match Resource names
    - For example, create one patch for increasing the deployment replica number and another patch for setting the memory limit.
- patchesJson6902
    - https://tools.ietf.org/html/rfc6902
    - To support modifying arbitrary fields in arbitrary Resources, Kustomize offers applying JSON patch

___

## Example 8

`$ mkdir example-8`

/example-8

### patchesStrategicMerge

Think of pathing as a config way of doing kubectl edit...

https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/strategic-merge-patch.md



```
cat <<EOF > ./example-8/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```


```
cat <<EOF > ./example-8/increase_replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
EOF
```


```
cat <<EOF > ./example-8/set_memory.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  template:
    spec:
      containers:
      - name: my-nginx
        resources:
        limits:
          memory: 512Mi
EOF
```


```
cat <<EOF >./example-8/kustomization.yaml
resources:
- deployment.yaml
patchesStrategicMerge:
- increase_replicas.yaml
- set_memory.yaml
EOF
```

`$ kubectl kustomize example-8/ > example-8/output.yaml`



## Example 9

`$ mkdir example-9`

/example-9

#### patchesJson6902

To find the correct Resource for a Json patch, the group, version, kind and name of that Resource need to be specified



```
cat <<EOF > ./example-9/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```



```
cat <<EOF > ./example-9/patch.yaml
- op: replace
  path: /spec/replicas
  value: 3
EOF
```



```
cat <<EOF >./example-9/kustomization.yaml
resources:
- deployment.yaml

patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: my-nginx
  path: patch.yaml
EOF
```

`$ kubectl kustomize example-9/ > example-9/output.yaml`

## Example 10

$ mkdir example-10

/example-10

In addition to patches, Kustomize also offers customizing container images or injecting field values from other objects into containers without creating patches.

```
cat <<EOF > ./example-10/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```

```
cat <<EOF >./example-10/kustomization.yaml
resources:
- deployment.yaml
images:
- name: nginx
  newName: my.image.registry/nginx
  newTag: 1.4.0
EOF
```

`$ kubectl kustomize example-10/ > example-10/output.yaml`



## Example 11

`$ mkdir example-11`

/example-11

a Pod may need to use configuration values from other objects
Kustomize can inject the Service name into containers through **vars**

```
cat <<EOF > ./example-11/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        command: ["start", "--host", "\$(MY_SERVICE_NAME)"]
EOF
```

```
cat <<EOF > ./example-11/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF
```

```
cat <<EOF >./example-11/kustomization.yaml
namePrefix: dev-
nameSuffix: "-001"

resources:
- deployment.yaml
- service.yaml

vars:
- name: MY_SERVICE_NAME
  objref:
    kind: Service
    name: my-nginx
    apiVersion: v1
EOF
```

`$ kubectl kustomize example-11/ > example-11/output.yaml`

## Bases and Overlays

## Example 12

`$ mkdir example-12`

/example-12

reate a directory to hold the base

`$ mkdir example-12/base`

/example-12/base

```
cat <<EOF > ./example-12/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
EOF
```

```
cat <<EOF > ./example-12/base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF
```


```
cat <<EOF > ./example-12/base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```


```
mkdir example-12/dev
cat <<EOF > example-12/dev/kustomization.yaml
bases:
- ../base
namePrefix: dev-
EOF

mkdir example-12/prod
cat <<EOF > example-12/prod/kustomization.yaml
bases:
- ../base
namePrefix: prod-
EOF
```

`$ kubectl kustomize example-12/ > example-12/output.yaml`

## apply/view/delete objects using Kustomize

Run one of the following commands to view the Deployment object

```
kubectl get -k ./
```
```
kubectl describe -k ./
```
```
kubectl delete -k ./
```