# Production-ready LLMs on Kubernetes from scratch

Let's provision three GPU nodes and create a Kubernetes cluster from scratch, then deploy a variety of options for running LLMs:

1. Ollama and Open WebUI
  * We'll observe the issues with Ollama out of the box: highly restricted context lengths and "lobotomized" quantization
2. vLLM with Helix.ml on top
  * We'll use this to motivate the issues that lead us to bake model weights into docker images and write our own GPU scheduler!
3. Helix.ml with its GPU scheduler

## Deploying the Kubernetes cluster

We'll use Lambdalabs where we can get 3xA100s for $3.87/hour.

Sign up at [lambdalabs.com](https://lambdalabs.com/) and provision three 1xA100 nodes.

SSH into them in three separate terminals/tmux panes and run.

### On first node
```
curl -sfL https://get.k3s.io | sh -
```
```
sudo cat /var/lib/rancher/k3s/server/node-token
```

### On second and third nodes

```
curl -sfL https://get.k3s.io | K3S_URL=https://ip-of-first-node:6443 K3S_TOKEN=mynodetoken sh -
```
replacing `ip-of-first-node` with the IP address of the first node (from lambdalabs UI) and `mynodetoken` with the value of `node-token` from the first node.

### Extract a `kubeconfig`

Run on the first node, and copy to a file on your local computer (e.g. a file named `kubeconfig` which you can place in the root of this git repo)
```
sudo cat /etc/rancher/k3s/k3s.yaml
```
Replace `127.0.0.1` with the IP of the first server.

Edit the `kubeconfig` and add the following line to the cluster:
```
    insecure-skip-tls-verify: true
```

(in practice, you'd want to configure the cluster to include the server's public IP address in the certificate)

For example:
```
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://129.146.62.66:6443
  name: default
[...]
```
Alternatively you can also run
```
kubectl config set-cluster default --insecure-skip-tls-verify=true
```

Now you should be able to run, from your local machine:
```
export KUBECONFIG=$(pwd)/kubeconfig
```
```
kubectl get nodes
```

Expected output:
```
NAME            STATUS   ROLES                  AGE    VERSION
129-146-62-66   Ready    control-plane,master   2d9h   v1.31.5+k3s1
144-24-39-60    Ready    <none>                 2d9h   v1.31.5+k3s1
144-24-52-195   Ready    <none>                 2d9h   v1.31.5+k3s1
```

### Configure NVIDIA device plugin on these nodes

First let's make nvidia the default container runtime on these k3s nodes:

```
sudo cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
```

Edit `/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl` to add 'default_runtime_name = "nvidia"' as below:

```
[plugins."io.containerd.grpc.v1.cri".containerd]
default_runtime_name = "nvidia"
```

On the first node:
```
sudo systemctl restart k3s
```
On the other nodes:
```
sudo systemctl restart k3s-agent
```

# Deploy Open WebUI with Ollama

```
helm upgrade --install --set image.tag=v0.5.7 open-webui open-webui/open-webui
```

(Use a newer version of webui if there's one available)


