# Use Docker Desktop enable kubernetes feature.
docs: https://docs.docker.com/desktop/kubernetes/#enable-kubernetes
![](/images/01.png)


## Pre-install
- [yq](https://github.com/mikefarah/yq)

## Get Cluster self-sign CACERT
```bash
$ cat ~/.kube/config  | yq '.clusters[0].cluster.certificate-authority-data' | base64 -d > cluser-certificate-authority-data.crt
```

## Get Cluster admin client certificate-data
```bash
$ cat ~/.kube/config  | yq '.users[] | select(.name == "docker-desktop").user.client-certificate-data' | base64 -d > admin-client-certificate-data.crt
```

## Get Cluster admin client key-data
```bash
$ cat ~/.kube/config  | yq '.users[] | select(.name == "docker-desktop").user.client-key-data' | base64 -d > admin-client-key-data.key
```


## Access kubernets version api
```bash
$ KUBE_API=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')
```

```bash
$ curl --cacert cluser-certificate-authority-data.crt $KUBE_API/version
{
  "major": "1",
  "minor": "24",
  "gitVersion": "v1.24.0",
  "gitCommit": "4ce5a8954017644c5420bae81d72b09b735c21f0",
  "gitTreeState": "clean",
  "buildDate": "2022-05-03T13:38:19Z",
  "goVersion": "go1.18.1",
  "compiler": "gc",
  "platform": "linux/arm64"
}
```

## List all Deployments in the cluster without authenticated
```bash
$ curl --cacert cluser-certificate-authority-data.crt $KUBE_API/apis/apps/v1/deployments
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "deployments.apps is forbidden: User \"system:anonymous\" cannot list resource \"deployments\" in API group \"apps\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "group": "apps",
    "kind": "deployments"
  },
  "code": 403
}
```


## How to send a request authenticated by that certificate to the Kubernetes API server using curl:

```bash
$ curl $KUBE_API/apis/apps/v1/deployments \
  --cacert cluser-certificate-authority-data.crt \
  --cert admin-client-certificate-data.crt \
  --key admin-client-key-data.key
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "resourceVersion": "4726"
  },
  "items": [...]
} 
```


## Authenticating Client to API Server using Service Account Tokens
Another way to authenticate API requests is through using a bearer header containing a valid Service Account JWT token.

### pre-create [source](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- service-account: `lab`
- role: `lab-admin`
- rolebinding: `lab-admin-binding`

```bash
$ kubectl apply -f create-lab-service-account.yaml

$ kubectl get role,rolebinding,sa
NAME                                       CREATED AT
role.rbac.authorization.k8s.io/lab-admin   2022-07-06T02:39:19Z

NAME                                                      ROLE             AGE
rolebinding.rbac.authorization.k8s.io/lab-admin-binding   Role/lab-admin   19s

NAME                     SECRETS   AGE
serviceaccount/default   0         77m
serviceaccount/lab       0         58s
```

### Create token for lab service account 
```bash
$ kubectl create token lab --duration=8760h > extra

$ kubectl delete secrets lab && kubectl create secret generic lab --from-file extra || kubectl create secret generic lab --from-file extra

$ kubectl patch secrets lab -p '{"metadata":{"annotations": {"kubernetes.io/service-account.name": "lab"}}}'
secret/lab patched
```

```bash
$ JWT_TOKEN_DEFAULT_DEFAULT=$(cat extra)
$ curl $KUBE_API/apis/apps/v1/ \                                                                                                          
  --cacert cluser-certificate-authority-data.crt  \
  --header "Authorization: Bearer $JWT_TOKEN_DEFAULT_DEFAULT"
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "apps/v1",
  "resources": [
    {
      "name": "controllerrevisions",
      "singularName": "",
      "namespaced": true,
      "kind": "ControllerRevision",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get"...]
    }]
    ...
}
```

## Here is how to create a new object using curl and a YAML manifest:
```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  --cacert cluser-certificate-authority-data.crt \
  --cert admin-client-certificate-data.crt \
  --key admin-client-key-data.key \
  -X POST \
  -H 'Content-Type: application/yaml' \
  -d '---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "365d"]
'

...
{
  "kind": "Deployment",
  "apiVersion": "apps/v1",
  "metadata": {
    "name": "sleep",
    "namespace": "default",
    "uid": "db69166f-31ac-43b4-addc-a17d79536761",
    "resourceVersion": "12029",
    "generation": 1,
    "creationTimestamp": "2022-07-06T04:00:41Z",
    "managedFields": [
      {
        "manager": "curl",
        "operation": "Update",
        "apiVersion": "apps/v1",
        "time": "2022-07-06T04:00:41Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {
          "f:spec": {
            "f:progressDeadlineSeconds": {},
            "f:replicas": {},
            "f:revisionHistoryLimit": {},
            "f:selector": {},
            "f:strategy": {...}}}}]}
}
```

## Here is how to get all objects in the default namespace:
```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  --cacert cluser-certificate-authority-data.crt \
  --cert admin-client-certificate-data.crt \
  --key admin-client-key-data.key
```

## And how to get an object by a name and a namespace:
```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert cluser-certificate-authority-data.crt \
  --cert admin-client-certificate-data.crt \
  --key admin-client-key-data.key
```


## A more advanced way of retrieving Kubernetes resources is continuously watching them for changes:
- for bash user
```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments?watch=true \
  --cacert cluser-certificate-authority-data.crt \
  --cert admin-client-certificate-data.crt \
  --key admin-client-key-data.key
```
- for zsh user
```zsh
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments\?watch=true \
  --cacert cluser-certificate-authority-data.crt \
  --cert admin-client-certificate-data.crt \
  --key admin-client-key-data.key
{"type":"ADDED","object":{"kind":"Deployment","apiVersion":"apps/v1","metadata":{"name":"sleep","namespace":"default","uid":"db69166f-31ac-43b4-addc-a17d79536761","resourceVersion":"12080","generation":1,"creationTimestamp":"2022-07-06T04:00:41Z","annotations":{"deployment.kubernetes.io/revision":"1"},"managedFields":[{"manager":"curl","operation":"Update","apiVersion":"apps/v1","time":"2022-07-06T04:00:41Z","fieldsType":"FieldsV1","fieldsV1":{"f:spec":{"f:progressDeadlineSeconds":{},"f:replicas":{},"f:revisionHistoryLimit":{},"f:selector":{},"f:strategy":{"f:rollingUpdate":{".":{},"f:maxSurge":{},"f:maxUnavailable":{}},"f:type":{}},"f:template":{"f:metadata":{"f:labels":{".":{},"f:app":{}}},"f:spec":{"f:containers":{"k:{\"name\":\"sleep\"}":{".":{},"f:command":{},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{},"f:terminationGracePeriodSeconds":{}}}}}},{"manager":"kube-controller-manager","operation":"Update","apiVersion":"apps/v1","time":"2022-07-06T04:01:07Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:annotations":{".":{},"f:deployment.kubernetes.io/revision":{}}},"f:status":{"f:availableReplicas":{},"f:conditions":{".":{},"k:{\"type\":\"Available\"}":{".":{},"f:lastTransitionTime":{},"f:lastUpdateTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}},"k:{\"type\":\"Progressing\"}":{".":{},"f:lastTransitionTime":{},"f:lastUpdateTime":{},"f:message":{},"f:reason":{},"f:status":{},"f:type":{}}},"f:observedGeneration":{},"f:readyReplicas":{},"f:replicas":{},"f:updatedReplicas":{}}},"subresource":"status"}]},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"sleep"}},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"sleep"}},"spec":{"containers":[{"name":"sleep","image":"curlimages/curl","command":["/bin/sleep","365d"],"resources":{},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","securityContext":{},"schedulerName":"default-scheduler"}},"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxUnavailable":"25%","maxSurge":"25%"}},"revisionHistoryLimit":10,"progressDeadlineSeconds":600},"status":{"observedGeneration":1,"replicas":1,"updatedReplicas":1,"readyReplicas":1,"availableReplicas":1,"conditions":[{"type":"Available","status":"True","lastUpdateTime":"2022-07-06T04:01:07Z","lastTransitionTime":"2022-07-06T04:01:07Z","reason":"MinimumReplicasAvailable","message":"Deployment has minimum availability."},{"type":"Progressing","status":"True","lastUpdateTime":"2022-07-06T04:01:07Z","lastTransitionTime":"2022-07-06T04:00:41Z","reason":"NewReplicaSetAvailable","message":"ReplicaSet \"sleep-7575b6d458\" has successfully progressed."}]}}}
```

## Here is how to update an existing object:

```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert cluser-certificate-authority-data.crt \
  --cert admin-client-certificate-data.crt \
  --key admin-client-key-data.key \
  -X PUT \
  -H 'Content-Type: application/yaml' \
  -d '---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "730d"]  # <-- Making it sleep twice longer
'
```


## Here is how to patch an existing object:
```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert cluser-certificate-authority-data.crt \
  --cert admin-client-certificate-data.crt \
  --key admin-client-key-data.key \
  -X PATCH \
  -H 'Content-Type: application/merge-patch+json' \
  -d '{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "sleep",
            "image": "curlimages/curl",
            "command": ["/bin/sleep", "1d"]
          }
        ]
      }
    }
  }
}'

```

## Last but not least - here is how to delete a collection of objects:
```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  --cacert cluser-certificate-authority-data.crt \
  --cert admin-client-certificate-data.crt \
  --key admin-client-key-data.key \
  -X DELETE
```

## And here is how to delete a single object:
```bash
$ curl $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  --cacert cluser-certificate-authority-data.crt \
  --cert admin-client-certificate-data.crt \
  --key admin-client-key-data.key \
  -X DELETE
```

---
All idea from https://iximiuz.com/en/posts/kubernetes-api-call-simple-http-client/