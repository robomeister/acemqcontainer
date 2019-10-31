# Installing ACE/MQ into OpenShift using HELM with Minimal Permissions

Steps 1 through 4 need to be executed by someone logged into OpenShift with `cluster-admin` privileges:

## Step 1 - Install Tiller

Install Tiller into it's own namespace:

```
$ oc new project tiller
$ export TILLER_NAMESPACE=tiller
$ oc process -f https://github.com/openshift/origin/raw/master/examples/helm/tiller-template.yaml -p TILLER_NAMESPACE="${TILLER_NAMESPACE}" -p HELM_VERSION=v2.14.1 | oc create -f -
```

This template will create the service account `system:serviceaccount:tiller:tiller`

## Step 2 - Assign Permissions to Tiller that will enable ACE/MQ install

The ACE/MQ Helm chart creates an MQ service account and sets it up with a cluster role binding.  To allow tiller to execute these operations, it needs some extra permissions.  

```
apiVersion: authorization.openshift.io/v1
kind: ClusterRole
metadata:
  name: tiller
rules:
- apiGroups:
  - ""
  - rbac.authorization.k8s.io
  resources:
  - clusterroles
  - secrets
  verbs:
  - '*'
---
apiVersion: authorization.openshift.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  name: tiller
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: tiller
userNames:
- system:serviceaccount:tiller:tiller
```

Copy the above text into `tiller-roles.yaml` and execute the following command:

`oc apply -f ./tiller-roles.yaml`

Please note these permissions satisfy the requirements to install the ACE/MQ container only.  Other helm charts may require the same, more or less permissions.  Most organizations simply assign the `cluster-admin` role to the tiller service account.

If desired, once the install has completed, the permissions can be removed.  Please note, however, that when a helm update or delete is to occur, the permissions will need to be added again.

## Step 3 - Create new Security Context

If it doesn't yet already exist, create the `ibm-anyuid-scc` security context using the following template:

```
kind: SecurityContextConstraints
apiVersion: security.openshift.io/v1
metadata:
  annotations:
    cloudpak.ibm.com/version: 1.0.0
    kubernetes.io/description: This policy allows pods to run with any UID and GID, but preventing access to the host.
  name: ibm-anyuid-scc
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities:
- SETPCAP
- AUDIT_WRITE
- CHOWN
- NET_RAW
- DAC_OVERRIDE
- FOWNER
- FSETID
- KILL
- SETUID
- SETGID
- NET_BIND_SERVICE
- SYS_CHROOT
- SETFCAP
defaultAddCapabilities: null
defaultAllowPrivilegeEscalation: true
forbiddenSysctls:
- '*'
fsGroup:
  type: RunAsAny
priority: null
readOnlyRootFilesystem: false
requiredDropCapabilities:
- MKNOD
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
```

Copy the text into a file named `ibm-anyuid-scc.yaml` and execute the following command:

```
$ oc apply -f ./ibm-anyuid-scc.yaml
```

## Step 4 - Create a Project for ACE/MQ, allow Tiller access, and set the Security Context

Execute the following commands to create an `ace` project and allow tiller to install to this project:

```
$ oc new-project ace
$ oc policy add-role-to-user admin "system:serviceaccount:tiller:tiller"
$ oc adm policy add-scc-to-group ibm-anyuid-scc system:serviceaccounts:ace
```

The environment is now ready.  The rest of the following commands can be executed by the end user:

## Step 5 - Install Helm client onto Workstation

You can download the helm client for Windows and Linux here:

```
https://get.helm.sh/helm-v2.14.1-windows-amd64.zip
https://get.helm.sh/helm-v2.14.1-linux-amd64.tar.gz
```

Extract the `helm.exe` or `helm` binary and place it in an executable path.

You will also need the oc command line tool, you can download it for Windows and Linux here:

```
https://github.com/openshift/origin/releases/download/v3.11.0/openshift-origin-client-tools-v3.11.0-0cbc58b-windows.zip
https://github.com/openshift/origin/releases/download/v3.11.0/openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz
```

Extract the `oc.exe` and `kubectl.exe` or `oc` and `kubectl` executables and place them in an executable path.

Once you have the tools installed, log into OpenShift on the command line and verify the helm and tiller install:

For Windows:

```
 oc login <openshift endpoint> --username=username --password=password
 set TILLER_NAMESPACE=tiller
 helm version
 helm init --client-only
```

For Linux:

```
$ oc login <openshift endpoint> --username=username --password=password
$ export TILLER_NAMESPACE=tiller
$ helm version
$ helm init --client-only
```

You should see the following:

```
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
```

### Step 6 - Add the ibm-charts repo and fetch the ACE/MQ helm chart

From you command line, type the following:

```
helm repo add ibm-charts https://icr.io/helm/ibm-charts
helm repo update
helm fetch ibm-charts/ibm-ace-server-dev
```

Decompress the `ibm-ace-server-dev-2.x.0.tgz` file and copy the ibm-ace-server-dev/values.yaml file up one level.

### Step 7 - Update values.yaml

Inside the values.yaml file, update the following properties:

```
license: "accept"
imageType: acemqserver
dashboardEnabled: false
```
We disable the monitoring dashboard, as it is Cloud Pak dependant and we are install ACE/MQ on its own.

If your contianer images are available locally, then update the following properties:

```
image:
  aceonly: ibmcom/ace:11.0.0.5.1
  acemqclient: ibmcom/ace-mqclient:11.0.0.5.1
  acemq: ibmcom/ace-mq:11.0.0.5.1
  configurator: ibmcom/ace-icp-configurator:11.0.0.5.1
  designerflows: ibmcom/ace-designer-flows:11.0.0.5.1
  pullPolicy: IfNotPresent
  pullSecret:
```

Finally, you will want to update the storage class name property to the dynamic storage class that will create the persistant storage for MQ:

```
storageClassName: ""
```

### Step 8 - Install ACE/MQ

Perform the following commands:

```
oc project ace
set TILLER_NAMESPACE=tiller
helm install -n ace -f ./values.yaml ./ibm-ace-server-dev
```

This assumes your customized `values.yaml` file is in the local directory and the rest of the helm chart files are in `ibm-ace-server-dev`

You should then see something like this:

```
NAME:   ace
LAST DEPLOYED: Thu Oct 31 11:12:26 2019
NAMESPACE: ace
STATUS: DEPLOYED

RESOURCES:
==> v1/Role
NAME                          AGE
ace-ibm-ace-server-dev-role  1s

==> v1/RoleBinding
NAME                                 AGE
ace-ibm-ace-server-dev-rolebinding  1s

==> v1/Service
NAME                                 TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)                                                                     AGE
ace-ibm-ace-server-dev              NodePort   172.21.201.76  <none>       7600:32606/TCP,7800:30248/TCP,7843:30251/TCP,9443:30392/TCP,1414:32764/TCP  1s
ace-ibm-ace-server-dev-ace-metrics  ClusterIP  172.21.9.144   <none>       9483/TCP                                                                    1s
ace-ibm-ace-server-dev-mq-metrics   ClusterIP  172.21.62.250  <none>       9157/TCP                                                                    1s

==> v1/ServiceAccount
NAME                                    SECRETS  AGE
ace-ibm-ace-server-dev-serviceaccount  2        1s

==> v1/StatefulSet
NAME                     READY  AGE
ace-ibm-ace-server-dev  0/1    1s


NOTES:

If you launched the deploy from the ACE Dashboard, then you can return to the ACE Dashboard to manage the server.

The HTTP and HTTPS endpoints for the ACE Integration Server are exposed with a NodePort by default.

export ACE_NODE_IP=$(kubectl get configmap -n kube-public ibmcloud-cluster-info   -o jsonpath="{.data.proxy_address}")
export ACE_HTTP_PORT=$(kubectl get service ace2-ibm-ace-server-dev --namespace ace2 -o jsonpath="{.spec.ports[1].nodePort}")
export ACE_HTTPS_PORT=$(kubectl get service ace2-ibm-ace-server-dev --namespace ace2 -o jsonpath="{.spec.ports[2].nodePort}")
export ACE_MQ_PORT=$(kubectl get service ace2-ibm-ace-server-dev --namespace ace2 -o jsonpath="{.spec.ports[3].nodePort}")

echo "HTTP workload can use: http://${ACE_NODE_IP}:${ACE_HTTP_PORT}"
echo "HTTPS workload can use: https://${ACE_NODE_IP}:${ACE_HTTPS_PORT}"
echo "MQ workload can use: ${ACE_NODE_IP}:${ACE_MQ_PORT}"
```

