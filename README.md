# Elevate Your Amazon EKS Environment with Cilium Service Mesh 

This project shows the steps involved to implement the solution architecture explained in this AWS blog: [Elevate Your Amazon EKS Environment with Cilium Service Mesh]()

## 🎯 Prerequisites

- [ ] A machine which has access to AWS and Kubernetes API server.
- [ ] You need the following tools on the client machine.
	- [ ] [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
   	- [ ] [Terraform](https://developer.hashicorp.com/terraform/install)
  	- [ ] [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
  	- [ ] [Cilium CLI](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli)

Assumption : You already configured a [default] in the AWS CLI config/credentials files.

## 🧩 Solution

### Step 1 - Clone this GitHub repo to your machine

```
git clone https://github.com/aws-samples/cilium-mesh-on-eks/
cd cilium-mesh-on-eks
```

### Step 2 - Deploy EKS cluster with Terraform

```
cd terraform
terraform init
terraform apply --auto-approve
```

Sample Output
```
Apply complete! Resources: 65 added, 0 changed, 0 destroyed.

Outputs:

configure_kubectl = "aws eks --region us-west-2 update-kubeconfig --name terraform"
```

It takes ~15 minutes for an EKS cluster creation process to complete. Update kubeconfig using the command provided in the Terraform output. 

Verify that the worker nodes status is `Ready` by doing `kubectl get nodes`.

### Step 3 - Deploy Cilium on EKS cluster with Helm

```
helm upgrade --install cilium cilium/cilium --version 1.14.7 \
--namespace kube-system \
--reuse-values -f ~/cilium-mesh-on-eks/values_cilium.yaml \
--set hubble.enabled=true \
--set hubble.tls.auto.enabled=true \
--set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp,http}" \
--set hubble.relay.enabled=true \
--set hubble.ui.enabled=true \
--set hubble.ui.service.type=NodePort \
--set hubble.relay.service.type=NodePort \
--set kubeProxyReplacement=strict \
--set encryption.enabled=false \
--set encryption.nodeEncryption=false \
--set routingMode=native \
--set ipv4NativeRoutingCIDR="0.0.0.0/0" \
--set bpf.masquerade=false \
--set nodePort.enabled=true \
--set autoDirectNodeRoutes=true \
--set hostLegacyRouting=false \
--set ingressController.enabled=true \
--set ingressController.loadbalancerMode=shared \
--set cni.chainingMode=aws-cni \
--set cni.install=true
```

Sample Output
```
Release "cilium" does not exist. Installing it now.
NAME: cilium
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
You have successfully installed Cilium with Hubble Relay and Hubble UI.

Your release version is 1.14.7.

For any further help, visit https://docs.cilium.io/en/v1.14/gettinghelp
```

- A few parameters worth mentioning from above : 
  - We replace kube-proxy functionality with Cilium' s own eBPF based implementation.
  - We enable Cilium Ingress Controller.
    - We use a specific annotation from values_cilium.yaml so that Cilium Ingress can be exposed through an AWS Network Load Balancer.
  - We enable Hubble.
- Verify with `kubectl get pods -A` that status of Cilium pods and Cilium agents are `Running` state.
- Verify with `kubectl get svc -A` that the `cilium-ingress` service has an AWS load balancer DNS name assigned to it (in the `EXTERNAL-IP` of the output.

### Step 4 - Deploy Product Catalog Application

```
kubectl create namespace workshop

cd .. 
cd productapp
helm install productapp . -n workshop
```

Sample Output
```
namespace/workshop created
NAME: productapp
NAMESPACE: workshop
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Product Catalog Application is succesfully installed !

1. Get the application URL by running the following command:
  
  CILIUM_INGRESS_URL=$(kubectl get svc cilium-ingress -n kube-system -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
  echo "http://$CILIUM_INGRESS_URL"

2. Once you configure the ingress in the next step you will be able to access this URL in a terminal using "curl" or via a browser window
```

### Step 5 - Investigate the Product Catalog Application

Application architecture is as shown below. 

APP ARCHICTURE DIAGRAM HERE / APP ARCHICTURE DIAGRAM HERE / APP ARCHICTURE DIAGRAM HERE

The user accesses `Frontend` microservice, then `Frontend` microservice calls the `Product Catalog` service, and then `Product Catalog` service calls the `Catalog Detail` microservice. The `Catalog Detail` microservice is comprised of two different deployments which are actually identical. The reason we use two deployments is to demonstrate traffic shifting capabilities of service mesh at a later step. 

Have a look at the resources deployed as part of the `Product Catalog Application`. 

```
kubectl get deployment,pod,service  -n workshop
```

Sample Output
```
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/catalogdetail    1/1     1            1           21m
deployment.apps/catalogdetail2   1/1     1            1           21m
deployment.apps/frontend         1/1     1            1           21m
deployment.apps/productcatalog   1/1     1            1           21m

NAME                                  READY   STATUS    RESTARTS   AGE
pod/catalogdetail-5896fff6b8-tc75k    1/1     Running   0          21m
pod/catalogdetail2-7d7d5cd48b-b9m7t   1/1     Running   0          21m
pod/frontend-78f696695b-tvh9p         1/1     Running   0          21m
pod/productcatalog-64848f7996-gpnl7   1/1     Running   0          21m

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/catalogdetail     ClusterIP   172.20.15.14     <none>        3000/TCP   21m
service/frontend          ClusterIP   172.20.95.212    <none>        9000/TCP   21m
```

Notice that there are two deployments for the `Catalog Detail` microservice; `catalogdetail` and `catalogdetail2`. The `catalogdetail` Kubernetes service points out to both `catalogdetail-....` and `catalogdetail2-....` pods.  Meaning that a request from the `productcatalog-.....` pod to the `catalogdetail` service can get forwarded to any of those pods. You can verify this by checking the `kubectl describe service catalogdetail` output. This is important to note since it will become relevant in the traffic shifting scenario later on.


### Step 5 - Configure Ingress to access the application

```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace : workshop
  name: productappingress # name given to the ingress
spec:
  ingressClassName: cilium
  rules:
  - http:
      paths:
      - path: / # this rule applies to all requests that specifies this path
        pathType: Prefix
        backend:
          service:
            name: frontend # route all these requests to this service
            port:
              number: 9000 # route the requests to this port of the frontend service
EOF
```

### Step 6 - Access the Product Catalog Application

Get the URL to access the application. 

```
CILIUM_INGRESS_URL=$(kubectl get svc cilium-ingress -n kube-system -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
echo "http://$CILIUM_INGRESS_URL"
```

Access the application URL either using `curl` or a browser. You should see the following web page.

Sample output

### Step 7 - Access Cilium Hubble UI for Service Map Visualization

You can use Cilium Hubble to visualize service dependencies. Use the commands in the following command snippet. You will see a new browser tab automatically being spun up and see the Hubble UI on that page. Select `workshop` namespace in there.

Below is a sample screenshot.

HUBBLE SCREENSHOT / HUBBLE SCREENSHOT / HUBBLE SCREENSHOT / HUBBLE SCREENSHOT





### Step 8 - Create deployment specific services

To test traffic shifting capabilities of Cilium we will create two additional Kubernetes service resources. `catalogdetailv1` service will be selecting the pods within the `catalogdetail` deployment. `catalogdetailv2` service will be selecting the pods within the `catalogdetail2`deployment.

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  labels:
    app: catalogdetail
  name: catalogdetailv1
  namespace: workshop
spec:
  ports:
  - name: http
    port: 3000
    protocol: TCP
    targetPort: 3000
  selector:
    app: catalogdetail
    version: v1
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: catalogdetail
  name: catalogdetailv2
  namespace: workshop
spec:
  ports:
  - name: http
    port: 3000
    protocol: TCP
    targetPort: 3000
  selector:
    app: catalogdetail
    version: v2
EOF
```
### Step 8 - Add a product on the web page

On the application web page add a product. Any id and name is fine. 

SAMPLE SCREENSHOT / SAMPLE SCREENSHOT / SAMPLE SCREENSHOT 

Refresh the page and notice that the vendors list sometimes shows `ABC.com` only and some other times both `ABC.com` and `XYZ.com`. This is because the product information is persisted in the `Product Catalog` microservice and the vendor information is persisted in the `Catalog Detail` microservice. When you add a product to the list, the randomized vendor information for the product you added gets persisted on one of the `catalogdetail` pods. Either `catalogdetail-...` pod which is part of `catalogdetail` deployment or `catalogdetail2-...` pod which is part of the `catalogdetail2` deployment.


### Step X - Deploy Traffic Management Policy (CiliumEnvoyConfig)

Let' s now define a traffic management policy to send half of the requests to the `catalogdetailv1` service and the other half to the `catalogdetailv2` service. 

```
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumEnvoyConfig
metadata:
  name: traffic-shifting-test
  namespace: workshop
spec:
  services:
    - name: catalogdetail
      namespace: workshop
  backendServices:
    - name: catalogdetailv1
      namespace: workshop
    - name: catalogdetailv2
      namespace: workshop
  resources:
    - "@type": type.googleapis.com/envoy.config.listener.v3.Listener
      name: traffic-shifting-test
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: traffic-shifting-test
                rds:
                  route_config_name: lb_route
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
    - "@type": type.googleapis.com/envoy.config.route.v3.RouteConfiguration
      name: lb_route
      virtual_hosts:
        - name: "lb_route"
          domains: [ "*" ]
          routes:
            - match:
                prefix: "/"
              route:
                weighted_clusters:
                  clusters:
                    - name: "workshop/catalogdetailv1"
                      weight: 50
                    - name: "workshop/catalogdetailv2"
                      weight: 50
                retry_policy:
                  retry_on: 5xx
                  num_retries: 3
                  per_try_timeout: 1s
    - "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
      name: "workshop/catalogdetailv1"
      connect_timeout: 2s
      lb_policy: ROUND_ROBIN
      type: EDS
      outlier_detection:
        split_external_local_origin_errors: true
        consecutive_local_origin_failure: 2
    - "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
      name: "workshop/catalogdetailv2"
      connect_timeout: 2s
      lb_policy: ROUND_ROBIN
      type: EDS
      outlier_detection:
        split_external_local_origin_errors: true
        consecutive_local_origin_failure: 2
EOF
```

### Step X - Test access from ProductCatalog service to CatalogDetail service

Let' s send requests from the `Product Catalog` microservice to `Catalog Detail` microservice and see that there is an even distribution of requests to both deployments of the `Catalog Detail` microservice. 

```
for i in {1..6}; do echo "Output $i:"; kubectl -n workshop exec -it productcatalog-64848f7996-bh7ch -- curl catalogdetail:3000/catalogDetail; echo ""; done
```

Sample output
```
Output 1:
{"version":"2","vendors":["ABC.com, XYZ.com"]}
Output 2:
{"version":"1","vendors":["ABC.com"]}
Output 3:
{"version":"2","vendors":["ABC.com, XYZ.com"]}
Output 4:
{"version":"2","vendors":["ABC.com, XYZ.com"]}
Output 5:
{"version":"1","vendors":["ABC.com"]}
Output 6:
{"version":"1","vendors":["ABC.com"]}
```

### Step X - Uninstall Product Catalog Application and Cilium

```
helm uninstall productapp -n workshop
kubectl delete ingress productappingress -n workshop
kubectl delete svc catalogdetailv1 -n workshop
kubectl delete svc catalogdetailv2 -n workshop
helm uninstall cilium -n kube-system
```



### Step X - Destroy

```
# Necessary to avoid removing Terraform's permissions too soon before its finished
# cleaning up the resources it deployed inside the cluster
terraform state rm 'module.eks.aws_eks_access_entry.this["cluster_creator"]' || true
terraform state rm 'module.eks.aws_eks_access_policy_association.this["cluster_creator_admin"]' || true

terraform destroy -target="module.eks_blueprints_addons" -auto-approve
terraform destroy -target="module.eks" -auto-approve
terraform destroy -auto-approve
```

## 🔐 Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## 💼  License

This library is licensed under the MIT-0 License. See the LICENSE file.

