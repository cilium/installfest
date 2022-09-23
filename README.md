# Archival note

The Installfest has now been superseded by the **Cilium Introduction and Live Q&A** and the **Cilium lab tutorials**. Head over to https://cilium.io/get-started/ to learn more.

---

# Cilium Installfest

Welcome! This repository contains documentation and resources for [Cilium](https://github.com/cilium/cilium) Installfest.

The Installfest is a live community event where experienced Cilium users will help you set up Cilium and answer your questions around specific setups or features.
Our goal is to provide anyone interested in Cilium an interactive opportunity to learn about its capabilities and how they could be tailored to their needs.

> Looking for lab resources to follow on your own? We recommend you instead try a [Cilium lab tutorial](https://cilium.io/enterprise/#trainings).

## Registering for the Installfest

Two schedules are available to accommodate for EMEA and NA timezones:

- EMEA: https://calendly.com/cilium-events/cilim-installfest-emea
- NA: https://calendly.com/cilium-events/cilium-installfest-na

## Before the Installfest

To ensure a smooth Installfest, please make sure your local environment is set up and ready before the event.

### Technical prerequisites

- Familiarity with command line usage and basic Kubernetes operations
- A Linux or Intel-based Mac machine with `kubectl` installed: https://kubernetes.io/docs/tasks/tools/#kubectl
- [Minikube](https://minikube.sigs.k8s.io/docs/start/) (>= 1.12.0) or [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) (>= 0.7.0) installed

> Note: we recommend using local clusters such as Minikube or Kind when playing with Cilium for testing, but the same could be done on any GKE, EKS, or AKS cluster.
> See our documentation for more details on preparing a GKE, EKS, or AKS cluster for Cilium: https://docs.cilium.io/en/stable/gettingstarted/

### Checklist

- Check that starting a local cluster works properly:
  - Minikube: run `minikube start --network-plugin=cni --cni=false --kubernetes-version=1.21.6`
  - Kind: run `curl -LO https://raw.githubusercontent.com/cilium/cilium/1.11.0/Documentation/gettingstarted/kind-config.yaml && kind create cluster --config=kind-config.yaml`
- [Download](https://github.com/cilium/installfest/archive/refs/heads/main.zip) or `git clone` this repository to have a local copy of the Installfest resources we might use.
- Make sure your computer is fully charged or plugged in, internet connection is up and running, water and snacks are at your disposal, and make yourself comfortable :)

## During the Installfest

The staff will introduce itself and give an overview of the session.
We will start by installing Cilium on our local clusters, using the resources in [the `installfest` directory](./installfest) for guidance.
Afterwards, we will discuss and choose together what to explore depending on the audience's wishes.

All questions are welcome :)
Don't hesitate to reach out either by opening the microphone or via chat.
Please be respectful and patient in all circumstances, and keep microphones muted when not talking.
