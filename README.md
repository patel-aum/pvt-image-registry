# Checkout my Medium blog with all the detailed instructions:
```
https://medium.com/@patel-aum/building-your-own-secure-docker-registry-with-tls-in-kubernetes-f3d086b9e00b
```


### First lets generate the certificate with openSSL:

```
cat > cert.conf << EOF
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = my-registry

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = my-registry
DNS.2 = registry-service
DNS.3 = registry-service.default.svc.cluster.local
IP.1 = 192.168.49.2
IP.2 = 10.111.137.49
EOF

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 1024 -out ca.crt -subj "/CN=Registry CA"

openssl genrsa -out tls.key 4096
openssl req -new -key tls.key -out tls.csr -config cert.conf
openssl x509 -req -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt -days 365 -extensions v3_req -extfile cert.conf

# Kubernetes secret
kubectl create secret tls certs --cert=tls.crt --key=tls.key

# Update host machine Docker certs
sudo mkdir -p /etc/docker/certs.d/my-registry:31334
sudo cp ca.crt /etc/docker/certs.d/my-registry:31334/ca.crt
sudo cp tls.crt /etc/docker/certs.d/my-registry:31334/cert.crt
sudo cp tls.key /etc/docker/certs.d/my-registry:31334/key.pem

# Update minikube Docker certs
minikube ssh "sudo mkdir -p /etc/docker/certs.d/my-registry:31334"
minikube cp ca.crt /etc/docker/certs.d/my-registry:31334/ca.crt
minikube ssh "sudo update-ca-certificates"

sudo cp ca.crt /usr/local/share/ca-certificates/registry-ca.crt
sudo update-ca-certificates
sudo cp ca.crt /etc/docker/certs.d/my-registry:31334/ca.crt

sudo systemctl restart docker
ls -l /etc/docker/certs.d/my-registry:31334/
ls -l /usr/local/share/ca-certificates/

```

### create a encrypted password file

```
docker run --entrypoint htpasswd httpd:2 -Bbn aum pa$$ > auth/htpasswd
kubectl create secret generic auth --from-file=auth/htpasswd
```

### after this lets create pv and pvc for our k8s.

```

apiVersion: v1
kind: PersistentVolume
metadata:
  name: reg-pv-volume
  labels:
    type: local
spec:
  storageClassName: standard
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: reg-pv-claim
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### After deploying the deployment you can expose as nodeport,

### make a service to expose the port 

```
kubectl expose deployment registry --name registry-service --port 5000 --target-port 5000 --type=NodePort
```

### we will add the host machine ip into /etc/hosts

```
vi /etc/hosts
192.168.49.2 my-registry
```

### Now login to the registry 

```
docker login my-registry:31334 -u aum -p pa$$

```
![Screenshot from 2025-01-01 18-42-20](https://github.com/user-attachments/assets/2af26062-7bbe-45ec-a440-82c13c630004)


### After sucessfull login you are all set to upload retag any image and upload to your own repository
```
docker tag httpd:2  my-registry:31334/httpd:2
docker push my-registry:31334/httpd:2
```


