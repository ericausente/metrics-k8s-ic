# metrics-k8s-ic


#Reading resource utilization metrics of various Ingress Controller Pods

Reference: 
https://signoz.io/blog/kubectl-top/
https://github.com/kubernetes-sigs/metrics-server


```
e.ausente@C02DR4L1MD6M ~ % kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```


# Installation of NGINX Ingress Controller (based on NGINX plus) with Helm
https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/

Go back to your home directory: 
```
$ cd ~
```

Clone the Ingress Controller repo:
```
$ git clone https://github.com/nginxinc/kubernetes-ingress.git --branch v2.3.0
```

Change your working directory to /deployments/helm-chart:
```
$ cd kubernetes-ingress/deployments/helm-chart
```

# Adding the Helm Repository
This step is required if you’re installing the chart via the helm repository.
```
$ helm repo add nginx-stable https://helm.nginx.com/stable
$ helm repo update
% helm repo list
NAME         	URL                                       
ingress-nginx	https://kubernetes.github.io/ingress-nginx
nginx-stable 	https://helm.nginx.com/stable       -------------->>>> 
```

# Using the NGINX IC Plus JWT token in a Docker Config Secret

Purpose: NGINX Plus Ingress Controller image from the F5 Docker registry in your Kubernetes cluster by using your NGINX Ingress Controller subscription JWT token 
Refer to https://docs.nginx.com/nginx-ingress-controller/installation/using-the-jwt-token-docker-secret/

1. Create a docker-registry secret on the cluster using the JWT token as the username, and none for password (password is unused). The name of the docker server is private-registry.nginx.com. Optionally namespace the secret.

Make sure to save your nginx-repo.jwt in /tmp/ directory. 

```
kubectl create secret docker-registry regcred --docker-server=private-registry.nginx.com --docker-username=$(cat /tmp/nginx-repo.jwt) --docker-password=none
```

Confirm the details of the created secret by running:
```
kubectl get secret regcred --output=yaml
```

Go back to your home directory: 
```
cd ~
```

Install the NGINX Plus Ingress Controller using Helm and specify the new ingress class nginx-plus using the --set controller.ingressClass=nginx-plus flag.

The rest of the command sets various configuration options for the Ingress Controller, such as the image repository, NGINX Plus features, and service settings.

By setting controller.nginxplus=true, you are enabling NGINX Plus features like dynamic reconfiguration, session persistence, and advanced traffic management.

By setting controller.appprotect.enable=true, you are enabling the NGINX App Protect module for application security and protection.

The controller.service.type=LoadBalancer flag creates an AWS Elastic Load Balancer to route traffic to your Ingress Controller, and controller.service.externalTrafficPolicy=Cluster ensures that traffic is routed to the Kubernetes nodes running your Ingress Controller pods instead of being directly routed to the pods.

By setting controller.ingressClass=nginx-plus, you have specified the new ingress class that you created earlier and can now use this class to route traffic to your Kubernetes services and applications using the NGINX Plus Ingress Controller.
  
  Command:

  ```
helm install ingress-plus nginx-stable/nginx-ingress --set controller.ingressClass=nginx-plus --set controller.image.repository=private-registry.nginx.com/nginx-ic-nap/nginx-plus-ingress --set controller.nginxplus=true --set controller.appprotect.enable=true --set controller.image.tag=3.0.1 --set controller.serviceAccount.imagePullSecretName=regcred --set controller.service.type=LoadBalancer --set controller.service.httpsPort.nodePort=32137 --set controller.replicaCount=1 --set controller.kind=deployment --set prometheus.create=true --set controller.readyStatus.initialDelaySeconds=30 --set controller.service.externalTrafficPolicy=Cluster
```

OUTPUT: 
```
% helm install ingress-plus nginx-stable/nginx-ingress --set controller.ingressClass=nginx-plus --set controller.image.repository=private-registry.nginx.com/nginx-ic-nap/nginx-plus-ingress --set controller.nginxplus=true --set controller.appprotect.enable=true --set controller.image.tag=3.0.1 --set controller.serviceAccount.imagePullSecretName=regcred --set controller.service.type=LoadBalancer --set controller.service.httpsPort.nodePort=32137 --set controller.replicaCount=1 --set controller.kind=deployment --set prometheus.create=true --set controller.readyStatus.initialDelaySeconds=30 --set controller.service.externalTrafficPolicy=Cluster

NAME: ingress-plus
LAST DEPLOYED: Fri Mar  3 17:17:19 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The NGINX Ingress Controller has been installed.

 e.ausente@C02DR4L1MD6M eks-play % kubectl get ingressclass
NAME         CONTROLLER                     PARAMETERS   AGE
nginx        k8s.io/ingress-nginx           <none>       4h24m
nginx-plus   nginx.org/ingress-controller   <none>       9s

% kubectl get svc
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.245.69    acf68105edd024bfe9fb72111124bfaf-32909998.ap-southeast-1.elb.amazonaws.com    80:31594/TCP,443:31068/TCP   4h25m
ingress-nginx-controller-admission   ClusterIP      10.100.123.13    <none>                                                                        443/TCP                      4h25m
ingress-plus-nginx-ingress           LoadBalancer   10.100.249.211   a31ce143913004c26a8386cbbe71434b-176863799.ap-southeast-1.elb.amazonaws.com   80:30711/TCP,443:31900/TCP   63s
kubernetes                           ClusterIP      10.100.0.1       <none>                                                                        443/TCP                      6h51m
nginx-dep                            ClusterIP      10.100.208.228   <none>                                                                        80/TCP  
```
