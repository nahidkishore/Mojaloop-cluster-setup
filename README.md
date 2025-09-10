# Mojaloop-cluster-setup


## 📖 About this Documentation

This documentation has been authored by **Nahidul Islam**, based on real-world, production-grade deployment experience as a **DevOps & Cloud Engineer**.
It provides both **conceptual clarity** and **hands-on practical steps** so that engineers can set up, validate, and operate a production-ready Kubernetes cluster.

---

## 🎯 Objectives

* Build an **HA Kubernetes cluster** with:

  * 3 × Control Plane Nodes
  * 3 × Worker Nodes
  * 1 × Load Balancer (HAProxy)
  * 3 × External etcd nodes(optional)
* Ensure **resilience**: cluster stays healthy even if one control-plane or etcd node fails.
* Provide a **reference architecture** for teams looking to deploy Kubernetes outside managed cloud environments.

---
