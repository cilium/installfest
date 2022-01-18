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
