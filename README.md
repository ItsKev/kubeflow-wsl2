# Kubeflow with GPU support on WSL2

## Prerequisites

### kustomize
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
sudo mv kustomize /usr/local/bin
```

### minikube
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

### GPU support
Follow the installation instructions here: https://minikube.sigs.k8s.io/docs/tutorials/nvidia/

### Start minikube
```bash
minikube start --driver docker --container-runtime docker --gpus all --memory 8192 --cpus 4 --disk-size 40g
minikube addons enable metrics-server
minikube stop
minikube start
```

### Update nvidia-device-plugin installation
```bash
kubectl delete -n kube-system daemonsets.apps nvidia-device-plugin-daemonset
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0-rc.2/nvidia-device-plugin.yml
```

### Verify the installation
```bash
kubectl apply -f samples/cuda-vectoradd.yaml
```

Output should look similar to the following:
```bash
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

## Install Kubeflow

Clone repository
```bash
git clone --depth 1 --branch v1.8.0 https://github.com/kubeflow/manifests.git
```

Install kubeflow
```bash
cd manifests
while ! kustomize build example | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done
```

Wait until all pods are ready. This takes a few minutes.

Connect to the Kubeflow Cluster:
```bash
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

Email Address: user@example.com  
Password: 12341234

## Create a notebook with GPU support

Notebooks -> New Notebook  
Name: gpu-test
GPUs: 1 - NVIDIA

Wait until the notebook is created and Connect to it.  
Open a new terminal and check if the GPU is recognized correctly by running:
```bash
nvidia-smi
```

## Build a Pipeline

Apply policies to create a pipeline with the kfp client:
```bash
kubectl apply -f samples/policies.yaml 
```

Create a new Jupyter Notebook and run the following code to create a pipeline which outputs "Hello, World!":
```python
import kfp
from kfp import compiler, dsl

@dsl.component(base_image='python:3.11')
def say_hello(name: str) -> str:
    hello_text = f'Hello, {name}!'
    print(hello_text)
    return hello_text

@dsl.pipeline
def hello_pipeline(recipient: str) -> str:
    hello_task = say_hello(name=recipient)
    return hello_task.output

compiler.Compiler().compile(hello_pipeline, 'pipeline.yaml')

client = kfp.Client()
run = client.create_run_from_pipeline_package(
    'pipeline.yaml',
    arguments={
        'recipient': 'World',
    },
)
```
