
# Usefull Link 

* Book: https://developers.redhat.com/e-books/path-gitops
* GitOps Repo example: https://github.com/christianh814/example-kubernetes-go-repo
* Miro: https://miro.com/app/board/uXjVObV39JY=/ 

# Add Sealsecret Primary Key to Bastion

```shell
export BASTION=xxx
scp $HOME/.ssh/sealed-secrets-from-gitops.* $BASTION:/home/lab-user
ssh $BASTION
```

# Bootstrap Cluster
```shell
until oc apply -k https://github.com/lcolagio/lab-gitops-v2/cluster-XXXX/bootstrap/overlays/default/; do sleep 5; done
```

## Kustomize

### Install Kustomize

```shell
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
sudo mv kustomize /usr/bin/kustomize
```

### Check kustomize

```shell
kustomize build https://github.com/lcolagio/lab-gitops-v2/cluster-XXXX/bootstrap/overlays/default
```


# Kubernetes GitOps Repo

This is an example on how I would structure a 1:1 (repo-to-single cluster)
git repo for a Kubernetes cluster. This is based on the [OpenShift](https://github.com/christianh814/example-openshift-go-repo)
repo I created.

This example assumes (as I mentioned in the 1:1 part above) that it's a
single repo for a single cluster. However, this can be modified (quite
easily) for poly/mono repos or for multiple clusters. This is meant as
a good starting point and not what your final repo will look like.

This is based on Argo CD but the same principals can be applied to Flux.

# Structure

Below is an explanation on how this repo is laid out. You'll notice
that I use [Kustomize](https://kustomize.io/) heavily. I do this since I
follow the [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)
principal when it comes to YAML files.

```shell
cluster-XXXX/ # 1
├── bootstrap # 2
│   ├── base
│   │   ├── argocd-ns.yaml
│   │   └── kustomization.yaml
│   └── overlays
│       └── default
│           └── kustomization.yaml
├── components # 3
│   ├── applicationsets
│   │   ├── core-components-appset.yaml
│   │   ├── kustomization.yaml
│   │   └── tenants-appset.yaml
│   └── argocdproj
│       ├── kustomization.yaml
│       └── test-project.yaml
├── cluster-config # 4
│   ├── gitops-controller
│   │   └── kustomization.yaml
│   └── sample-admin-config
│       ├── kustomization.yaml
│       └── sample-admin-config.yaml
└── tenants # 5
    ├── bgd-blue
    │   ├── bgd-deployment.yaml
    │   └── kustomization.yaml
    └── myapp
        ├── kustomization.yaml
        ├── myapp-deployment.yaml
        ├── myapp-ns.yaml
        └── myapp-service.yaml
```

|#|Directory Name|Description|
|---|----------------|-----------------|
| 1. |`cluster-XXXX`| This is the cluster name. This name should be unique to the specific cluster you're targeting. If you're using CAPI, this should be the name of your cluster, the output of `kubectl get cluster`|
| 2. | `bootstrap` | This is where bootstrapping specifc configurations are stored. These are items that get the cluster/automation started. They are usually install manifests.<br /><br />`base` is where are the "common" YAML would live and `overlays` are configurations specific to the cluster.<br /><br />The `kustomization.yaml` file in `default` has `cluster-XXXX/components/applicationsets/` and `cluster-XXXX/components/argocdproj/` as a part of it's `bases` config.|
| 3. | `components` | This is where specific components for the GitOps Controller lives (in this case Argo CD).<br /><br />`applicationsets` is where all the ApplicationSets YAMLs live and `argocdproj` is where the ArgoAppProject YAMLs live.<br /><br />Other things that can live here are RBAC, Git repo, and other Argo CD specific configurations (each in their repsective directories).|
| 4. | `cluster-config` | This is where YAML for the core functionality of the cluster live. Here is where the Kubernetes administrator will put things that is necissary for the functionality of the cluster (like cluster configs or cluster workloads).<br /><br />Under `gitops-controller` is where you are using Argo CD to manage itself. The `kustomization.yaml` file uses `cluster-XXXX/bootstrap/overlays/default` in it's `bases` configuration. This `core` directory gets deployed as an applicationset which can be found under `cluster-XXXX/components/applicationsets/core-components-appset.yaml`.<br /><br />To add a new "core functionality" workoad, one needs to add a directory with some yaml in the `core` directory. See the `sample-admin-config` directory as an example.|
| 5. | `tenants` | This is where the workloads for this cluster live.<br /><br />Similar to `core`, the `tenants` directory gets loaded as part of an ApplicationSet that is under `cluster-XXXX/components/applicationsets/tenants-appset.yaml`.<br /><br />This is where Devlopers/Release Engineers do the work. They just need to commit a directory with some YAML and the applicationset takes care of creating the workload.<br /><br /> **Note** that `bgd-blue/kustomization.yaml` file points to another Git repo. This is to show that you can host your YAML in one repo, or many repos.|

# Testing

To see this in action, first get yourself a cluster (using [kind](kind.sigs.k8s.io/) as an example)

```shell
kind create cluster
```

Then, just apply this repo.

```shell
until kubectl apply -k https://github.com/christianh814/example-kubernetes-go-repo/cluster-XXXX/bootstrap/overlays/default; do sleep 3; done
```

This should give you 4 applications

```shell
$ kubectl get applications -n argocd
NAME                          SYNC STATUS   HEALTH STATUS
bgd-blue                      Synced        Healthy
sample-admin-workload         Synced        Healthy
myapp                         Synced        Healthy
gitops-controller             Synced        Healthy
```

Backed by 2 applicationsets

```shell
$ kubectl get appsets -n argocd
NAME      AGE
cluster   110s
tenants   110s
```

To see the Argo CD UI, you'll first need the password

```shell
kubectl get secret/argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d ; echo
```

Then port-forward to see it in your browser (using `admin` as the username).

```shell
kubectl -n argocd port-forward service/argocd-server 8080:443
```

