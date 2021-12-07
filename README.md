# Cilium Installfest

Welcome! This repository contains documentation and resources for [Cilium](https://github.com/cilium/cilium) Installfest.

The Installfest is a live community event where experienced Cilium users will help you set up Cilium and give you a hands-on overview of its capabilities.
Our goal is to provide anyone interested in Cilium with step-by-step instructions for quickly installing Cilium and manipulating it while illustrating real-world use cases.

> Note: the step-by-step instructions below can still be followed offline should you prefer to run through on your own.

## Before the Installfest

To ensure a smooth Installfest, please make sure your local environment is set up and ready before the event.

### Technical prerequisites

- Familiarity with command line usage and basic Kubernetes operations
- A Linux or Intel-based Mac machine with `kubectl` installed: https://kubernetes.io/docs/tasks/tools/#kubectl
- Minikube installed (>= 1.12.0): https://minikube.sigs.k8s.io/docs/start/

> Note: while we strongly recommend using Minikube, the Installfest may also be followed on any GKE, EKS, or AKS cluster, though you will not get support from the staff in case of cluster-related issues during the event.
> See our documentation for more details on preparing a GKE, EKS, or AKS cluster for Cilium: https://docs.cilium.io/en/stable/gettingstarted/

### Checklist

- Run `minikube start --network-plugin=cni --cni=false --kubernetes-version=1.21.6` a few hours / days before the Installfest in order to download Minikube images, which can take some time.
- [Download](https://github.com/cilium/installfest/archive/refs/heads/main.zip) or `git clone` this repository to have a local copy of the resources used during the Installfest.
- Make sure your computer is fully charged or plugged in, internet connection is up and running, water and snacks are at your disposal, and make yourself comfortable :)

## During the Installfest

The staff will introduce itself and give an overview of what we will be covering.
We will then all follow the step-by-step instructions located in [the `installfest` directory](./installfest).

Please be respectful and patient in all circumstances, and keep microphones muted while the presentation is ongoing.

If at any point you are lost or something is not working, reach out for help via chat.
All questions are welcome, no matter how stupid they may seem :)
