## Procedures

### Set vars and folder
```bash
{
tee /var/tmp/$(who am i |awk '{print $1}')-migration-vars <<EOF
NS="test"
OLD_NFS=nfs.old.com
NEW_NFS=new.nfs.com
POD_LABEL=rsync-pod
DIR="oc-conf-\$NS"
GRACE_PERIOD=0
EOF
}
```


### RSYNC Base copy 
```bash
{
test -f /var/tmp/$(who am i |awk '{print $1}')-migration-vars && {
. /var/tmp/$(who am i |awk '{print $1}')-migration-vars
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rsync-${NS}
  namespace: ${NS}
  labels:
    type: ${POD_LABEL} 
spec:
  containers:
  - name: app
    image: registry.redhat.io/openshift4/ose-cli
    command: ["/bin/sh", "-c", "sleep infinity"]
    volumeMounts:
    - name: new-nfs
      mountPath: /new-nfs
    - name: old-nfs 
      mountPath: /old-nfs
  volumes:
  - name: new-nfs
    nfs:
      server: ${NEW_NFS}
      path: /oc/exports/${NS}
  - name: old-nfs
    nfs:
      server: ${OLD_NFS}
      path: /oc/exports/${NS}
EOF
oc wait --for=condition=Ready -n $NS pod/rsync-$NS && oc exec rsync-$NS -n $NS -- rsync -avP /old-nfs/ /new-nfs/
}
}

```

### Stop Applications (Deployments/Statefulsets/Pod/hpa)
```bash
{
. /var/tmp/$(who am i |awk '{print $1}')-migration-vars
mkdir -p $DIR/pv $DIR/pvc
for TYPE in {job,dc,deploy,sts,hpa,cronjob}; do
RSC_COUNT=$(oc get $TYPE --no-headers 2>/dev/null | wc -l)
if [ "$RSC_COUNT" -gt 0 ];then
oc get $TYPE -n $NS -o json | jq  '.items[] | del(.metadata.uid,.metadata.creationTimestamp,.metadata.resourceVersion,.status)' | oc create -n $NS -f - --dry-run=client -o yaml > $DIR/$TYPE.yaml
fi
done
NKD_PODS=$(oc get pods -n $NS -o json | jq -r '.items[] | select(.metadata.ownerReferences == null) | .metadata.name' 2>/dev/null | wc -l)
if [ "$NKD_PODS" -gt 0 ];then
oc get pods -n $NS -l "type!=$POD_LABEL" -o json | jq '.items[] | select(.metadata.ownerReferences == null) | del(.metadata.uid, .metadata.resourceVersion, .metadata.creationTimestamp, .metadata.selfLink, .metadata.managedFields, .metadata.annotations["k8s.v1.cni.cncf.io/network-status"], .metadata.annotations["k8s.ovn.org/pod-networks"], .spec.nodeName, .status)' | oc create -f - --dry-run=client -o yaml > "$DIR/naked-pods.yaml"
fi
oc scale dc,deploy,sts --replicas=0 --all -n $NS
oc exec rsync-${NS} -n ${NS} -- rsync -avP --delete /old-nfs/ /new-nfs/
oc delete hpa --all -n $NS
oc get cronjob -n $NS -o name | xargs -I {} oc patch {} -p '{"spec" : {"suspend" : true}}' -n $NS # xargs -n1 oc patch ???
oc delete jobs --all -n $NS
test -f $DIR/naked-pods.yaml && oc delete -f $DIR/naked-pods.yaml --grace-period=$GRACE_PERIOD
(oc wait --for=delete pod -l "type!=$POD_LABEL" --timeout=120s -n $NS && echo "You can continue") || echo "Check your pods"
}
```


### Copy old pv/pvc , renmame, recreate the new one
```bash
{
. /var/tmp/$(who am i |awk '{print $1}')-migration-vars
PVS=$(oc get pvc -n $NS -o json | jq -r '.items[] | select(.spec.storageClassName == null or .spec.storageClassName == "") | [.metadata.name,.spec.volumeName] | @tsv')
while read -r pvc pv;do
[[ -z "$pvc" ]] && continue
echo "Patching PV $pv to Retain policy..."
oc patch pv "$pv" -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
oc get pv "$pv" -o json | jq 'del(.metadata.uid,.metadata.resourceVersion,.metadata.creationTimestamp,.metadata.selfLink,.metadata.managedFields,.metadata.annotations["pv.kubernetes.io/bound-by-controller"],.spec.claimRef,.status)' | oc create -f - --dry-run=client -o yaml > $DIR/pv/$pv.yaml
oc get pvc "$pvc" -n $NS -o json | jq 'del(.metadata.uid,.metadata.creationTimestamp,.metadata.annotations,.status,.metadata.resourceVersion)' | oc create -f - --dry-run=client -o yaml > $DIR/pvc/$pv.yaml
oc delete pvc "$pvc" -n $NS --wait=true
oc delete pv "$pv" --wait=true
done <<< "$PVS"
find $DIR/pv/ -type f -exec sed -i "s/server: $OLD_NFS/server: $NEW_NFS/g" {} \;
oc apply -f $DIR/pv/ 
oc apply -f $DIR/pvc/ 
}
```



### Restart all resources
```bash
{
. /var/tmp/$(who am i |awk '{print $1}')-migration-vars
for TYPE in {sts,dc,deploy,hpa,cronjob}; do
test -f  $DIR/$TYPE.yaml && oc apply -f $DIR/$TYPE.yaml -n $NS
done

#Reapply pods
test -f $DIR/naked-pods.yaml && oc apply -f $DIR/naked-pods.yaml
}
```

