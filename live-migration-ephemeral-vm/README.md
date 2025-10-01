# Experiment: Live Migration of an Ephemeral VM with NGINX

This experiment explores how **live migration** works on a completely **stateless (ephemeral)** Virtual Machine in an OpenShift Virtualization cluster.

The goal is to answer a practical question: is it possible to live migrate a VM without persistent disks?

## Experiment Architecture

The architecture consists of two main components within the same namespace:

1.  **A "Hello, World" Application**: A simple Pod (`hello-app`) exposing a web service on port `8080`, accessible via a `ClusterIP` Kubernetes Service.
2.  **A Reverse Proxy in a VM**:
    * A Fedora `VirtualMachine` provisioned using a **`containerDisk`**, making it completely ephemeral.
    * Inside the VM, a `cloud-init` script installs and starts **NGINX**.
    * A **ConfigMap** (`nginx-conf`) containing the proxy configuration is mounted into the VM as a virtual disk.
    * NGINX is configured to forward all incoming traffic to the `hello-app` Service.

## Main Result

**Live migration works flawlessly.**

Because the VM does not use a Persistent Volume Claim (PVC), the standard requirement for `ReadWriteMany` (RWX) shared storage does not apply. KubeVirt transfers the VM's memory state from one node to another, while the root disk is simply instantiated from the original container image on the destination node.
