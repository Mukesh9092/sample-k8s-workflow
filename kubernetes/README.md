Minikube install and configuration
GoCD Install via Helm
Kubernetes settings for GoCD


## Minikube install and configuration
Steps for operating `minikube` on a Mac
```
#  Install minikube
brew install minikube

#  Use one option to start minikube (depending on virtualization driver installed)
minikube start --vm-driver=hyperkit --cpus=4 --memory 6144 --addons=ingress
minikube start --vm-driver=virtualbox --cpus=4 --memory 6144 --addons=ingress

#  Check kubernetes points to minikube
kubectl config get-contexts
kubectl config current-context

#  Check minikube health
kubectl get pods --namespace kube-system

#  Delete/reset minikube cluster
minikube delete
```

## GoCD Install via Helm
Installation using [Helm v3](https://helm.sh/)
```
#  Install helm3
brew install helm

#  Create GoCD namespace (needed since helm doesn't create namespaces during install anymore!!!)
kubectl create ns gocd

#  Add stable charts repo
helm repo add stable https://kubernetes-charts.storage.googleapis.com/

#  Check repo is listed
helm repo list

#  Search for charts
helm search hub gocd
helm search repo gocd

#  Install GoCD
helm install gocd stable/gocd --namespace gocd

#  Check status of pods
kubectl get pods --namespace gocd
```


## Kubernetes settings for GoCD
Installation of [sample app](https://docs.gocd.org/current/gocd_on_kubernetes/importing_a_sample_workflow.html)
**Note**: Ensure all commands are run against `gocd` namespace
```
#
#  Kubernetes service-account + cluster-role
#

#  Create service-account for "gocd" user
kubectl create serviceaccount gocd-user --namespace gocd

#  Create cluster-role for "gocd" deployer
kubectl create -f deployment-role.yml

#  Bind "gocd" serviceaccount to "gocd" cluster-role
kubectl create rolebinding gocd-user:gocd-deployer --clusterrole gocd-deployer --serviceaccount default:gocd-user --namespace gocd


#
#  Kubernetes "secrets"
#
#  Extract encrypted gocd Kubernetes API "token" (already in base64) to use within GoCD pipeline
#
SERVICE_ACCOUNT=gocd-user
SECRET=$(kubectl get serviceaccount ${SERVICE_ACCOUNT} -o json | jq -Mr '.secrets[].name | select(contains("token"))')
TOKEN=$(kubectl get secret ${SECRET} -o json | jq -Mr '.data.token')

#  Copy "token" to clipboard and paste it to "secrets-for-gocd.yaml" file under 'data.K8S_API_TOKEN',
#  ensure no spurious CR/LFs
echo -n ${TOKEN} | pbcopy

#  For 'data.DOCKERHUB_USERNAME' and 'data.DOCKERHUB_ORG', base64 + copy the following commands output
echo -n 'ifarfan' | base64 | pbcopy
echo -n 'ifarfanorg' | base64 | pbcopy

#  Load "gocd" secrets, to be used by GoCD pipelines to interact with Kubernetes during deployments
kubectl apply -f secrets-for-gocd.yaml -n gocd

#  Check "gocd" secrets
kubectl get secret --namespace gocd secrets-for-gocd -o json | jq
kubectl get secret --namespace gocd secrets-for-gocd -o yaml | hly


#
#  Kubernetes "miscellaneous"
#

#  Get the API Server location
APISERVER=https://$(kubectl -n default get endpoints kubernetes --no-headers | awk '{ print $2 }')
echo ${APISERVER}
```
