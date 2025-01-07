# Istio Installation

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-<version-number>
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
```	

# Istio injection 

```bash
# Verify the istio-injection label is set to enabled
istioctl analyze
```
=> If Isio is not enabled in the namespace, the output will be like the following:
```console
Info [IST0102] (Namespace default) The namespace is not enabled for Istio injection.
Run 'kubectl label namespace default istio-injection=enabled' to enable it, or
'kubectl label namespace default istio-injection=disabled' to explicitly mark it as
not needing injection.
```
also, you can verify that by cheking the number of containers in each pod in the specified namespace. it should be 2 if istio is enabled.
```bash
kubectl get pods -n <namespace>
```
Ouput:
```console
NAME                     READY   STATUS    RESTARTS   AGE
productpage-v1-5d6d8646d-42424   2/2     Running   0          10m
```
=> If the output is 1/1, then istio is not enabled in the namespace.
```bash
# Enable Istio injection in the Default namespace
kubectl label namespace default istio-injection=enabled
```
```bash
# Disable Istio injection in the Default namespace
kubectl label namespace default istio-injection=disabled
```
Note: If you have already pods deployed in the namespace before istio inejction being enabled, Istio will not inject the envoy sidecar container in them automatically. You need to redeploy them.
```bash
# Delete all deployments in the namespace
kubectl delete deployment --all -n <namespace>
# Recreate the deployments 
kubectl apply -f <deployment-file>.yaml
```

