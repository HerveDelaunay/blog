+++
title = 'Adding TLS to Your Ingress Traffic'
date = 2024-06-22T10:26:00+02:00
draft = false
tags = ['kubernetes', 'ingress']
+++

Today at my job I managed to get our new product up and running on kubernetes, deployment and statefulsets are working but I wanted to test the app's behavior while accessing it from the outside. As we have multiple services running on the cluster and as we want to secure the connection through TLS it is best to use an Ingress (resources and controllers). I already did the Ingress controller setup earlier, using [Ingress-nginx-controller](https://kubernetes.github.io/ingress-nginx/) so I just had to create an Ingress-resources and I wanted to discuss about my process here.

First and foremost we need an access to our fully qualified domain certificate. In my case I use a "wildcard certificate" that backup all the sub domains of my domain : 

![certificates-1](/Pasted-image-20240621160210.png)

If we take a look at the Ingress TLS section of the [kubernetes docs](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls) we can find a sample Ingress resource manifest as such : 
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80

```

The TLS section of the manifest is mentioning a *secret*, that is the secret containing the tls certificate and key. 

To generate the secret : 
```bash
kubectl create secret tls <secret-name> --cert=./path-to-your-certificate --key=./path-to-your-key
```

![certificates-2](/Pasted-image-20240621161528.png)

Let's take a look at the secret : 
```bash
k get secret -n herve tls-secret -o yaml
```
Output : 
```yaml
apiVersion: v1
data:
  tls.crt: <encoded-data>
  tls.key: <encoded-data>
kind: Secret
metadata:
  creationTimestamp: "2024-06-21T14:14:22Z"
  name: tls-secret
  namespace: herve
  resourceVersion: "15599806"
  uid: 8f0h6e84-7678d-67581-ad25-c72acwggbta29
type: kubernetes.io/tls
```
We can see that the `kubectl create secret tls` command took our cert and key file, encoded them and finally embedded them inside the secret object (I've hidden my cert and key data here).

Now we just have to mention the secret under the TLS section of our Ingress resource :

```yaml
  tls:
  - hosts:
      - https-example.foo.com
    secretName: tls-secret
```

As I am using an Ingress-nginx-controller I have to mention its name under the `ingressClassName` section : 

![certificates-3](/Pasted-image-20240621163623.png)
*Keep in mind that the ingress resource should be created inside the same namespace as the workload it is targeting, it is not done here. And the service name should be the service representing the targeted pods*

The traffic will now be secured with TLS, and the users will be able to request our service through https.
