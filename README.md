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
terraform apply --auto-approve
```

It takes 15 minutes for an EKS cluster creation to be ready. Terraform script updates the kubeconfig file automatically. 

Verify that the worker nodes status is Ready by doing `kubectl get nodes`.


## 🔐 Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## 💼  License

This library is licensed under the MIT-0 License. See the LICENSE file.

