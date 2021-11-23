# Set up Minikube cluster

Documentation: https://minikube.sigs.k8s.io/docs/commands/start/

Run the following command to set up your local Minikube cluster:

```sh
minikube start --network-plugin=cni --cni=false --kubernetes-version=1.21.6
```

We can check that the cluster has properly started by verifying the current `kubectl` context is `minikube`:

```sh
kubectl config current-context
```

And listing the nodes and pods in the cluster:

```sh
kubectl get nodes
kubectl get pods -A
```

Depending on your local environment and Minikube version, a CNI might or might not have been installed automatically by Minikube.
Ideally, no CNI has been installed and you will see the Minikube node marked as `NotReady` and `coredns` pod failing to start.
However we can continue even if a CNI has been installed and node/`coredns` are ready, as the installation will override the CNI and restart unmanaged pods.

# Cilium

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

## Install Cilium in the Minikube cluster

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

- The Minikube node should be correctly marked as `Ready`.
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

# Network Policies

Resources:

- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://networkpolicy.io/
- https://networkpolicy.io/editor

One of the most basic CNI functions is the ability to enforce network policies and implement an in-cluster zero-trust container strategy.
Network policies are a default Kubernetes object for controlling network traffic, but a CNI such as Cilium is required to enforce them.
Let's deploy a very simple application to demonstrate how it works.

Inspect [`simple-app.yaml`](./simple-app.yaml).
The application consists of two client deployments (`frontend` and `not-frontend`) and one backend deployment (`backend`).
We are going to send requests from the `frontend` and `not-frontend` pods to the `backend` pod.

Deploy the app:

```sh
kubectl create -f simple-app.yaml
kubectl get all
```

In Kubernetes, all traffic is allowed by default. Check connectivity between pods:

```sh
FRONTEND=$(kubectl get pods -l app=frontend -o jsonpath='{.items[0].metadata.name}')
NOT_FRONTEND=$(kubectl get pods -l app=not-frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -ti ${FRONTEND} -- curl -I --connect-timeout 5 backend:8080 | head -1
kubectl exec -ti ${NOT_FRONTEND} -- curl -I --connect-timeout 5 backend:8080 | head -1
```

Let's disallow traffic by applying a network policy.
Inspect [`backend-ingress-deny.yaml`](./backend-ingress-deny.yaml).
The policy will deny all ingress traffic as it is of type `Ingress` but specifies no allow rule, and will be applied to all pods with the `app=backend` label thanks to the `podSelector`.

Apply the network policy:

```
kubectl create -f backend-ingress-deny.yaml
kubectl get netpol
```

And check that we traffic is now denied:

```sh
kubectl exec -ti ${FRONTEND} -- curl -I --connect-timeout 5 backend:8080 | head -1
kubectl exec -ti ${NOT_FRONTEND} -- curl -I --connect-timeout 5 backend:8080 | head -1
```

The network policy correctly switched the default ingress behavior from default allow to default deny.
Let's now selectively re-allow traffic again, but only from frontend to backend.

We can do it by crafting a new network policy manually, but we can also use the Network Policy Editor to help us out:

- Go to https://networkpolicy.io/editor.
- Upload our initial `backend-ingress-deny` policy.
- Rename the network policy to `backend-ingress-allow-frontend` (using the **Edit** button in the center).
- On the ingress side **In Namespace** on the left, add a **pod selector** rule with expression `app=frontend`.
- Inspect the ingress flow colors: the policy will deny all ingress traffic to pods labelled `app=backend`, except for traffic coming from pods labelled `app=frontend`.
- Download the policy YAML file.

Apply the new policy and check that connectivity has been restored, but only from the frontend:

```sh
kubectl create -f backend-ingress-allow-frontend.yaml
kubectl exec -ti ${FRONTEND} -- curl -I --connect-timeout 5 backend:8080 | head -1
kubectl exec -ti ${NOT_FRONTEND} -- curl -I --connect-timeout 5 backend:8080 | head -1
```

Note that this is working despite the fact we did not delete the previous `backend-ingress-deny` policy:

```sh
kubectl get netpol
```

Network policies are additive.
Just like with firewalls, it is thus a good idea to have default `DENY` policies and then add more specific `ALLOW` policies as needed.

# Cilium Network Policies

Documentation:

- https://docs.cilium.io/en/stable/policy/
- https://docs.cilium.io/en/stable/concepts/kubernetes/policy/

On top of the default Kubernetes network policies, Cilium provides extended policy enforcement capabilities (such as Identity-aware, HTTP-aware and DNS-aware) via Cilium Network Policies.

Let's say we want to forbid our `backend` pods from reaching anything except FQDN `kubernetes.io`.
Again, in Kubernetes, all traffic is allowed by default, and since we did not apply any `Egress` network policy for now, the `backend` is able to reach out anywhere:

```sh
BACKEND=$(kubectl get pods -l app=backend -o jsonpath='{.items[0].metadata.name}')
kubectl exec -ti ${BACKEND} -- curl -Ik --connect-timeout 5 https://kubernetes.io | head -1
kubectl exec -ti ${BACKEND} -- curl -Ik --connect-timeout 5 https://cilium.io | head -1
```

Inspect [`backend-egress-allow-fqdn.yaml`](./backend-egress-allow-fqdn.yaml).
The policy will deny all egress traffic from pods labelled `app=backend` except when traffic is destined for `kubernetes.io` or is a DNS request (necessary for resolving `kubernetes.io` from `coredns`).

Apply the network policy:

```
kubectl create -f backend-egress-allow-fqdn.yaml
kubectl get cnp
```

Note the usage of `cnp` (standing for `CiliumNetworkPolicy`) instead of the default `netpol` since we are using custom Cilium resources.

And check that the traffic is now only authorized when destined for `kubernetes.io`:

```sh
kubectl exec -ti ${BACKEND} -- curl -Ik --connect-timeout 5 https://kubernetes.io | head -1
kubectl exec -ti ${BACKEND} -- curl -Ik --connect-timeout 5 https://cilium.io | head -1
```

With the ingress and egress policies in place on `app=backend` pods, we have implemented a very simple zero-trust model to all traffic to and from our backend.
In a real world scenario, cluster administrators may leverage network policies and overlay them at all levels and for all kinds of traffic in order to switch from the Kubernetes default of all traffic being allowed to only specific traffic being allowed.

# Hubble

Documentation:

- https://docs.cilium.io/en/stable/gettingstarted/hubble_setup/
- https://docs.cilium.io/en/stable/gettingstarted/hubble_cli/
- https://docs.cilium.io/en/stable/gettingstarted/hubble/

Hubble offers deep observability capabilities thanks to eBPF, but by default is only exposed locally via gRPC from the Cilium Agents.
For convenience, Cilium offers optional components to allow accessing Hubble data in a user-friendly way:

- Hubble Relay: provides cluster-wide observability via the Hubble CLI.
- Hubble UI: user-friendly web UI (requires Hubble Relay).

## Install the Hubble CLI

Source: https://github.com/cilium/hubble/

Akin to the `cilium` CLI with Cilium, the `hubble` CLI interfaces with Hubble and allows observing network traffic within Kubernetes.

### Linux

```sh
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz{,.sha256sum}
```

### MacOS

```sh
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-darwin-amd64.tar.gz{,.sha256sum}
shasum -a 256 -c hubble-darwin-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-darwin-amd64.tar.gz /usr/local/bin
rm hubble-darwin-amd64.tar.gz{,.sha256sum}
```

## Basic usage

A few `hubble` CLI commands to try out after installing:

```sh
hubble version
hubble help
hubble observe --help
```

## Enable Hubble in Cilium

We can use the Cilium CLI to reconfigure Cilium and enable Hubble:

```sh
cilium hubble enable --ui --wait=false
```

Note the optional flags:

- `--ui` to enable both the required Hubble Relay component which we will showcase first, and the optional Hubble UI component which we will showcase afterwards.
- `--wait=false` to avoid waiting for the pods to be ready, just so we can have a look at what happened under the hood:

```sh
kubectl get pods -A
kubectl get cm -n kube-system
```

With the reconfiguration:

- New Hubble pods for both components have been added to the `kube-system` namespace.
- The `cilium-config` ConfigMap has been updated.
- New Hubble ConfigMaps for both components have been added.

Since we bypassed the wait, we can wait for Cilium and Hubble to be ready by running `cilium status` with the optional `--wait` flag:

```sh
cilium status --wait
```

Note that `cilium status` now displays additional information regarding Hubble components.

Once ready, we can locally port-forward to Hubble Relay:

```sh
cilium hubble port-forward&
```

And then check Hubble status via the Hubble CLI:

```sh
hubble status
```

The Hubble CLI is now primed for observing network traffic within the cluster.

## Observing flows with Hubble

```sh
hubble observe
hubble observe -f
```

If we open a separate terminal and generate some network activity between our frontends and backend again, we will see it pop up in the feed:

```sh
FRONTEND=$(kubectl get pods -l app=frontend -o jsonpath='{.items[0].metadata.name}')
NOT_FRONTEND=$(kubectl get pods -l app=not-frontend -o jsonpath='{.items[0].metadata.name}')
for i in {1..10}; do
  kubectl exec -ti ${FRONTEND} -- curl -I --connect-timeout 5 backend:8080 | head -1
  kubectl exec -ti ${NOT_FRONTEND} -- curl -I --connect-timeout 5 backend:8080 | head -1
done
```

Traffic can be selectively filtered.
Some examples to specifically retrieve the network activity between our frontends and backend:

```sh
hubble observe --to-pod backend
hubble observe --namespace default --protocol tcp --port 8080
hubble observe --verdict DROPPED
```

Note that Hubble tells us the reason a packet was `DROPPED` (in our case, denied by the network policies applied above).
This is really handy when developing / debugging network policies.

## Hubble UI

Since we enabled the additional Hubble UI component, not only can we inspect flows from the command line, we can also see them in real time on a graphical service map.

To start Hubble UI:

```sh
cilium hubble ui
```

The browser should automatically open http://localhost:12000/ (open it manually if not).
We can then access the graphical service map by selecting a namespace.

In this example, we will generate some network activity by running a connectivity test in a separate terminal:

```sh
cilium connectivity test
```

The `cilium-test` namespace should now be available from Hubble UI.

Hubble flows (identical to those from `hubble observe`) are displayed in real time at the bottom, with a visualization of the namespace objects in the center.
Click on any flow, and click on any property from the right side panel: notice that the filters at the top of the UI have been updated accordingly.

After some time, the connectivity test will run tests reaching out to the outside world: Hubble observes not only flows within our cluster but also flows going in or out.

# Conclusion

In this Installfest, we have used Cilium and Hubble CLIs to install Cilium in a Kubernetes cluster, and then manipulated Network Policies and Hubble.
We are not going to showcase them in this demo, but Cilium supports many more features.
To highlight some which you might want to explore next:

## Encryption

Documentation: https://docs.cilium.io/en/stable/gettingstarted/encryption/

Cilium supports transparent encryption with IPsec and Wireguard.
If using the CLI for installing Cilium, enabling it is extremely simple:

```sh
cilium install --encryption ipsec
cilium install --encryption wireguard
```

## Clustermesh

Documentation:

- https://docs.cilium.io/en/stable/gettingstarted/clustermesh/clustermesh/
- https://docs.cilium.io/en/stable/gettingstarted/external-workloads/

Cilium supports meshing together multiple clusters or connecting non-Kubernetes workloads (e.g. VMs) to a Kubernetes cluster.
Akin to Hubble, clustermesh capabilities are an optional component managed from the Cilium CLI:

```sh
cilium clustermesh --help
```

## Slack

Thank you for following the Installfest :)
If you have any questions or topics you'd like to discuss, now is a good time.
You are also most welcome to join our Slack at https://cilium.herokuapp.com/.

# Clean up

Documentation: https://minikube.sigs.k8s.io/docs/commands/delete/

Run the following command to delete your local Minikube cluster:

```sh
minikube delete
```

If you also want to uninstall the `cilium` and `hubble` CLI, run:

```sh
sudo rm /usr/local/bin/{cilium,hubble}
```
