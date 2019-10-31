# Installing ACE/MQ into OpenShift using HELM with Minimal Permissions


## Step 1

If it doesn't already exist, create the `ibm-anyuid-scc` security context using the following template:

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
oc apply -f ./ibm-anyuid-scc.yaml
```
