# Kubernetes Ingress

By the end of this exercise, you should be able to:
- Understand the purpose of ingresses
- Know how to make your app publicly accessible.

## Ingress vs Service

With services, we exposed our pods to the internal kubernetes network. We also managed to load balance our traffic across multiple pods. Now, we also want to enable access to our pod from the outside world. 

## Ingress

Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.

An Ingress basically connects Services to externally-reachable URLs.


## Ingress Controller

In order for an Ingress to work, we first need to install an Ingress controller. For simplicity, we will install Ingress-Nginx. 
We also need to set several settings, to make it work with our current bare-metal setup.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx -n kube-system \
--set controller.hostNetwork=true,controller.kind=DaemonSet,controller.ingressClassResource.default=true,controller.watchIngressWithoutClass=true
```

## Demo App

For demonstration, lets just start a simple pod with a service:

```bash
kubectl run example --image strm/helloworld-http --port 80 --expose true 
```

This is a simple web server that will respond to requests with "Hello, World!".
Now, to expose the service to the outside world, we need to create an Ingress.


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example
            port:
              number: 80
```

This directs all http traffic to port 80 of the service.
Try out the service by typing the ip address of any node into your browser.