# argocd-app-config

## Apply argocd-cm.yaml if you are using app-of-apps pattern and orchestrating synchronization using sync waves

```
kubectl apply -f argocd-cm.yaml
```

## Create Argo Applications

```
kubectl apply -f main.yaml
```

## Cert-Manager

### Generate an rsa key
```
openssl genrsa -out .ssl/root-ca-key.pem 2048
```
### Generate root certificate
```
openssl req -x509 -new -nodes -key .ssl/root-ca-key.pem \
  -days 3650 -sha256 -out .ssl/root-ca.pem -subj "/CN=kube-ca"
```
### Generate sealed-secret root-ca
```
kubectl create secret tls -n cert-manager root-ca \
    --cert=/opt/ca-certificates/root-ca.pem \
    --key=/opt/ca-certificates/root-ca-key.pem --dry-run=client -oyaml | \
    kubeseal --controller-name=sealed-secrets -oyaml > \ resources/cert-manager/root-ca.yaml
```

## Linkerd
### Install step CLI tool and then create trust public and private keys

```
step certificate create root.linkerd.cluster.local sample-trust.crt sample-trust.key \
  --profile root-ca \
  --no-password \
  --not-after 43800h \
  --insecure
```

### Create sealed secret trust-anchor.yaml

```
kubectl create ns linkerd
kubectl -n linkerd create secret tls linkerd-trust-anchor \
  --cert sample-trust.crt \
  --key sample-trust.key \
  --dry-run=client -oyaml | \
kubeseal --controller-name=sealed-secrets -oyaml - | \
kubectl patch -f - \
  -p '{"spec": {"template": {"type":"kubernetes.io/tls", "metadata": {"labels": {"linkerd.io/control-plane-component":"identity", "linkerd.io/control-plane-ns":"linkerd"}, "annotations": {"linkerd.io/created-by":"linkerd/cli stable-2.12.0"}}}}}' \
  --dry-run=client \
  --type=merge \
  --local -oyaml > resources/linkerd/trust-anchor.yaml
```

### References
https://argo-cd.readthedocs.io/en/stable/operator-manual/health/#argocd-app