# nfs-dynamic-provisioner-openshift
Create project
```bash
oc new-project nfs-provisioner
```
Create provisioner
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          # you can bump version if needed
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: cluster.local/nfs-client
            - name: NFS_SERVER
              value: "192.168.122.24"     # <-- change to your NFS server IP
            - name: NFS_PATH
              value: "/srv/ocp-nfs"      # <-- change to your export path
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.122.24        # <-- same as above
            path: /srv/ocp-nfs

```
Create storageclass
```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: cluster.local/nfs-client   # must match PROVISIONER_NAME above
parameters:
  archiveOnDelete: "true"
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```
Set as default storageclass
```bash
oc patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
Create permissions
```bash
oc adm policy add-scc-to-user hostmount-anyuid -z nfs-client-provisioner -n nfs-provisioner
oc -n nfs-provisioner create role nfs-provisioner-endpoints   --verb=get,list,watch,create,update,patch   --resource=endpoints
oc -n nfs-provisioner create rolebinding nfs-provisioner-endpoints   --role=nfs-provisioner-endpoints   --serviceaccount=nfs-provisioner:nfs-client-provisioner
```
Create permissions as well
```bash
oc apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get","list","watch","create","delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get","list","watch","update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create","update","patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get","list","watch","create","update","patch"]
EOF

oc apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: run-nfs-client-provisioner
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nfs-client-provisioner-runner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: nfs-provisioner
EOF
```
Create pvc
```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs-pvc
  namespace: nfs-provisioner
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-client
```
Check the pvc status if it is bound
```bash
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
test-nfs-pvc   Bound    pvc-97dc2ef0-7d4b-413c-a8f3-a4705a0440a9   1Gi        RWX            nfs-client     <unset>                 5m22s
```

