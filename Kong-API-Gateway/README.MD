
## CERTIFICATES:
>Domain CN/SANs for CP services; CN for the wild card cert for cp-cluster communication: *.kong-cp.svc.cluster.local; For the rest: *.192.168.29.48.nip.io 
###### admin_api_uri
admin.192.168.29.48.nip.io
###### admin_gui /manager(svc)
manager.192.168.29.48.nip.io 
###### portal_api /portalapi(svc)
portal-api.192.168.29.48.nip.io
###### portal_gui /portal
portal-dev.192.168.29.48.nip.io
###### cluster
cp-kong.192.168.29.48.nip.io
###### cluster telemetry
cp-tel.192.168.29.48.nip.io
###### DP. 
dp.192.168.29.48.nip.io

### Cert Generation:
Step 1: Generate Open SSl cert 

```
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.crt 
```

Step2: Generate private key and CSR(certificate signing request) for each: CP(one cert can include multiple SANs mentioned above), DP 

Generate a private key CSR for CP (cp-tls, cp-cluster) & DP
```
openssl req -new -newkey rsa:2048 -nodes -keyout cp-tls.key -out cp-tls.csr -subj "/CN=*.192.168.29.48.nip.io"
openssl req -new -newkey rsa:2048 -nodes -keyout dp-tls.key -out dp-tls.csr -subj "/CN=*.192.168.29.48.nip.io"

openssl genrsa -out cp-cluster.key 2048
openssl req -new -key cp-cluster.key -out cp-cluster.csr -subj "/CN=*.kong-cp.svc.cluster.local"
```

Step3: Use ca from step1 to sign the generated CSRs from step2

Generate a self-signed cert for CP(cp-tls, cp-cluster) & DP
```
openssl x509 -req -in cp-tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out cp-tls.crt -days 365
openssl x509 -req -in dp-tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dp-tls.crt -days 365
openssl x509 -req -in cp-cluster.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out cp-cluster.crt -days 365 -sha256
```

Step4: create bundles (cp.cert and ca.cert will be one bundle) .
```
cat cp-tls.crt ca.crt > cp-tls-bundle.crt
cat cp-cluster.crt ca.crt > cp-cluster-bundle.crt
cat dp-tls.crt ca.crt > dp-tls-bundle.crt
```

>To verify what SANs/ DNS the cert is serving:
openssl x509 -in cp-tls.crt -text -noout 


## Kong Installations

### CP
Create ns, Create secrets, update helm repo:
```
sudo snap install helm --classic
helm repo add kong https://charts.konghq.com 
helm repo update

kubectl create namespace kong-cp
kubectl create namespace kong-dp
```

Create session config secret
```
kubectl create secret generic kong-session-config -n kong-cp \
 --from-literal=portal_session_conf='{"storage":"kong","secret":"super_secret_salt_string","cookie_name":"portal_session","cookie_samesite":"off","cookie_secure":false, "cookie_domain": ".192.168.29.48.nip.io"}' \
 --from-literal=admin_gui_session_conf='{"cookie_name":"admin_session","cookie_samesite":"off","secret":"super_secret_salt_string","cookie_secure":false,"storage":"kong", "cookie_domain": ".192.168.29.48.nip.io"}'
 ```

Create kong admin password secret
```
kubectl create secret generic kong-manager-password -n kong-cp \
 --from-literal=password=password
 ```

Create postgres secrets
```
kubectl create secret generic kong-cp-postgresql -n kong-cp \
 --from-literal=password=password
 ```

Create a Kubernetes Secret for the Certificates for ns kong-cp.
```
kubectl create secret tls kong-tls-cert -n kong-cp \
 --cert=cp-tls-bundle.crt \
 --key=cp-tls.key
 
kubectl create secret tls kong-cluster-cert -n kong-cp \
 --cert=certs/cp-cluster-bundle.crt \
 --key=cp-cluster.key

kubectl create secret generic kong-ca-cert -n kong-cp \
 --from-file=ca.crt=ca.crt
 ```
> Update values-cp.yaml and install CP:
```
helm upgrade --install kong-cp kong/kong --namespace kong-cp -f values-cp.yaml
```



##### DP
Repeat the secret gen while installing DP
```
kubectl create secret tls kong-cluster-cert -n kong-dp \
 --cert=dp-tls-bundle.crt \
 --key=dp-tls.key

kubectl create secret generic kong-ca-cert -n kong-dp \
 --from-file=ca.crt=ca.crt
```
> Update values-dp.yaml and install DP:
```
helm upgrade --install kong-dp kong/kong --namespace kong-dp -f values-dp.yaml
```

#####
Once the installations are done, add a service and a route through admin and verify it through dp.
```
curl -i -k -X POST https://admin.192.168.29.48.nip.io/services \
  --data name=test-service \
  --data url='http://httpbin.org' -H "Kong-Admin-Token:password"


curl -i -k -X POST https://admin.192.168.29.48.nip.io/services/test-service/routes \
  --data 'paths[]=/test' \
  --data name=test-route -H "Kong-Admin-Token:password"
```