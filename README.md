[![CI](https://github.com/jkroepke/kubernetes-from-scratch/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/jkroepke/kubernetes-from-scratch/actions/workflows/ci.yaml)
[![GitHub license](https://img.shields.io/github/license/jkroepke/kubernetes-from-scratch)](https://github.com/jkroepke/kubernetes-from-scratch/blob/master/LICENSE.txt)
[![Current Release](https://img.shields.io/github/release/jkroepke/kubernetes-from-scratch.svg?logo=github)](https://github.com/jkroepke/kubernetes-from-scratch/releases/latest)
[![GitHub Repo stars](https://img.shields.io/github/stars/jkroepke/kubernetes-from-scratch?style=flat&logo=github)](https://github.com/jkroepke/kubernetes-from-scratch/stargazers)

# kubernetes-from-scratch

⭐ Don't forget to star this repository! ⭐

## About

Kubernetes From Scratch is a hands on guide that walks through setting up Kubernetes step by step without installers, kubeadm, or managed services. You use official binaries, but you configure and wire everything yourself.

No automation. No magic.

You generate certificates, configure etcd, set up the API server, controller manager, scheduler, and kubelet manually. You define RBAC rules, bootstrap nodes, and configure networking without abstraction layers.

The focus is deep understanding of how Kubernetes components interact, how trust is established with TLS, how authentication and authorization work, and how the control plane behaves under the hood.

You are not compiling Kubernetes from source. You are building a cluster from first principles.

After finishing, you will not just operate Kubernetes.
You will understand why it works.
