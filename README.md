Building a Kubernetes cluster with Vagrant
===

This Vagrantfile allows building a Kubernetes cluster with a configurable number of workers. The cluster is configured using `kubeadm`.

Required Vagrant plugins
===

Required Vagrant plugins can be installed with `vagrant plugin install`. **If you are behind proxies that perform TLS interception**, you may need to add the required CA certificates to Vagrant's certificate store. This can usually be found at `/opt/vagrant/embedded/cacert.pem`, but may vary depending on your OS and Vagrant installation method.

Required plugins
---
* **vagrant-hostmanager**
* **vagrant-proxyconf**

Quick start
===

Building without any proxies
---
1. Just run `vagrant up`.

Building with proxies
---
1. Define `http_proxy`, `https_proxy`, and `no_proxy` in your environment. These environment variables will automatically be pulled into the VMs and used by **apt** and **Docker**.
2. **If your proxy servers perform man-in-the-middle against TLS connections**, create a directory on your build host, place the necessary CA certificates inside, and define `cacerts_dir` in your environment to be the full path to this directory. Any certificates found in this directory will be added to the system CA certificate chain.
3. Run `vagrant up`.

Building without Internet access
---
1. You will need to have access to:
  * a Docker registry that has mirrored all necessary containers to build a Kubernetes cluster
  * a mirror of Canonical's Xenial package repository
  * a mirror of apt.kubernetes.io
2. In the `custom` directory:
  * create `sources.list`, containing the apt paths to use for Xenial packages
  * create `sources.list.kubernetes`, containing the apt paths to use for Kubernetes packages (**NOTE**: this repo must have the GPG key at `doc/apt-key.gpg`)
  * create `kube-flannel.yml` (this can be a straightforward copy of the upstream manifest - URL is in the Vagrantfile)
  * create `kube-flannel-rbac.yml` (above also applies here)
  * create `calico.yml` if you want to use Calico as the network provider (above also applies here)
3. If your internal resources have TLS certificates that are not signed by a standard root CA, create a directory on your build host, place the necessary CA certificates inside, and define `cacerts_dir` in your environment to be the full path to this directory. Any certificates found in this directory will be added to the system CA certificate chain.
4. Run `kubernetes_version=X.Y.Z vagrant up`, where **X.Y.Z** is the desired version of Kubernetes that you want to install. **NOTE** that this version number will be used for both Kubernetes packages and containers. Specifying the version number is required in order to stop **kubeadm** from reaching out to the Internet to query for the latest version.

Files
===

The build process will look for files with these names in the `custom` directory in this repository. If found, they will be used during the build process.

* `calico.yml`: override Calico deployment manifest
* `kube-flannel.yml`: override Flannel deployment manifest
* `kube-flannel-rbac.yml`: override Flannel RBAC deployment manifest
* `sources.list`: override default system apt sources
* `sources.list.kubernetes`: override default kubernetes apt repo

If the directory `custom/extra_yaml` is present, this directory will be passed to `kubectl apply -f` once the master becomes ready. This can be used to deploy arbitrary manifests to the cluster.

After the cluster is built the following files will be created:

`kube.config`: local Kubernetes config generated during `kubeadm init` (can be used like `kubectl --kubeconfig kube.config get pods`, or `export KUBECONFIG=/path/to/kube.config`)

Environment variables
===

The following environment variables can be configured to control the disposition of the cluster:

Proxy variables
---
* `http_proxy`: configures `http_proxy` inside cluster VMs
* `https_proxy`: configures `https_proxy` inside cluster VMs
* `no_proxy`: configures `no_proxy` inside cluster VMs. If proxies are being configured, the master IP will be appended to `no_proxy` so that `kubeadm` functions as expected.
* `cacerts_dir`: if your SSL proxies perform MITM on SSL connections using custom root CA certificates, you may provide a path to a directory on the build host that contains additional CA certificates to add to the VMs. All certificates should end in `.crt` and each file should only contain one certificate.

Cluster sizing variables
---
* `worker_count`: configures the number of worker VMs that will be built (default **3**).
* `cpu_count`: configures the number of CPUs each VM will have (default **1**).
* `memory_mb`: configures the amount of memory in MB each VM will have (default **1024**).
* `disk_size`: configures the size of the additional volumes that will be created in the VM (default **20480**).

Kubernetes variables
---
* `enable_podpreset_admission_controller`: enable the PodPreset admission controller so that PodPreset resources can be injected into pods (https://kubernetes.io/docs/tasks/inject-data-application/podpreset/)
  * **NOTE**: Kubernetes will still accept PodPreset resources even without this feature enabled, but they will not be injected into the pods
* `kube_version`: version number of Kubernetes to install (currently used both for apt packages and container tags)
* `network_provider`: pod networking provider to use, options are **calico** or **flannel** (default is **flannel**)
* `repo_prefix`: prefix used for container names within the Kubernetes cluster (if you want to pull from a local registry rather than the Internet)
* `skip_preflight_checks`: skip preflight checks if this variable exists (can be set to anything)
