##Generate Cert and Secret

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


##Create a kubernetes Secret using the cert

```
kubectl create secret tls cert-secret --cert=/home/mremini/registry/certs/Docker_Registery_Westeu.crt --key=/home/mremini/registry/certs/Docker_Registery_Westeu_nopass.key

```