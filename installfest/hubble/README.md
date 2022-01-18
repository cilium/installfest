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

Let's generate some network activity by running a connectivity test in a separate terminal:

```sh
cilium connectivity test
```

And then try to observe network traffic:

```sh
hubble observe
hubble observe -f
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
We can then access the graphical service map by selecting the `cilium-test` namespace.

Hubble flows (identical to those from `hubble observe`) are displayed in real time at the bottom, with a visualization of the namespace objects in the center.
Click on any flow, and click on any property from the right side panel: notice that the filters at the top of the UI have been updated accordingly.

Hubble observes not only flows within our cluster but also flows going in or out: as the connectivity tests reach out to the outside world, we should these pop up in the graphical service map.
