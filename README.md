# OpenShit custom regstry

This is a "cookbook" for configuring a custom image registry on OpenShift that can be used instead of an integrated or external registry.

## Configuration

Prepare variable for OpenShift project that will contain the registry
```sh
export PROJECT=demo-registry
```

Login to OpenShift cluster and create project
```sh
oc new-project $PROJECT
```

Define registry username and password (this is just an example, create more complex password in a real life)
```sh
export REG_USER="reguser"
export PASSWORD="passw0rd"
```

Create secret that contains username and password
```sh
podman run --rm --entrypoint htpasswd registry:2.7.0 -Bbn $REG_USER $PASSWORD > htpasswd
oc create secret generic auth-secret --from-file=htpasswd --namespace $PROJECT
```

Determine available storage classes
```sh
oc get sc
```

Select file based storage class (example)
```sh
export STORAGE_CLASS=ocs-storagecluster-cephfs
```

Create persistent volume claim (the size in this exaple is 50GB, correct it if needed)
```sh
cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-pvc
  namespace: $PROJECT
spec:
  storageClassName: $STORAGE_CLASS
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 50Gi
EOF
```

Create deployment
```sh
cat << EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: $PROJECT
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
      - name: registry
        image: registry:2.7.0
        env:
        - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
          value: "/var/lib/registry"
        - name: REGISTRY_AUTH 
          value: "htpasswd"
        - name: REGISTRY_AUTH_HTPASSWD_REALM
          value: "Registry Realm"
        - name: REGISTRY_AUTH_HTPASSWD_PATH
          value: "/auth/htpasswd"
        volumeMounts:
        - name: registry-storage
          mountPath: "/var/lib/registry"
        - name: htpasswd-secret
          mountPath: "/auth"  
      volumes:
      - name: registry-storage
        persistentVolumeClaim:
          claimName: registry-pvc
      - name: htpasswd-secret
        secret: 
          secretName: auth-secret
EOF
```

Create service
```sh
cat << EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: $PROJECT
spec:
  selector:
    app: registry
  ports:
    - name: registry-svc
      port: 5000
      targetPort: 5000
      protocol: TCP
EOF
```

Create route
```sh
cat << EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: registry
  namespace: $PROJECT
spec:
  to:
    kind: Service
    name: registry
    weight: 100
  port:
    targetPort: registry-svc
  tls:
    termination: edge
  wildcardPolicy: None
EOF
```

Check the result route URL
```sh
oc describe route registry -n $PROJECT
```

Store route's hostname to variable for easier testing (the sample value is specific to my environment)
```sh
export REGISTRY=registry-demo-registry.apps.infra01-lb.fra02.acme.com
```

## Testing

Try to login
```sh
podman login --tls-verify=false -u $REG_USER -p $PASSWORD $REGISTRY
```

Pull a simple image from the public registry, tag it and push to our registry
```sh
podman pull quay.io/podman/hello:latest
podman tag quay.io/podman/hello:latest $REGISTRY/hello:latest
podman push $REGISTRY/hello:latest
```

Cleanup local podman registry
```sh
podman rmi $REGISTRY/hello:latest
podman rmi quay.io/podman/hello:latest
```

Pull image from our registry and verify it is successfully pulled
```sh
podman pull $REGISTRY/hello:latest
podman images | grep hello
```

List registry catalg
```sh
curl -k -u $REG_USER:$PASSWORD -X GET https://$REGISTRY/v2/_catalog
```

List tags of the specific image (in this case "hello")
```sh
curl -k -u $REG_USER:$PASSWORD -X GET https://$REGISTRY/v2/hello/tags/list
```