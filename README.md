# Homelab Bootstrap

The intention of the repo is to create a full featured homelab / mini cluster which can run on arm and amd64 machines.

Current Status

- [x] Gitops: Fluxcd
- [x] Gitops: Helm-Operator
- [x] Storage: Openebs
- [x] Ingress-Remote: Inlets
- [x] Ingress-Local: Metallb
- [x] Ingress: Nginx
- [ ] Ingress: TLS
- [X] Monitoring - Logs: Loki
- [ ] Monitoring - Logs: Grafana
  - [x] Grafana
  - [ ] auto provisioning of datasource and dashboard
- [ ] Monitoring - Merics: Prometheus
- [ ] Secret-Management - Sealed Secrets
- [ ] External-DNS - external-dns

## Requirements

* Raspberry Pi4 with Ubuntu 18.04 (19.10 has some strange ipv6 issues ) installed and the following additional software packages installed:
  * iscsi 
    * sudo apt-get update
    * sudo apt-get install open-iscsi
    * sudo service open-iscsi restart
* cgroups enabled / cgroupsmemory enabled (https://askubuntu.com/questions/1189480/raspberry-pi-4-ubuntu-19-10-cannot-enable-cgroup-memory-at-boostrap)
* SSH-Keyexchange for a user, who can sudo su
* GCE service account token as described in @alexellis [great tutorial](https://github.com/inlets/inlets-operator/blob/master/README.md])
* Installed tools:
  * [helm3](https://github.com/helm/helm)
  * [kubens](https://github.com/ahmetb/kubectx)
  * [k3sup](https://github.com/alexellis/k3sup)
  * [fluxctl](https://github.com/fluxcd/flux)

## Install k3s
```
k3sup install --ip <IP OF THE PI> --user <USER> --k3s-extra-args "--no-deploy=traefik,svclb"
export KUBECONFIG=/PATH/TO/KUBECONFIG
```

I removed traefik an the svclb in favor for nginx. I had some trouble with traefik :-/ I need to invest more time here.

## Setup the "real" system

All system tools should be installed in the *kube-system* namespace.
```
kubens kube-system
```
Openebs should be installed at first, to get distributed storage for all other tools:
```
#TODO: not working arm helm chart
#helm repo update
#helm upgrade -i openebs --namespace kube-system stable/openebs --version 1.7.0 --set defaultStorageConfig.enabled=true
kubectl apply -f https://raw.githubusercontent.com/openebs/charts/master/docs/openebs-operator-arm-dev.yaml
k annotate sc openebs-snapshot-promoter storageclass.kubernetes.io/is-default-class=true
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