# crc-volume

## update April 2024

Original Link: https://developers.redhat.com/articles/2022/04/20/create-and-manage-local-persistent-volumes-codeready-containers

Added:
- relogin inside nfs container
- add no timeout on SSH

### Note: This should be done on the VM, not remote SSH.

```
# Login as kubeadmin
export sec=xxxxxxxxxxxxxxx
eval $(crc oc-env) &&  oc login -u kubeadmin -p $sec https://api.crc.testing:6443 
oc adm policy add-cluster-role-to-user cluster-admin kubeadmin --as system:admin

# Create a new namespace
oc new-project nfsprovisioner-operator

cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nfs-provisioner-operator
  namespace: openshift-operators
spec:
  channel: alpha
  installPlanApproval: Automatic
  name: nfs-provisioner-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
EOF


export target_node=$(oc get node --no-headers -o name|cut -d'/' -f2)
oc label node/${target_node} app=nfs-provisioner

oc debug node/${target_node}

#disable timeout
cat << 'EOF' > /etc/ssh/sshd_config
TCPKeepAlive no 
ClientAliveInterval 30
ClientAliveCountMax 240
EOF

chroot /host
mkdir -p /home/core/nfs
chcon -Rvt svirt_sandbox_file_t /home/core/nfs
exit; exit

#login inside
export sec=xxxxxxxxxxxxxxx
oc login -u kubeadmin -p $sec https://api.crc.testing:6443 

cat << EOF | oc apply -f -
apiVersion: cache.jhouse.com/v1alpha1
kind: NFSProvisioner
metadata:
  name: nfsprovisioner-sample
  namespace: nfsprovisioner-operator
spec:
  nodeSelector:
    app: nfs-provisioner
  hostPathDir: "/home/core/nfs"
EOF


oc patch storageclass nfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
oc get sc

oc project default


cat << EOF | oc apply -f -
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs
EOF


oc get pv, pvc
oc project nfsprovisioner-operator
```
