
# Gadgetron in Kubernetes

This repository contains a setup for running a distributed deployment of the [Gadgetron](https://github.com/gadgetron/gadgetron) in a Kubernetes cluster. 

The setup uses [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) to adjust the number of Gadgetron instances (pods) running in the cluster in response to gadgetron activity and it relies on [cluster-autoscaling](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) to adjust the number of nodes. Specifically, an increase reconstruction activity will lead to the deployment of more Gadgetron instances and when the resources on existing nodes are exhausted more will be added. Idle nodes will be removed from the cluster after some idle time. 

Shared files (dependencies and exported data) are stored in persistent volumes, which could be backed by [Azure Files](https://azure.microsoft.com/en-us/services/storage/files/).

The Gadgetron used a script to discover remote worker nodes. The script is specified in the `GADGETRON_REMOTE_WORKER_COMMAND` environment variable, which references a script added in a ConfigMap. The is also a PreStop lifecycle hook script, which is used to ensure that Gadgetron instances with active connections are not abruptly disconnected. 

## Deployment Instructions

1. Set up a Kubernets cluster. Please see instructions [Azure Kubernetes Service (AKS)](aks-setup.md) or [Minikube](minikube-setup.md).

1. Deploy [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator) to allow metrics collection from the Gadgetron:

    ```bash
    helm upgrade --install prometheus stable/prometheus-operator \
        --namespace monitoring \
        --create-namespace \
        --set commonLabels.prometheus=monitor \
        --set prometheus.prometheusSpec.serviceMonitorSelector.matchLabels.prometheus=monitor
    ```

    This will install the operator, Prometheus server, Grafana, etc. 

1. Deploy the [Prometheus Adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter):

    ```bash
    helm install --namespace monitoring prometheus-adapter stable/prometheus-adapter -f custom-metrics/custom-metrics.yaml
    ```

    The Prometheus Adapter is responsible for aggregating metrics from Promtheus and exposing them as custom metrics that we can use for scaling the Gadgetron. 

1. Deploy gadgetron in AKS with helm chart:

    ```bash
    helm install <nameofgadgetroninstance> helm/gadgetron/
    ```

    To select a specific storage class (e.g. Azure Files) for dependencies, etc.:

    ```bash
    helm install <nameofgadgetroninstance> helm/gadgetron/ --set storage.storageClass=azurefile
    ```

    To use a specific node pool:

    ```bash
    helm install --set nodeSelector.agentpool=<nodepoolname> <nameofgadgetroninstance> helm/gadgetron/
    ```

1. Check that metrics are flowing. After deploying the Gadgetron, it should start emitting metrics and they should be exposed as custom metrics. You can check that you can read them with:

    ```bash
    kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/gadgetron_activity | jq .
    ```

## Connecting with port forwarding

Once the Gadgetron deployment is live, you can find the cluster ip address for the Gadgetron with something like:

```bash
kubectl get svc
```

And you can easily open a tunnel from your desktop to the gadgetron with something like:

```bash
kubectl port-forward svc/<helm release>-gadgetron 9002:9002
```

And then connect directly to `localhost:9002`.

## Connecting with VPN (Azure)

To connect secure to the Gadgetron in the Kubernetes cluster, it is recommended that you establish a VPN point to site connection. Please consult the [Azure P2S VPN guide](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-resource-manager-portal). The basic steps are:

1. Create a gateway subnet in your [AKS cluster network](https://docs.microsoft.com/azure/aks/concepts-network).
1. Create a VPN Gateway in the subnet.
1. Obtain/generate keys and [install client software](https://docs.microsoft.com/en-us/azure/vpn-gateway/point-to-site-vpn-client-configuration-azure-cert).
1. Connect securely using VPN connection.


## Connecting with SSH jump server

This repo contains a helm chart and other artifacts for deploying an SSH jump server in the cluster and you can use this jump server to establish an SSH tunnel. Maintaining these tunnels can be cumbersome and VPN is the recommended approach. That said, an SSH jump server can provide a fast way to test the deployment. 

### Deploy SSH jump server:

> Approach adopted from [https://github.com/kubernetes-contrib/jumpserver](https://github.com/kubernetes-contrib/jumpserver).

First deploy the ssh key:

```bash
#Get an SSH key, here we are using the one for the current user
SSHKEY=$(cat ~/.ssh/id_rsa.pub |base64 -w 0)
sed "s/PUBLIC_KEY/$SSHKEY/" gadgetron-ssh-secret.yaml.tmpl > gadgetron-ssh-secret.yaml

#Create a secret with the key
kubectl create -f gadgetron-ssh-secret.yaml
```

This will install the key in a kubernetes secret called `sshkey`, you can edit `gadgetron-ssh-secret.yaml` to give the key a different name. 

Then deploy the jump server:

```bash
helm install sshjump helm/sshjump
```

Or with a custom ssh key secret:

```bash
helm install --set sshKeySecret=alternative-ssh-secret-name sshjump2 helm/sshjump/
```

### Connecting with SSH to the jump server

The jump sever enables the "standard" Gadgetron connection paradigm through an SSH tunnel. The Gadgetron instances themselves are not directly accessible. Discover the relvant IPs and open a tunnel with:

```bash
#Public (external) IP:
EXTERNALIP=$(kubectl get svc <sshd-jumpserver-svc> --output=json | jq -r .status.loadBalancer.ingress[0].ip)

#Internal (cluster) IP:
GTCLUSTERIP=$(kubectl get svc <gadgetron-frontend> --output=json | jq -r .spec.clusterIP)

#Open tunnel:
ssh -L 9022:${GTCLUSTERIP}:9002 root@${EXTERNALIP}
```

# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.