# Cilium Installfest

Welcome! This repository contains documentation and resources for [Cilium](https://github.com/cilium/cilium) Installfest.

The Installfest is a community event where experienced Cilium users will help you set up Cilium and give you a hands-on overview of its capabilities.
Our goal is to provide anyone interested in Cilium with step-by-step instructions for quickly installing Cilium and manipulating it while illustrating real-world use cases.

> Note: while the Installfest is conceived as a live event where experienced users can help you get up to speed, the step-by-step instructions below can still be followed offline should you prefer to run through on your own.

## Before the Installfest

To ensure a smooth Installfest, please make sure your local environment is set up and ready before the event.

### Technical prerequisites

- Familiarity with Kubernetes
- A Linux or Intel-based Mac machine with `kubectl` installed: https://kubernetes.io/docs/tasks/tools/#kubectl
- Minikube (>= 1.5.2): https://minikube.sigs.k8s.io/docs/start/

> Note: while we strongly recommend using Minikube, the Installfest may also be followed on any GKE, EKS, or AKS cluster, though you will not get support from the Installfest staff in case of cluster-related issues during the event.
> See our documentation for more details on preparing a GKE, EKS, or AKS cluster for Cilium: https://docs.cilium.io/en/stable/gettingstarted/

### Preparing for the event

- Please run `minikube start --network-plugin=cni` a few hours / days before the Installfest in order to download Minikube images, which can take some time.
- [Download](https://github.com/cilium/installfest/archive/refs/heads/main.zip) or `git clone` this repository to have a local copy of the resources used during the Installfest.
- Make sure to have your computer fully charged or plugged in, internet connection up and running, water and snacks at your disposal, and make yourself comfortable :)

## During the Installfest

To ensure a smooth Installfest, please make sure to read and respect the following:

- Be respectful and patient in all circumstances.
- All questions are welcome in the chat, no matter how stupid they may seem.
- Microphones should be kept muted at all times, except when authorized by the staff.

### Step-by-step instructions

All attendees and staff will follow the same step-by-step instructions: see [**Installfest**](./installfest).

If at any point you are lost or something is not working, reach out for help via chat so that the staff and other attendees might have a chance to help you.
If necessary, staff will offer you to open your microphone and share your screen for easier troubleshooting, or ask that the issue be deferred and addressed later in order not to block the session.
