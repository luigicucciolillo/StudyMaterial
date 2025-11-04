# Cert-Manager Beyond Ingress – Exploring the Variety of Use Cases - Matthew Bates, Jetstack

Cert-manager is a widely used project for the automation of X.509 TLS certificates. In 2020, it reached 1.0 and landed in the CNCF Sandbox. cert-manager has been popularised by its support of ACME and Ingress, enabling many millions of certificates to be issued and renewed, and to help secure the cloud native web with Kubernetes and all the various ingress controllers. But cert-manager, with its custom resources and controllers, extensible with issuers including those out-of-tree, can also be used for a myriad of other use cases in which certificates are required. This talk will walk through the various use cases for cert-manager, including ingress, control plane and nodes (kubeadm, CAPI), webhooks, intra-service mTLS (cert-manager-csi) and service mesh (OpenServiceMesh, Istio).

https://www.youtube.com/watch?v=wEW2kVKxgss
