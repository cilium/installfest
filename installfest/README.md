# Set up Minikube cluster

Documentation: https://minikube.sigs.k8s.io/docs/commands/start/

Run the following command to set up your local Minikube cluster:

```sh
minikube start --network-plugin=cni
```

We can check that the cluster has properly started by verifying the current `kubectl` context is `minikube`:

```sh
kubectl config current-context
```

And listing the pods in the cluster:

```sh
kubectl get pods -A
```

Note that the `coredns` pod is stuck in `ContainerCreating`: this is normal as our cluster does not have any CNI installed for now.

# Cilium

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

## Install Cilium in the Minikube cluster

Install Cilium:

```sh
cilium install
```

Then verify the installation:

```sh
cilium status
```

Take a look at the pods again to see what happened under the hood:

```sh
kubectl get pods -A
```

Cilium is now properly installed and manages connectivity within the cluster, allowing `coredns` to start properly.

We can verify connectivity by running a connectivity test (which will be deployed in namespace `cilium-test`):

```sh
cilium connectivity test
```

Once done, clean up the connectivity test namespace:

```sh
kubectl delete ns cilium-test&
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
echo ${FRONTEND}
NOT_FRONTEND=$(kubectl get pods -l app=not-frontend -o jsonpath='{.items[0].metadata.name}')
echo ${NOT_FRONTEND}
kubectl exec -ti ${FRONTEND} -- curl -I --connect-timeout 5 backend:8080
kubectl exec -ti ${NOT_FRONTEND} -- curl -I --connect-timeout 5 backend:8080
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
kubectl exec -ti ${FRONTEND} -- curl -I --connect-timeout 5 backend:8080
kubectl exec -ti ${NOT_FRONTEND} -- curl -I --connect-timeout 5 backend:8080
```

The network policy correctly switched the default ingress behavior from default allow to default deny.
Let's now selectively re-allow traffic again, but only from frontend to backend.

We can do it by crafting a new network policy manually, but we can also use the Network Policy Editor to help us out:

- Go to https://networkpolicy.io/editor.
- Upload our initial `backend-ingress-deny` policy.
- Rename the network policy to `backend-allow-ingress-frontend` (using the `Edit` button in the center).
- On the ingress side, add `app=frontend` as `podSelector` for pods in the same namespace.
- Inspect the ingress flow colors: the policy will deny all ingress traffic to pods labeled `app=backend`, except for traffic coming from pods labeled `app=frontend`.
- Download the policy YAML file.

Apply the new policy and check that connectivity has been restored, but only from the frontend:

```sh
kubectl create -f backend-allow-ingress-frontend.yaml
kubectl exec -ti ${FRONTEND} -- curl -I --connect-timeout 5 backend:8080
kubectl exec -ti ${NOT_FRONTEND} -- curl -I --connect-timeout 5 backend:8080
```

Note that this is working despite the fact we did not delete the previous `backend-ingress-deny` policy:

```sh
kubectl get netpol
```

Network policies are additive.
Just like with firewalls, it is thus a good idea to have default `DENY` policies and then add more specific `ALLOW` policies as needed.

# Cilium Network Policies

TODO

# Hubble

By default, Cilium acts only as a CNI and is thus mostly responsible for networking, though it can help a bit with security (e.g. advanced network policies).
To take full advantage of eBPF deep observability and security capabilities, we must enable the optional Hubble component (which is disabled by default).

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

Enable the optional Hubble component:

```sh
cilium hubble enable
```

Take a look at the pods again to see what happened under the hood:

```sh
kubectl get pods -A
```

Cilium agents are restarting, and a new Hubble Relay pod is now present.
We can wait for Cilium and Hubble to be ready by running:

```sh
cilium status --wait
```

Once ready, we can locally port-forward to the Hubble pod:

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

If we try to spawn some network activity between our frontends and backend again, we might see it pop up in the feed:

```sh
for i in {1..10}; do
  kubectl exec -ti ${FRONTEND} -- curl -I --connect-timeout 5 backend:8080
  kubectl exec -ti ${NOT_FRONTEND} -- curl -I --connect-timeout 5 backend:8080
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

Not only does Hubble allow to inspect flows from the command line, it also allows us to see them in real time on a graphical service map via Hubble UI.
Again, this also is an optional component that is disabled by default.

Enable the optional Hubble UI component:

```sh
cilium hubble enable
```

Take a look at the pods again to see what happened under the hood:

```sh
kubectl get pods -A
```

Cilium agents are restarting, and a new Hubble UI pod is now present on top of the Hubble Relay pod.
As above, we can wait for Cilium and Hubble to be ready by running:

```sh
cilium status --wait
```

And then check Hubble status:

```
hubble status
```

> Note: our earlier `cilium hubble port-forward` should still be running (can be checked by running `jobs` or `ps aux | grep "cilium hubble port-forward"`).
> If it does not, `hubble status` will fail and we have to run it again:
>
> ```sh
> cilium hubble port-forward&
> hubble status
> ```

To start Hubble UI:

```sh
cilium hubble ui
```

The browser should automatically open http://localhost:12000/ (open it manually if not).
We can then access the graphical service map by selecting our `default` namespace, and generating some network activity again:

```sh
for i in {1..10}; do
  kubectl exec -ti ${FRONTEND} -- curl -I --connect-timeout 5 backend:8080
  kubectl exec -ti ${NOT_FRONTEND} -- curl -I --connect-timeout 5 backend:8080
done
```

Hubble flows are displayed in real time at the bottom, with a visualization of the namespace objects in the center.
Click on any flow, and click on any property from the right side panel: notice that the filters at the top of the UI have been updated accordingly.

Let's run a connectivity test again and see what happens in Hubble UI in the `cilium-test` namespace:

```sh
cilium connectivity test
```

We can see that Hubble UI is not only capable of displaying flows within a namespace, it also helps visualizing flows going in or out.

# Conclusion

In this Installfest, we have used Cilium and Hubble CLIs to install Cilium in a Kubernetes cluster, and then manipulated Network Policies and Hubble.
We are not going to showcase them in this demo, but Cilium supports many more features.
To highlight some of them which you might want to explore as a next step building upon the knowledge gained during this Installfest:

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

Cilium support meshing together multiple clusters or connecting non-Kubernetes workloads (e.g. VMs) to a Kubernetes cluster.
Akin to Hubble, clustermesh capabilities are an optional component managed from the Cilium CLI:

```sh
cilium clustermesh --help
```

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
