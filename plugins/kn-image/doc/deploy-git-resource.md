## Define a service account for github resource on IKS

Please create service account so that it can access protected resources. The service account ties together secrets containing credentials for authentication with RBAC-related resources for permission to create and modify specific Kubernetes resources.

First you enable programmatic access to your private container registry by creating either a registry token or an IBM Cloud Identity and Access Management (IAM) API key. The process for creating a token or an API key is described here.

After you have the token or API key, you can create the following secret:
```
kubectl create secret generic ibm-cr-push-secret --type="kubernetes.io/basic-auth" --from-literal=username=<USER> --from-literal-password=<TOKEN/APIKEY>
kubectl annotate secret ibm-cr-push-secret tekton.dev/docker-0=<REGISTRY>
```
where

- <USER> is either token if you are using a token or iamapikey if you are using an API key
- <TOKEN/APIKEY> is either the token or API key that you created
- <REGISTRY> is the URL of your container registry, such as us.icr.io or registry.ng.bluemix.net

Now you can create the service account using the following YAML file. The complete YAML file is available at tekton/pipeline-account.yaml.
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-account
secrets:
- name: ibm-cr-push-secret

---

apiVersion: v1
kind: Secret
metadata:
  name: kube-api-secret
  annotations:
    kubernetes.io/service-account.name: pipeline-account
type: kubernetes.io/service-account-token

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pipeline-role
rules:
- apiGroups: ["serving.knative.dev"]
  resources: ["services"]
  verbs: ["get", "create", "update", "patch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pipeline-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pipeline-role
subjects:
- kind: ServiceAccount
  name: pipeline-account
```

This YAML creates the following Kubernetes resources:

- A ServiceAccount named pipeline-account. The PipelineRun you saw earlier uses this name to reference the account. The service account references the ibm-cr-push-secret secret so that the pipeline can authenticate to your private container registry when it pushes a container image.
- A Secret named kube-api-secret, which contains an API credential (generated by Kubernetes) for accessing the Kubernetes API. This kube-api-secret allows the pipeline to use kubectl to talk to your cluster.
- A Role named pipeline-role and a RoleBinding named pipeline-role-binding, which provide the resource-based access control permissions needed for the pipeline to create and modify Knative services.

To create the service account and related resources, apply the file to your cluster:
```
kubectl apply -f tekton/pipeline-account.yaml
```

The readme comes from: https://developer.ibm.com/tutorials/knative-build-app-development-with-tekton/
