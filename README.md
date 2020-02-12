# Homelab Bootstrap
## Requirements

* Raspberry Pi4 with Raspbian installed
* SSH-Keyexchange for user PI
* GCE service account token as described in @alexellis [great tutorial](https://github.com/inlets/inlets-operator/blob/master/README.md])
* Installed tools:
  * [helm3](https://github.com/helm/helm)
  * [kubens](https://github.com/ahmetb/kubectx)
  * [k3sup](https://github.com/alexellis/k3sup)
  * [fluxctl](https://github.com/fluxcd/flux)

## Install k3s
```
k3sup install --ip <IP OF THE PI> --user pi --k3s-extra-args "--no-deploy=traefik,svclb"
export KUBECONFIG=/PATH/TO/KUBECONFIG
```

I removed traefik an the svclb in favor for nginx. I had some trouble with traefik :-/ I need to invest more time here.

## Setup the "real" system

All system tools should be installed in the *kube-system* namespace.
```
kubens kube-system
```
The GCE token is not managed bei flux, so I had to add it manually.
```
kubectl create secret generic inlets-access-key --from-file=inlets-access-key=gce-token.json
```
The helm-operator and flux are installed first time by hand. After the bootstrap both should be managed by flux.
```
helm upgrade -i helm-operator fluxcd/helm-operator --namespace kube-system --set git.ssh.secretName=flux-git-deploy --set helm.versions=v3 --set image.repository=docker.io/onedr0p/helm-operator --set image.tag=latest

helm upgrade -i flux fluxcd/flux --wait --namespace kube-system --set image.repository=docker.io/onedr0p/flux --set git.user=aidun --set git.email=aidun@users.noreply.github.com --set git.url=git@github.com:aidun/kubernetes_homelab --set git.path='namespaces\,workloads\,releases' --set image.tag=latest
```
The last step is to get the public key of flux and add it to the github deploy keys.
```
fluxctl identity --k8s-fwd-ns kube-system
```
Now you have to wait several minutes. Maybe it needs some loops of the flux sync to get it all installed correct.
To follow the progress watch the logs of flux.
```
kubectl logs deployment.apps/flux
```
# Parts that dont work
* sealed secrets
* external-dns
* traefik

*I will describe later why...*