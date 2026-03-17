#OpenEBS

mkfs.xfs -L openebslocal /dev/vdb
echo 'LABEL=openebslocal /var/openebs/local xfs defaults,pquota 0 0' >> /etc/fstab
cat /etc/fstab
mount -a
systemctl daemon-reload


https://openebs.io/docs/main/Solutioning/openebs-on-kubernetes-platforms/openshift
https://openebs.io/docs/main/Solutioning/read-write-many/nfspvc


helm upgrade --install openebs --namespace openebs openebs/openebs --create-namespace --set engines.replicated.mayastor.enabled=false --set alloy.enabled=false --set loki.enabled=false --set obs.callhome.enabled=false --set openebs-crds.csi.volumeSnapshots.enabled=false --set engines.local.zfs.enabled=false 

--set hostpathClass.xfsQuota.enabled=true --set hostpathClass.xfsQuota.softLimitGrace=0% --set hostpathClass.xfsQuota.hardLimitGrace=0%

oc adm policy add-scc-to-user privileged system:serviceaccount:openebs:openebs-lvm-node-sa
oc adm policy add-scc-to-user privileged system:serviceaccount:openebs:openebs-localpv-provisioner
oc adm policy add-scc-to-user privileged system:serviceaccount:openebs:openebs-service-account
oc adm policy add-scc-to-user privileged system:serviceaccount:openebs:default

#ggf
oc auth can-i use scc/privileged --as system:serviceaccount:openebs:openebs-localpv-provisioner

oc rollout restart deployment/openebs-localpv-provisioner -n openebs

#test
helm upgrade --install openebs \
  --namespace openebs \
  openebs/openebs \
  --create-namespace \
  --set engines.replicated.mayastor.enabled=false \
  --set alloy.enabled=false \
  --set loki.enabled=false \
  --set obs.callhome.enabled=false \
  --set openebs-crds.csi.volumeSnapshots.enabled=false \
  --set engines.local.zfs.enabled=false \
  --set localprovisioner.basePath=/var/openebs/local \
  --set hostpathClass.basePath=/var/openebs/local \
  --set hostpathClass.xfsQuota.enabled=true \
  --set hostpathClass.xfsQuota.softLimitGrace=0% \
  --set hostpathClass.xfsQuota.hardLimitGrace=0%