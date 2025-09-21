+++
title = 'Secure GitOps : Exposing Internal Kubernetes Services With Cloudflare Tunnels'
date = 2025-09-21T12:29:44+02:00
draft = false
+++

It has been some time that I've been searching for a way to expose my self-hosted apps without opening my network to the internet. I've finally found the solution : Cloudflare tunnels. They only require an outbound connection and remove the need to expose any ports directly.

Requirements:
- A Cloudflare-managed domain (you can also transfer an existing domain to Cloudflare)  
- A Kubernetes cluster (in my case, managed with FluxCD following GitOps best practices)

## Authenticating and creating the tunnel

First, [download cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/) on your machine

Then authenticate to Cloudflare :

`cloudflared tunnel login`

This will open your browser, ask for your Cloudflare credentials, and create a cert at `~/.cloudflared/cert.pem`

Next, create the tunnel from the CLI. In my case, I want to expose the Linkding app:

`cloudflared tunnel create linkding`

This command generates a `.json` file containing the tunnel's tokens. This is the file that allows us to create a tunnel from our `cloudflared` deployment to the cloudflare network.

We'll want to transform this file in a kubernetes secret inside of our cluster : 

```
kubectl create secret generic tunnel-credentials \
--from-file=credentials.json=<YOUR_TUNNEL_ID>.json \
--dry-run=client \
-o yaml > tunnel_credentials.yaml
```

## Encrypting secrets with SOPS and Age

As we are using FluxCD as our continuous deployment tool, we need to keep in mind that all our code is going to get pushed to Github. We need a way to encrypt our secret before pushing it to our repo. 

For this we'll use `sops` encryption with an `age` key as described in [Flux's docs](https://fluxcd.io/flux/guides/mozilla-sops/#encrypting-secrets-using-age)

Essentially, we'll encrypt our `tunnel-credentials` secret with our `age` public key. We'll then want to place our age *private key* inside of our cluster, and we'll do it manually. The `tunnel-credentials` secret will then be decripted by Flux, using the sops private key living inside of our cluster as a secret object.

Let's first create the age *keypair* :
```console
# if not already installed
$ brew install sops age

$ age-keygen -o age.agekey
Public key: age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xtz0hpfwq98cmsg
```

We then *imperatively* create a secret object on the cluster, containing our age *private key*

```bash
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```
*the key name must end with `.agekey` to be detected as an age key*

Now that the private key is in the cluster, we can encrypt the `tunnel-credentials` file with sops and our age public key

```bash
cat age.agekey

sops --age=age1helqcqsh9464r8chnwc2fzj8uv7vr5ntnsft0tn45v2xtz0hpfwq98cmsg \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place tunnel-credentials.yaml
```

## Configuring FluxCD for Decryption

In order for Flux to know it has to decrypt `tunnel-credentials` with Sops, you'll need to add this in the Flux Kustomization object pointing to your apps : 

```yaml
  # note that the secret is called sops-age in my cluster
  # it's our age private key
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

In my case, I want Flux to be aware that there will be a Sops encrypted file in my `/apps/staging` directory : 

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m0s
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/staging
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

At this point, we can commit and push our code safely, Flux will pick it up and apply it in our cluster, decoding `tunnel-credentials.yaml` with the age secret.

## Deploying Cloudflared in the Cluster

Right now Flux can manage the `tunnel-credentials` Secret securely. But nothing is actually running the tunnel yet. We need a `cloudflared` Deployment in the cluster that connects to Cloudflare’s edge and routes traffic to our internal app.

Here’s an example Deployment and ConfigMap that run `cloudflared` with the tunnel we created earlier (in my case, `linkding`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
spec:
  selector:
    matchLabels:
      app: cloudflared
  replicas: 2
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
      - name: cloudflared
        image: cloudflare/cloudflared:latest
        args:
        - tunnel

        # Points cloudflared to the config file, which configures what
        # cloudflared will actually do. This file is created by a ConfigMap
        # below.
        - --config
        - /etc/cloudflared/config/config.yaml
        - run
        livenessProbe:
          httpGet:
            path: /ready
            port: 2000
          failureThreshold: 1
          initialDelaySeconds: 10
          periodSeconds: 10
        volumeMounts:
        - name: config
          mountPath: /etc/cloudflared/config
          readOnly: true
        # Each tunnel has an associated "credentials file" which authorizes machines
        # to run the tunnel. cloudflared will read this file from its local filesystem,
        # and it'll be stored in a k8s secret.
        - name: creds
          mountPath: /etc/cloudflared/creds
          readOnly: true
      volumes:
      - name: creds
        secret:
          secretName: tunnel-credentials

      # Create a config.yaml file from the ConfigMap below.
      - name: config
        configMap:
          name: cloudflared
          items:
          - key: config.yaml
            path: config.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared
data:
  config.yaml: \|
    # Name of the tunnel you want to run
    
    tunnel: linkding

    credentials-file: /etc/cloudflared/creds/credentials.json

    metrics: 0.0.0.0:2000
    no-autoupdate: true

    ingress:
    - hostname: linkding.hervedelaunay.com
      service: http://linkding:9090

    - hostname: hello.example.com
      service: hello_world
    
	- service: http_status:404

```

This tells `cloudflared` to:

- use the credentials we stored in the `tunnel-credentials` Secret,
- run the tunnel named `linkding`,
- and route requests from `linkding.hervedelaunay.com` to the Linkding service running in the cluster.

## Wrapping Up

With this setup, you can safely manage Cloudflare Tunnel credentials in Git, thanks to FluxCD and SOPS. 

Your services stay private - only outbound traffic is needed - while still being accessible securely through Cloudflare’s network.  

This approach works not just for Linkding, but for any self-hosted app you want to expose securely.
