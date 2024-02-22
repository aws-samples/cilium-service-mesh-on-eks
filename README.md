# Elevate Your Amazon EKS Environment with Cilium Service Mesh 

This project shows the steps involved to implement the solution architecture explained in this AWS blog: [Elevate Your Amazon EKS Environment with Cilium Service Mesh]()

## 🎯 Prerequisites

- [ ] A machine which has access to AWS and Kubernetes API server.
- [ ] You need the following tools on the client machine.
	- [ ] [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
   	- [ ] [Terraform](https://developer.hashicorp.com/terraform/install)
  	- [ ] [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

Assumption : You already configured a [default] in the AWS CLI config/credentials files.

## 🧩 Solution

### Step 1 - Clone this GitHub repo to your machine

```bash
git clone https://github.com/aws-samples/cilium-mesh-on-eks/
cd cilium-mesh-on-eks
```

### Step 2 - Deploy EKS cluster with Terraform

```bash
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

```bash
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
LAST DEPLOYED: Wed Feb 21 23:46:36 2024
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
- Verify with `kubectl get pods -A` that status of cilium pods and cilium agents are `Running` state.
- Verify with `kubectl get svc -A` that the `cilium-ingress` service has an AWS load balancer DNS name assigned to it (in the `EXTERNAL-IP` of the output.

### Step 4 - Deploy Product Catalog Application

```bash
kubectl create namespace workshop

cd .. 
cd productapp
helm install productapp . -n workshop
```

Sample Output
```

```

### Step 5 - Configure Ingress to access the application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace : workshop
  name: workshopingress # name given to the ingress
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

```bash
CILIUM_INGRESS_URL=$(kubectl get svc cilium-ingress -n kube-system -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
echo "http://$CILIUM_INGRESS_URL"
```

Either use `curl` or a browser to navigate to the URL.

Sample Application Screenshot

### Step X - Create deployment specific services

```bash
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
### Step X - Add a product in the web page

### Step X - Deploy Traffic Shifting Policy (CiliumEnvoyConfig)

```bash
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

```bash
for i in {1..4}; do kubectl -n workshop exec -it <productcatalog-pod-name> -- curl catalogdetail:3000/catalogDetail; done
```


### Step X - Destroy

```bash
# Necessary to avoid removing Terraform's permissions too soon before its finished
# cleaning up the resources it deployed inside the cluster
terraform state rm 'module.eks.aws_eks_access_entry.this["cluster_creator"]' || true

terraform destroy -target="module.eks_blueprints_addons" -auto-approve
terraform destroy -target="module.eks" -auto-approve
terraform destroy -auto-approve
```

## 🔐 Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## 💼  License

This library is licensed under the MIT-0 License. See the LICENSE file.

