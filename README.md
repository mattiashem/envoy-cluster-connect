# Envoy proxy service to connect them all

This is a proxy service that will connect diffrent env. We use it to connect our AWS EKS cluster to our Openshift cluster.
And to connect to databases in the old env.

With the cluster connect running we can deploy apps that still have deps on the old env. 

We also have some IP blocks and can only access some onlien service from some ip.
With this cluster connect we can send out traffic from thoose ip and still host our service in AWS.

No more i cant deploy untill that service is deploy ore that ip is open


## How it works

The setup is with tho envoy proxy servers one for the client in the cluster talk to. The proxy then send the request downstream to a proxy inside the env example prod. The downstream proxy then redirect the request to the correct host.

- The proxy will send request based on host headers 
- Pure TCP connections like mySQL have to done over port so eveery mysql need to have its own port
- All connections are over TLS
- Auth is done my PKI and client cert.



### Example of conenctions

(Client --> google-com.via-office.svc --> [proxy envoy]  EKS  )   ==== TLS =====  ([proxy envoy] --> google.com   )



## Setup

First we need to setup the PKI needed for the deploy. We use client and server certs signed by a CA to allow traffic. So the TLS tunnel between the cluster are secure


### PKI

Follow the pki docs in the pki docs to generate certs.
Then deploy the certs to the cluster with the secrets yaml

- make the CN name to the dns name you will use to connect with


When the certs are done do the following to get them into the secrets file

- Remove all enter på serach and replace all \n to empty
- Base64 the certs and key and add them to the certs.

downstream envoy (Openshift) will have the following certs

ca.pem = certs/ca-chain.cert.pem
tls.key = private/envoy.int.key.pem
tls.crt = certs/envoy.int.cert.pem


upstream envoy (kubernetes) will have the following certs


ca.pem = certs/ca-chain.cert.pem
tls.key = private/envoy.int-client.key.pem
tls.crt = certs/envoy.int-client.cert.pem



### Deploy  downstream

Now deploy the downstream envoy the config are for openshift so i use the oc commands.


```
oc apply -f secrets.yaml
oc apply -f envoy-ingress.yaml
oc apply -f deployment.yaml
```


So you need to have the nodeport open so you can connect to them now test that is working


```

curl -v (-k use this if you havet give the CN name the correct name) https://proxy.example.com:30002 --cacert ../pki/certs/ca-chain.cert.pem  --cert ../pki/certs/envoy.int-client.cert.pem  --key ../pki/private/envoy.int-client.key.pem -cert ../pki/certs/envoy.int-client.cert.pem

```

Also test remove without the cert and se so it gets block.
You should get a 404


```

curl -v (-k use this if you havet give the CN name the correct name) --header "Host:google-com.via-office.svc" https://proxy.example.com:30002 --cacert ../pki/certs/ca-chain.cert.pem  --cert ../pki/certs/envoy.int-client.cert.pem  --key ../pki/private/envoy.int-client.key.pem -cert ../pki/certs/envoy.int-client.cert.pem

```
Now we added the Host header and you should get a connection to google

### Upstream proxy

Deploy the upstream proxy in the kubernetes cluster so verify so you use the correct namespace


```
kubectl apply -f envoy-configs.yaml -n via-office
kubectl apply -f deploymemts.yaml -n via-office
```

then deploy our test client.


No you should be able to inside a continer in the cluster run 


```
curl -v http://google-com.via-office.svc
```
This will gice you a connection to google by the office 