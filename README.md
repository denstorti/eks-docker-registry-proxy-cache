# eks-docker-registry-proxy-cache

POC reduce costs cross-region and cross-cloud by having Docker Registry as pull-through cache.

## Prerequisites

- Use AWS Cloudshell to enter commands
- eksctl
  - Install eksctl: https://eksctl.io/introduction/#installation
- kubectl
  - Install kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl/

## Create cluster

```
eksctl create cluster --name eks-docker-registry-proxy-cache --region us-east-1 --node-type t3.small --nodes 3 --node-volume-size 20 --spot
```

# Update kubeconfig

```
aws eks update-kubeconfig --name="eks-docker-registry-proxy-cache"
```

# Check current context

```
kubectl config current-context
```

```
kubectl get po -A
```

# Deploy Docker Registry Helm Chart

https://github.com/twuni/docker-registry.helm

```
helm repo add twuni https://helm.twun.io
helm repo update
```

Install the chart:
```
helm upgrade docker-registry twuni/docker-registry --namespace docker-registry-proxy-cache --create-namespace --values values.yaml
```

# Check the status of the deployment

```
echo "Visit https://127.0.0.1:5000/v2/_catalog to use your application"
kubectl -n docker-registry-proxy-cache port-forward svc/docker-registry 5000:5000
```

Should return:
```
{"repositories":[]}
```

# Configure TLS (self-signed) - Didn't work

Create secret:
```
MY_DOMAIN=docker-registry.example.com
mkdir -p certs

openssl req -x509 -newkey rsa:4096 -days 365 -nodes -sha256 \
  -keyout certs/tls.key -out certs/tls.crt \
  -subj "/CN=${MY_DOMAIN}" \
  -addext "subjectAltName=DNS:${MY_DOMAIN}"

kubectl create secret tls docker-certs-secret \
  --cert=./certs/tls.crt --key=./certs/tls.key
```

Update chart:
```
helm upgrade docker-registry twuni/docker-registry --namespace docker-registry-proxy-cache --create-namespace --set tlsSecretName=docker-certs-secret
```

Validate HTTPS by visiting https://127.0.0.1:5000/v2/_catalog.

# Configure TLS (cert-manager and ACM)

https://aws.amazon.com/blogs/containers/use-private-certificates-to-enable-a-container-repository-in-amazon-eks/

Problem: every node needs to trust the CA, `preBootstrapCommands` copying CA from S3 bucket. Required AWS Private CA.

# Configure TLS (NLB with TLS + cert-manager + Private CA)

Install ALB controller (https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html):


Cons:
- Node --> NLB --> Svc --> Registry pod

# Configure TLS (cert-manager and Let's Encrypt)


# Skopeo

Copy image to private registry:
```
skopeo copy docker://quay.io/buildah/stable docker://docker-registry.docker-registry-proxy-cache:5000/buildah --dest-tls-verify=false
```

Pull image from private registry:
```
curl -k https://docker-registry.docker-registry-proxy-cache:5000/v2/_catalog
```

Pull image from private registry using crane:
```
crane pull docker-registry.docker-registry-proxy-cache:5000/buildah buildah --insecure
```

STEPS TO FIX DEPLOYMENT:
- Run Docker Registry without TLS (remove tlsSecretName)
- Change SVC to NodePort and map to 30008
  - Possible to confirm from node `curl -L localhost:30008/v2/_catalog`
- Force Copy image to private registry using another pod (proxy not set yet): `skopeo copy docker://quay.io/buildah/stable docker://docker-registry.docker-registry-proxy-cache:5000/buildah --dest-tls-verify=false`
- Run image point to `kubectl run bu --image localhost:30008/buildah sleep 60`
- Image is pulled from private registry successfully.

Setting up proxy:
- kubectl create secret generic docker-io-auth -n docker-registry-proxy-cache --from-env-file=dockerio
- helm upgrade docker-registry twuni/docker-registry --namespace docker-registry-proxy-cache --create-namespace --values values.yaml



# Push image to Docker Registry

```
docker build -t localhost:5000/hello-world -f hello-world.Dockerfile .
docker push localhost:5000/hello-world
```



# Pull image from Docker Registry (to local machine)


# Pull image from Docker Registry (another pod)


# Clean up

```
eksctl delete cluster --name eks-docker-registry-proxy-cache
```
