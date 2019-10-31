# Installing ACE/MQ into OpenShift using HELM with Minimal Permissions

The following operations need to be executed by someone with `cluster-admin` privileges:

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

## Step 3 - Create Proper Security Context

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

## Step 4 - Create a Project for ACE/MQ and allow Tiller access

Execute the following commands to create an `ace` project allow tiller to install to this project:

```
$ oc new project ace
$ oc policy add-role-to-user admin "system:serviceaccount:tiller:tiller"
```

