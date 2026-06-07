# Chaos Engineering: Resilience Testing with Chaos Mesh
## AGH Large Scale Computing Project
### Authors: Tomasz Krępa, Michał Zienkowicz

## Introduction:
This instruction provides materials necessary for experimenting with Chaos Engineering and Resilience Testing.
The project includes deploying a multi-servie application on Kubernetes (Kind) and instrumenting it with basic health metrics. Then systematic failure injection is demonstrated (pod kills, network partitions, latency injections and CPU/memory stress). This excersise, allows you to see how each failure manifests in the metrics and how the system recovers.

## Tools used:


# Instruction:

## 1. Docker Desktop (& WSL)
Firs, we recommend using Docker Desktop, which you can install via following link:
[https://docs.docker.com/desktop/](https://docs.docker.com/desktop/)

Most of tools in this excersise are designed for Linux OS. If you work on machine with Windows, we recommend installing Linux via WSL. In our case we have used {mint?}, but most of distributions are expected to be compatible:
[https://learn.microsoft.com/en-us/windows/wsl/install](https://learn.microsoft.com/en-us/windows/wsl/install)

If you are using Linux via WSL, make sure to enable WSL integration in Docker Desktop by going to `Settings` -> `Resources` -> `WSL Integration` and enabling the integration.

![alt text](image-2.png)



