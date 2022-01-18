# Set up a local cluster

## Minikube

Documentation: https://minikube.sigs.k8s.io/docs/commands/start/

```sh
minikube start --network-plugin=cni --cni=false --kubernetes-version=1.21.6
```

## Kind

Documentation: https://kind.sigs.k8s.io/docs/user/quick-start/#creating-a-cluster

```sh
curl -LO https://raw.githubusercontent.com/cilium/cilium/1.11.0/Documentation/gettingstarted/kind-config.yaml
kind create cluster --config=kind-config.yaml
```

## Inspection

The current `kubectl` context should now be `minikube` or `kind-kind`:

```sh
kubectl config current-context
```

Allowing us to list the nodes and pods in the cluster:

```sh
kubectl get nodes
kubectl get pods -A
```

Ideally, we should see the nodes marked as `NotReady` and `coredns` pods failing to start as no CNI plugin is installed yet.

> This might not be the case if using Minikube: depending on your local environment and Minikube version, a CNI plugin might or might not have been installed automatically.
> However we can continue even if a CNI plugin has been installed and nodes/`coredns` are ready, as the installation will override it and restart unmanaged pods.

# Installing Cilium

Documentation:

- https://docs.cilium.io/en/stable/concepts/overview/
- https://docs.cilium.io/en/stable/gettingstarted/

We will start by installing the `cilium` CLI and then installing Cilium in our cluster.

## Install Cilium CLI

Source: https://github.com/cilium/cilium-cli/

The `cilium` CLI helps Kubernetes administrators interface with Cilium and manage it.

### Linux

```sh
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
```

### MacOS

```sh
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-darwin-amd64.tar.gz{,.sha256sum}
shasum -a 256 -c cilium-darwin-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-darwin-amd64.tar.gz /usr/local/bin
rm cilium-darwin-amd64.tar.gz{,.sha256sum}
```

## Basic usage

A few `cilium` CLI commands to try out after installing:

```sh
cilium version
cilium help
cilium status
cilium install --help
```

As we can see, `cilium install` offers various configuration knobs that can be leveraged to tweak Cilium to our needs.

## Install Cilium in the cluster

Install Cilium:

```sh
cilium install
```

In this example we use the default configuration: note how `cilium install` automatically chooses a default configuration based on the detected environment, and then creates various Kubernetes objects in order to install Cilium.
Once done, we then verify the installation as suggested:

```sh
cilium status
```

Note how `cilium status` is reporting information on both:

- Cilium Agents, which run on each node in the cluster (scheduled via `DaemonSet`) and are responsible for the actual operations (managing eBPF programs for pod networking).
- Cilium Operator, which is responsible for cluster-wide operations such as IPAM or `kvstore` operations (scheduled via `Deployment`).

If we want to take a look under the hood:

```sh
kubectl get nodes
kubectl get pods -A
kubectl get cm cilium-config -o yaml -n kube-system
```

- The nodes should now be marked as `Ready` and `coredns` pods running.
- The Cilium pods as shown in `cilium status` have been added to the `kube-system` namespace.
- The Cilium configuration is stored in a ConfigMap.

Cilium is now properly installed and manages connectivity within the cluster.

We can verify connectivity by running a connectivity test:

```sh
cilium connectivity test
```

This will deploy test pods in the `cilium-test` namespace and go through our CLI's test suite.
The test suite is designed as a complementary check to `cilium status`, verifying that Cilium is correctly handling connectivity in various scenarios (e.g. with and without network policies).
It's explicitly intended to be run on any cluster any time you want to check Cilium is working properly: it auto-detects the environment/configuration and automatically chooses which tests to run.

Since the test takes a little while to run, an additional information: Cilium can be installed via the CLI as we just did, or via Helm if better suited to your use case (see our documentation: https://docs.cilium.io/en/stable/gettingstarted/k8s-install-helm/).

No matter how Cilium was installed (CLI or Helm), you can always use the Cilium CLI to interact with Cilium: we definitely recommend you to run `cilium connectivity test` at least once after any Cilium install.

Once done with the connectivity test, clean up the test namespace:

```sh
kubectl delete ns cilium-test --wait=false
```

# Next steps

Now that Cilium is up and running, we can finally explore its capabilities.
It's up to you to tell us what you'd like to see :)

Depending on your questions, we might improvise or go through some of our prepared resources:

- [Network Policies and Cilium Network Policies](./netpol)
- [Observability with Hubble and Hubble UI](./hubble)

If you don't what to start with, a good idea might be to give us some context around what you're doing and the challenges you're trying to tackle!

# Conclusion

Thank you for attending our Installfest :)
We encourage you to keep playing with Cilium, and invite you to join our community Slack at https://cilium.herokuapp.com/ for any future question you may have :)

# Clean up

## Minikube

Documentation: https://minikube.sigs.k8s.io/docs/commands/delete/

```sh
minikube delete
```

## Kind

Documentation: https://kind.sigs.k8s.io/docs/user/quick-start/#deleting-a-cluster

```sh
kind delete cluster
```

## Uninstalling CLIs

If you want to uninstall the `cilium` and `hubble` CLIs installed during the Installfest, run:

```sh
sudo rm /usr/local/bin/{cilium,hubble}
```
