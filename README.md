<h1 align="center">
  <img src="assets/flynnt-logo.svg" alt="flynnt" width="100">
+
  <img src="assets/strimzi-logo.png" alt="strimzi" width="100">
</h1>

<h4 align="center">Deploy Kafka through Strimzi on a Flynnt Cluster</h4>

---

This repository contains resources that deploy an instance of [Kafka](https://kafka.apache.org/) through [strimzi](https://strimzi.io/) on a [flynnt](https://flynnt.io) kubernetes cluster.
It is build for private Cloud and on-premise environments.

The deployment is GDPR/DSGVO-compliant. This means, no data stored in your ArgoCD instance will leave your server. As long as you use trusted infrastructure providers (for example your own datacenter) your data is safe.

It is meant to be used for reference and as a blueprint for your own deployment.

#### Special features of this deployment
- Uses [Hetzner Cloud](https://www.hetzner.com/cloud) and the [Hetzner CSI Driver](https://github.com/hetznercloud/csi-driver) for compute nodes and as the StorageClass
- You can optionally use ArgoCD for your deployment. This is recommended, but not mandatory. The alternative is plain Helm/Kustomize. See here for a sample on [how to deploy ArgoCD](https://github.com/flynnt-io/flynnt-argocd-sample)

> **Note**
>
> Even though this sample is built to be deployed on a flynnt managed kubernetes cluster, you can easily customize this to use a different managed k8s provider.
> In general, almost every technology choice made here is opinionated and exchangeable with different products.

## Used Tooling

We use several open-source tools and stitch them together for a nice, standalone deployment experience.
- Terraform, Helm and Kustomize for deploying infrastructure, auxiliary apps and the operator itself
- Sops and Age for secret encryption and handling

### Binaries
- [kustomize](https://github.com/kubernetes-sigs/kustomize/releases)
- [sops](https://github.com/getsops/sops/releases)
- [ksops](https://github.com/viaduct-ai/kustomize-sops/releases)
- [age-keygen / age](https://github.com/FiloSottile/age/releases)

## Prerequisites and External Dependencies

- At least two nodes to use in your cluster in different availability zones. (You can use the terraform instructions from below)
- A way to store secrets and share them with your team. As an example, a keepass database is sufficient.
- A kubernetes cluster and access via kubectl to it. This example uses a flynnt cluster. Save the `kubeconfig.yaml` in the root of this repo.
- (optional) A place to store the terraform state. If you work in a team, you should use some form of shared storage.
- (optional) An external prometheus-compatible monitoring layer. Because of the shared failure-domain with the application, it's not recommended to deploy prometheus, alertmanager and grafana in the same cluster.

## Getting Started

### Generate a new age secret
We included the `age`-binaries in this repository. Feel free to update or remove them. They are only used for this step.

```bash
./age/age-keygen
```
Copy the private and public key of the output and save it to your secret database. Both keys will be used throughout this repository.

### Deploy infrastructure with Terraform (optional)

You should only do this, if your flynnt cluster does not have any nodes. This uses [Hetzner Cloud](https://www.hetzner.com/cloud) to deploy compute nodes and you need an [Hetzner API Key](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/) to use this.

We have some secrets that we want to use with terraform. Namely, our `hcloud_token` and the `flynnt_token`. We obviously don't want to store them in plain text in git, so we need to encrypt them first.

To do this, there is a `terraform/secrets.sample.yaml` file in this repo. It contains sample values. Replace them with real values. Next, we encrypt the file to be used by terraform as variables.

```bash
./sops --encrypt --age <put-age-public-key-here> terraform/secrets.sample.yaml > terraform/secrets.enc.yaml
```

> **Important**
> Don't commit the plain secrets file to git. The `secrets.enc.yaml` is fine to commit though. That's the whole point of `sops` and `age`.

Next, we will deploy the nodes through terraform and join them to a flynnt cluster. If you want to use a different backend to store your state, customize the `terraform/main.tf` accordingly.
Also, check the `terraform/terraform.tfvars` file for the correct cluster name.

```bash
export SOPS_AGE_KEY=<put-private-secret-key-here>
terraform -chdir=terraform init -upgrade #(only on first deploy)
terraform -chdir=terraform apply
```
Check that the nodes are successfully added to the cluster. Either through the [Flynnt Dashboard](https://app.flynnt.io) or directly by using `kubectl get nodes`

### Prepare k8s and sops secrets

We need to prepare a secret for this deployment to work.

The secret is for the Hetzner CSI Driver to access the Hetzner API. This is optional and not needed if you are using a different CSI provider.
It's located in `applications/hetzner-csi-driver/secrets.sample.yaml`. Change it and encrypt it like so:

```bash
./sops --encrypt --age <put-age-public-key-here> applications/hetzner-csi-driver/secrets.sample.yaml > applications/hetzner-csi-driver/secrets.enc.yaml
```

### Deploy Strimzi Kafka Operator via ArgoCD (_recommended_)

If you have [ArgoCD](https://argoproj.github.io/cd/) installed in your cluster, you can simply customize the `argocd-appliations.yaml` to your needs.
ArgoCD needs access to this repository if you want this to work.

```bash
export KUBECONFIG=kubeconfig.yaml
kubectl apply -f argocd-applications.yaml
```

See [this repository](https://github.com/flynnt-io/flynnt-argocd-sample) on how to install ArgoCD to your cluster.

> **Note**
>
> The [Hetzner CSI Driver](https://github.com/hetznercloud/csi-driver) is included in this deployment. If your cluster is not using compute nodes from Hetzner, replace it with the StorageClass of your choice.

### Deploy Strimzi Kafka Operator via Helm & Kustomize

#### Deploying Hetzner CSI Driver (optional)
This is optional if you already have a different CSI provider deployed.
The kustomize below automatically decrypts the secret by using ksops. This is also exactly how ArgoCD would do it.
```bash
export KUBECONFIG=kubeconfig.yaml
./kustomize build --enable-alpha-plugins --enable-exec applications/hetzner-csi-driver | kubectl apply -f -

```
#### Deploying Strimzi Kafka Operator

Next, we are ready to deploy the Strimzi Operator. We want the operator to watch all namespaces, hence we provide this as a config to the helm chart.

In future, if there is need for more config options to the strimzi operator, one should use a separate values.yaml file to store them.
```bash
export KUBECONFIG=kubeconfig.yaml
./helm upgrade -n strimzi-operator --install strimzi-cluster-operator --version 0.38.0 --set watchAnyNamespace=true --create-namespace oci://quay.io/strimzi-helm/strimzi-kafka-operator
```

Check the pods in the cluster to see if it deployed successfully
```bash
kubectl -n strimzi-operator get pod
```

#### Deploying a sample Kafka Cluster

Now we are ready to deploy a sample Kafka Cluster.
The official [strimzi repository](https://github.com/strimzi/strimzi-kafka-operator) has great samples, so we just use one of them.

```bash
export KUBECONFIG=kubeconfig.yaml
kubectl create namespace my-kafka
kubectl -n my-kafka apply -f https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/main/examples/kafka/kafka-ephemeral-single.yaml
```
The operator should now take care of provisioning the kafka cluster. 

For anything more advanced you should consult the official strimzi and kafka documentation.

Also see our other guides on how to setup your ingress and cert-manager if you want to allow http access to your Kafka cluster.


## Encryption of secrets
We use [sops](https://github.com/mozilla/sops) and [age](https://github.com/FiloSottile/age).

Editing the secrets file inline:
```bash
SOPS_AGE_KEY=<put-secret-key-here> ./sops -i secrets.enc.yaml
```
