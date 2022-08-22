# Example of Podinfo in a public repo
Secret is optional.



## Create Secret for GitRepository credentials




This is necessary for Private repositories. But for Public repo, this is not needed.

In the example below, we use a GitHub basic authentication. Other authentication options are explained on the flux documentation.

The basic authentication uses the GitHub username and Personal Access Token (PAT), that you can generate through this guide.




Encode your GitHub username and PAT as base64
```
echo -n “<github_username>” | base64
```
```
echo -n “<github_token>” | base64
```

Use the encoded username and PAT to create the secret

```
apiVersion: v1
data:
  password: <base64_PAT>
  username: <base64_username>
kind: Secret
metadata:
  name: private-repo-credentials
  namespace: kommander-flux
type: Opaque
```


## Optionally you can use kubeseal to seal the secret if it will be uploaded to a repo.
But first you need to deploy kubeseal on the cluster.
```
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.1/controller.yaml
```
Install kubeseal CLI
```
brew install kubeseal
```
Fetch the deployed cert on the cluster
```
kubeseal --fetch-cert > mycert.pem
```
Seal the nonsealed secret
```
kubeseal -f non-sealed-secret.yaml --cert myclustercert.pem > sealed-secret.yaml
```

And apply the manifest using the kubectl apply
```
kubectl apply -f sealed-secret.yaml
```
Or just push it to the repo to let flux take care of the deployment.




## Create a GitRepository Source

The GitRepository object will contain the URL of the repo, the branch reference and the secret reference for authentication.

```
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
 name: podinfo-gitrepo-private
 namespace: kommander-flux
spec:
 interval: 60s
 ref:
   branch: main
 secretRef:
   name: private-repo-secret
 url: https://github.com/<githhub_username>/<repository_name>
```
```
kubectl apply -f podinfo-gitrepo-private.yaml
```



## Deploy Kustomization

Kustomization will include the referenced GitRepository(source) created earlier, the path of the directory where the manifests are in the repo and the targetNamespace where to deploy the application. TargetNamespace is optional if the deployment manifests does not specify the namespace.



```
apiVersion: v1
kind: Namespace
metadata:
  name: podinfo-private-ns
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: podinfo-kustomization-private
  namespace: kommander-flux
spec:
  interval: 5m0s
  path: ./kustomize
  prune: true
  sourceRef:
    kind: GitRepository
    name: podinfo-gitrepo-private
  targetNamespace: podinfo-private-ns
```
```
Kubectl apply -f podinfo-kustomization-private.yaml
```
