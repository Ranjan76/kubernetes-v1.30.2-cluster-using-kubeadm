# ✅ Kubernetes Installation – Pre-Requisites & Setup Plan

This document outlines the prerequisites and system setup plan for installing Kubernetes using **MobaXterm** as the terminal interface.

---

## 🧪 Testing Environment (For Basic Setup & Learning)

**System Configuration:**

* **CPU:** 2 vCores
* **RAM:** 4 GB
* **Disk Volume:** 16 GB

**Purpose:**

* Practice Kubernetes installation
* Run basic Pods and Services
* Validate cluster setup and network

> ⚠️ *Note: This setup is only suitable for basic learning and experimentation. Not recommended for production or real workloads.*

---

## 🚀 Production Environment (Minimum Viable Setup)

**System Configuration:**

* **CPU:** 2 vCores (minimum)
* **RAM:** 4 GB
* **Disk Volume:** 25–30 GB

**Purpose:**

* Deploy and manage real Kubernetes clusters
* Host microservices and applications
* Perform basic CI/CD integration and scaling tests

> ℹ️ *This configuration is a starting point for small-scale production. For more demanding workloads, consider scaling resources accordingly.*

---

## ✅ Tools & Terminal Access

* **MobaXterm** – Used for SSH access to nodes and executing cluster commands
* **kubectl** – Kubernetes command-line tool (to be installed post-setup)
* **Container Runtime** – Docker / containerd
* **Operating System:** Ubuntu 20.04 or higher (preferred)

---

## 🛠️ Installation Steps Overview

### Step 1: Disable Swap on All Nodes

Kubernetes requires that swap is disabled on all nodes to ensure that memory is managed exclusively by the operating system and Kubernetes.

**Run the following commands to disable swap:**

```bash
# Disable swap temporarily
sudo swapoff -a

# Comment out swap entry in /etc/fstab to prevent it from being enabled after a reboot
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

* `swapoff -a`: Disables swap memory on the system.
* `sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab`: Comments out the swap entry in `/etc/fstab` to ensure it is not re-enabled on reboot.

---

### Step 2: Enable IPv4 Packet Forwarding

Kubernetes requires IPv4 packet forwarding to allow the communication between containers across different nodes.

**Run the following commands to enable IPv4 packet forwarding:**

```bash
# Enable IPv4 packet forwarding
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

* Creates a configuration file `/etc/sysctl.d/k8s.conf` for IPv4 forwarding.
* `sudo sysctl --system`: Applies all sysctl settings from configuration files.

---

### Step 3: Verify IPv4 Packet Forwarding

Run the following command to verify IPv4 packet forwarding:

```bash
sysctl net.ipv4.ip_forward
```

* This command should return `net.ipv4.ip_forward = 1`, indicating that IPv4 packet forwarding is enabled.

---

### Step 4: Install containerd

Install `containerd` as the container runtime for Kubernetes.

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt-get update && sudo apt-get install -y containerd.io

# Enable and start containerd
sudo systemctl enable --now containerd
```

---

### Step 5: Install CNI Plugin

Kubernetes uses a CNI plugin to manage container network interfaces.

```bash
# Download the CNI plugin
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz

# Create the directory for CNI plugins
sudo mkdir -p /opt/cni/bin

# Extract and install the CNI plugins
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.0.tgz
```

---

### Step 6: Forward IPv4 and Configure iptables

Ensure the required kernel modules and sysctl parameters are set correctly for Kubernetes networking.

```bash
# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl parameters required by Kubernetes
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

# Apply sysctl settings
sudo sysctl --system

# Verify applied settings
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

# Reload sysctl config for good measure
sudo modprobe br_netfilter
sudo sysctl -p /etc/sysctl.conf
```

---

### Step 7: Modify containerd Configuration for systemd Support

```bash
# Edit the containerd configuration file using vim
sudo vim /etc/containerd/config.toml
```
Here's the full `containerd` configuration file that you can copy easily:

```bash
# Paste the configuration in the file and save it

disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
version = 2

[cgroup]
  path = ""

[debug]
  address = ""
  format = ""
  gid = 0
  level = ""
  uid = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216
  tcp_address = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0

[metrics]
  address = ""
  grpc_histogram = false

[plugins]

  [plugins."io.containerd.gc.v1.scheduler"]
    deletion_threshold = 0
    mutation_threshold = 100
    pause_threshold = 0.02
    schedule_delay = "0s"
    startup_delay = "100ms"

  [plugins."io.containerd.grpc.v1.cri"]
    disable_apparmor = false
    disable_cgroup = false
    disable_hugetlb_controller = true
    disable_proc_mount = false
    disable_tcp_service = true
    enable_selinux = false
    enable_tls_streaming = false
    ignore_image_defined_volumes = false
    max_concurrent_downloads = 3
    max_container_log_line_size = 16384
    netns_mounts_under_state_dir = false
    restrict_oom_score_adj = false
    sandbox_image = "k8s.gcr.io/pause:3.5"
    selinux_category_range = 1024
    stats_collect_period = 10
    stream_idle_timeout = "4h0m0s"
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    systemd_cgroup = false
    tolerate_missing_hugetlb_controller = true
    unset_seccomp_profile = ""

    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = ""
      max_conf_num = 1

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      disable_snapshot_annotations = true
      discard_unpacked_layers = false
      no_pivot = false
      snapshotter = "overlayfs"

      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        base_runtime_spec = ""
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          base_runtime_spec = ""
          container_annotations = []
          pod_annotations = []
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = true

      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        base_runtime_spec = ""
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime.options]

    [plugins."io.containerd.grpc.v1.cri".image_decryption]
      key_model = "node"

    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = ""

      [plugins."io.containerd.grpc.v1.cri".registry.auths]

      [plugins."io.containerd.grpc.v1.cri".registry.configs]

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""

  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"

  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"

  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"

  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false

  [plugins."io.containerd.runtime.v1.linux"]
    no_shim = false
    runtime = "runc"
    runtime_root = ""
    shim = "containerd-shim"
    shim_debug = false

  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]

  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]

  [plugins."io.containerd.snapshotter.v1.aufs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.btrfs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.devmapper"]
    async_remove = false
    base_image_size = ""
    pool_name = ""
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.native"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.overlayfs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.zfs"]
    root_path = ""

[proxy_plugins]

[stream_processors]

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar"

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar+gzip"

[timeouts]
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"

[ttrpc]
  address = ""
  gid = 0
  uid = 0
```

---

## Step 8: Restart containerd and Check the Status

After modifying the containerd configuration, restart the service and verify it's running correctly:

```bash
sudo systemctl restart containerd && systemctl status containerd
```

✅ If everything is configured properly, you should see output indicating that `containerd` is **active (running)**.

---

## Step 9: Install kubeadm, kubelet, and kubectl

Install the Kubernetes tools `kubeadm`, `kubelet`, and `kubectl` by following these steps:

```bash
# Update the package list
sudo apt-get update

# Install dependencies for apt over HTTPS
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Create a directory to store the GPG key
sudo mkdir -p -m 755 /etc/apt/keyrings

# Download the Kubernetes GPG key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update apt sources again to include Kubernetes repository
sudo apt-get update -y

# Install kubelet, kubeadm, and kubectl
sudo apt-get install -y kubelet kubeadm kubectl

# Prevent updates to kubelet, kubeadm, and kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Explanation:

* These commands add the Kubernetes GPG key and repository, then install `kubelet`, `kubeadm`, and `kubectl`.
* The `apt-mark hold` command ensures that these tools are not upgraded automatically when running `apt-get upgrade`.

---

### **Step 10: Initialize the Cluster and Apply CNI**

This step covers initializing your Kubernetes cluster and applying the CNI (Weave CNI in this guide).

---

#### **1. Pull the Necessary Container Images for Kubernetes**

```bash
# Pull Kubernetes images required for the cluster setup
sudo kubeadm config images pull
```

---

#### **2. Initialize the Kubernetes Master Node**

```bash
# Initialize the master node for Kubernetes cluster
sudo kubeadm init
```

---

#### **3. Set up the Kubernetes Config for User Access**

To start using your cluster, set up the kubeconfig file for `kubectl` access.

* **For a Regular User:**

```bash
# Create the .kube directory for storing Kubernetes config
mkdir -p $HOME/.kube

# Copy the Kubernetes admin config to the user's kube directory
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# Change the ownership of the config file to the user
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

* **For the Root User:**

If you’re the root user, you can set the `KUBECONFIG` environment variable directly:

```bash
# Set the KUBECONFIG environment variable for root user
export KUBECONFIG=/etc/kubernetes/admin.conf
```

---

#### **4. Apply the CNI YAML File (Weave CNI)**

```bash
# Apply the CNI configuration (Weave CNI) to set up networking
kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.30/net.yaml
```

---

Here's a revised version of Step 11 with a more streamlined format:

---

**Step 11: Join Worker Nodes to the Cluster**

After initializing the Kubernetes master node, you will need to run the `kubeadm join` command on each worker node to add them to the cluster.

1. **Run the Command on Each Worker Node**
   You will receive a command after initializing the master node to join the worker nodes to the cluster. The command will look like this:

   ```bash
   kubeadm join 192.168.122.100:6443 --token zcijug.ye3vrct74itrkesp \
        --discovery-token-ca-cert-hash sha256:e9dd1a0638a5a1aa1850c16f4c9eeaa2e58d03f97fd0403f587c69502570c9cd
   ```

   **Note:**

   * The `kubeadm join` command will contain a unique token and certificate hash generated during the master node initialization.
   * Ensure you use the command exactly as shown when it is generated to avoid errors.
   * If you encounter any syntax errors or issues, please ensure the command is copied exactly and that the token and certificate hash are valid.

---

#### **2. Verify Worker Node Join Status**

After running the command on the worker node(s), go to the master node and run the following command to verify that the worker nodes have successfully joined:

```bash
kubectl get nodes
```

You should see the worker node(s) listed as **Ready**.

---









