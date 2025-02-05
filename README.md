# Production-ready LLMs on Kubernetes from scratch

Let's provision three GPU nodes and create a Kubernetes cluster from scratch, then deploy a variety of options for running LLMs:

1. Ollama and Open WebUI
  * We'll observe the issues with Ollama out of the box: highly restricted context lengths and "lobotomized" quantization
2. vLLM with Helix.ml on top
  * We'll use this to motivate the issues that lead us to bake model weights into docker images and write our own GPU scheduler!
3. TODO: Helix.ml with its GPU scheduler

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

(in practice, you'd want to configure the cluster to include the server's public IP address in the certificate, or use a DNS name e.g. with multiple A records)

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

The NVIDIA drivers and NVIDIA container runtime are fortunately already installed on these Lambdalabs machines.

To make k3s work with NVIDIA, first let's make nvidia the default container runtime on these k3s nodes ([ref](https://github.com/NVIDIA/k8s-device-plugin/issues/406#issuecomment-1852531772)):

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

You should now see NVIDIA GPUs in the output of:
```
sudo kubectl describe nodes | grep nvidia.com/gpu.count
```
e.g.
```
                    nvidia.com/gpu.count=1
```

# Deploy Open WebUI with Ollama

```
helm repo add open-webui https://helm.openwebui.com/
helm repo update
```
```
helm upgrade --install --set image.tag=v0.5.7 open-webui open-webui/open-webui
```

(Use a newer version of webui if there's one available, check on [webui github releases](https://github.com/open-webui/open-webui/releases))

Copy and paste the port-forward command after the `helm upgrade`.
To get back to this later:
```
helm status open-webui
```

Click buttons in the web UI to deploy `llama3.1:8b`

## Observe issues with context length

Paste in a large JSON response and observe that the LLM simply truncates that message!

Fix: pass `num_ctx` through ollama API (doesn't work in OpenAI compatible API)

Note this will increase GPU memory usage.

## Observe issues with quantization

Ask the LLM: What is real?
Notice that it gets confused.

Fix: use a q8_0 variant of the model weights. Note this will increase GPU memory usage.


## Observe issues with quadratic memory usage

Fix:
```
		"OLLAMA_FLASH_ATTENTION=1",
```

Also note that Ollama aggressively unloads models, which we don't want in production!

Fix:
```
		"OLLAMA_KEEP_ALIVE=-1",
```

Additionally, using a quantized kv cache helps reduce long context GPU memory usage:
```
		"OLLAMA_KV_CACHE_TYPE=q8_0",
```

# Deploy vLLM

Edit `vllm/secret.yaml` to add your huggingface token.
Ensure you accept the terms at [https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.3](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.3)

```
cd vllm
kubectl apply -f secret.yaml
kubectl apply -f pvc.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

## Deploy Helix.ml configured as a frontend to vLLM

```
helm upgrade --install keycloak oci://registry-1.docker.io/bitnamicharts/keycloak \
  --version "24.3.1" \
  --set image.tag="23.0.7" \
  --set auth.adminUser=admin \
  --set auth.adminPassword=oh-hallo-insecure-password \
  --set httpRelativePath="/auth/"
```
```
cd helix
helm repo add helix https://charts.helix.ml
helm repo update
```

```
export LATEST_RELEASE=$(curl -s https://get.helix.ml/latest.txt)
helm upgrade --install my-helix-controlplane helix/helix-controlplane \
  -f values-example.yaml \
  --set image.tag="${LATEST_RELEASE}"
```

Copy and paste the port-forward command after the `helm upgrade` and load in your browser.

To get back to this later:
```
helm status my-helix-controlplane
```

Create an account, log in, and observe nice, fast vLLM powered inference, e.g. for API integrations and RAG.
