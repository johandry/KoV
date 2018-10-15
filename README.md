# Kubernetes on Vagrant

This is a simple Vagrantfile, and just a Vagrant file, to create a Kubernetes cluster on VirtualBox using Vagrant. It was inspired from the [Kubernetes Fundamentals](https://www.cncf.io/certification/training/) training from [The Linux Foundation](https://www.cncf.io).

Some of the specifications are:

- It is just one Vagrantfile
- Creates a 1xN cluster (1 Master, multiple Workers)
- Uses Ubuntu for the nodes OS
- Installs Kubernetes 1.11 and Docker 17.12.1-ce
- The Kubernetes configuration is done with Kubeadm
- Uses Canal for Pod Networking

The difference between other Kubernetes on Vagrant projects is that it's just one Vagrantfile, there is no other scripts or manifest files to download. All the code is included inside the Vagrantfile using shell script and Ruby. Also, it's a simple cluster, nothing complex, just what you need to learn Kubernetes or to use it for quick developments.

## Dependencies

- **VirtualBox**:

  Download it from [here](https://www.virtualbox.org/wiki/Downloads) or execute `brew cask install virtualbox virtualbox-extension-pack`

- **Vagrant**:

  Download it from [here](https://www.vagrantup.com/downloads.html) or execute `brew cask install vagrant`

- **Kubectl**:

  Download it from [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/) or execute `brew install kubernetes-cli`. Highly recommended to [enable shell autocompletion](https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion).

## Quick Start

To get a 1x3 cluster with nginx:

```bash
export WORKERS=3
vagrant up
export KUBECONFIG=${PWD}/shared_folder/remote_config
kubectl get nodes -w
watch -n 0.5 kubectl get pods --all-namespaces

kubectl run nginx --replicas=2 --image nginx --port=80
kubectl expose deployment nginx --type=NodePort

# Option 1:
vagrant reload worker-01
vagrant status
port=$(vagrant status | grep nginx: | cut -f3 -d' ')
curl http://localhost:${port}

# Option 2:
port=$(kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')
PORTS="8080>${port}" vagrant reload worker-01
curl http://localhost:8080

vagrant destroy

rm shared_folder/*
rm *.log
rm -rf .vagrant/

vagrant box remove ubuntu/bionic64 --all
```

The **option #1**, will forward **all** the identified ports of services of type nodePort. Use this option to not worry about having a service not accessible from outside the cluster.

The **option #2**, will forward only the ports specified in the variable `PORTS` but also using specified the host port. Use this option to get always the same port number.

## Create a Cluster

To get a 1x1 cluster, just execute:

```bash
vagrant up
```

To get a 1xN cluster use the variable `WORKERS`. For example, to get a 1x3 cluster execute:

```bash
export WORKERS=3
vagrant up
```

Export the KUBECONFIG variable and verify the cluster is running, executing:

```bash
export KUBECONFIG=${PWD}/shared_folder/remote_config
# Or:
cp ./shared_folder/remote_config ~/.kube/config

kubectl get nodes -w
watch -n 0.5 kubectl get pods --all-namespaces
```

If `watch` is not installed, use the flag `-w` of `kubectl`.

## Services

Vagrant or Virtualbox is not supported yet by LoadBalancer services type. So, all the services has to be of type [**NodePort**](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) if you want to access them from the host machine, your computer.

After adding all the services, reload the Vagrant configuration to add the forward port rules to the service ports. There is no need to reload all the machines, just the `worker-01`:

```bash
vagrant reload worker-01
```

This will identify all the services of type **NodePort**, get the exposed port and forward a host port to the service port. The host ports will be `3000 + (guest_port - 30000)`. For example, if service nginx expose port `30001`, the host port is `3001`. So, the service is accessible with `http://localhost:3001`

To define your own host port, use the environment variable **`PORTS`**, for example to access the service using port `8080`:

```bash
service_name=nginx

port=$(kubectl get svc ${service_name} -o jsonpath='{.spec.ports[0].nodePort}')
PORTS="8080>${port}" vagrant reload worker-01
curl http://localhost:8080
```

The environment variable `PORTS` has the following format: `"[host_port>]guest_port[:[host_port>guest_port]]"`, for example: `"8080>30001:8090>30123:32345"`

If the host_port is not specified, will use `3000 + (guest_port - 30000)` for example:

- `"8080>30001:8090>30123"` :
  - forward​  `localhost:8080` to `worker:30001`
  - forward ​ `localhost:8090` to `worker:30123`

- `"8080>30456:30123"` :
  - forward​  `localhost:8080` to `worker:30456`
  - forward​  `localhost:3123` to `worker:30123`

If you add more services, or execute the reload again, make sure to include the previous ports definition, otherwise will be deleted. Or, don't use the `PORTS` variable and let Vagrant get the host ports.

If you get an error about ports collition that does not allow you to forward the ports, then reload again using the `PORTS` variables and a different host port.

## Destroy the Cluster

To destroy the cluster, execute:

```bash
vagrant destroy
```

To cleanup everything:

```bash
rm shared_folder/*
rm *.log
rm -rf .vagrant/
vagrant box remove ubuntu/bionic64 --all
```

## TODO

- [x] Include **Canal** into the Vagrantfile
- [x] Make services accessible from outside the cluster
- [x] Add more workers to the Vagrantfile
- [x] Calculate host ports without variable `PORTS`
- [ ] Use other OS, like CoreOS, for the cluster nodes
