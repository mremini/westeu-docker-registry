## Generate Cert and Secret

1. Create a registry folder with certs and auth subfolder

```
mkdir -p registry && cd "$_"
mkdir certs
mkdir auth
```

2. generate a certificate and upload the crt, private key and the CA to the certs folder
3. Create a login / password that we'll use to authenticate for the registry access 

```
docker run --rm --entrypoint htpasswd registry:2.6.2 -Bbn mremini xxxxx > auth/htpasswd
```

4. At the end of this step our registry folder should look like this

```
mremini@mre-az-westeu-spoke1-lnx1:~$ ls -R registry/
registry/:
auth  certs

registry/auth:
htpasswd

registry/certs:
Docker_Registery_Westeu.crt  Docker_Registery_Westeu_nopass.key  ca.ftntcloudpoc.net.crt

```


## Create kubernetes Secrets

```
kubectl create secret tls cert-secret --cert=/home/mremini/registry/certs/Docker_Registery_Westeu.crt --key=/home/mremini/registry/certs/Docker_Registery_Westeu_nopass.key

kubectl create secret generic reg-auth-secret --from-file=/home/mremini/registry/auth/htpasswd
```

## Create Persistent Volume and Persistent Volume claim for to store registry images
Use the command  **kubectl create -f repo-volume.yaml** . The yaml file is below

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: docker-reg-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /tmp/repository
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: docker-reg-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi

```

## Create the Registry deployment
1. Use the command  **kubectl apply -f docker-registry-deployment.yaml** . The yaml file is below

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-registry-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: docker-registry
  template:
    metadata:
      labels:
        app: docker-registry
    spec:
      containers:
      - name: registry
        image: registry:2.6.2
        volumeMounts:
        - name: repo-vol
          mountPath: "/var/lib/registry"
        - name: certs-vol
          mountPath: "/certs"
          readOnly: true
        - name: auth-vol
          mountPath: "/auth"
          readOnly: true
        env:
        - name: REGISTRY_AUTH
          value: "htpasswd"
        - name: REGISTRY_AUTH_HTPASSWD_REALM
          value: "Registry Realm"
        - name: REGISTRY_AUTH_HTPASSWD_PATH
          value: "/auth/htpasswd"
        - name: REGISTRY_HTTP_TLS_CERTIFICATE
          value: "/certs/tls.crt"
        - name: REGISTRY_HTTP_TLS_KEY
          value: "/certs/tls.key"
      volumes:
      - name: repo-vol
        persistentVolumeClaim:
          claimName: docker-reg-pvc
      - name: certs-vol
        secret:
          secretName: cert-secret
      - name: auth-vol
        secret:
          secretName: reg-auth-secret
```

2. expose the registry using a NodePort service.  Use the command **kubectl apply -f docker-registry-svc.yaml**. The yaml file is below
```
apiVersion: v1
kind: Service
metadata:
  name: docker-registry-svc
spec:
  type: NodePort
  selector:
    app: docker-registry
  ports:
  - port: 5000
    targetPort: 5000
    protocol: TCP
```

3. Use the command **kubectl describe svc docker-registry-svc** to verify the service