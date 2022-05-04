<div align="center">
<h1>Volume Snapshot Mover</h1>

<h2>A Data Mover for CSI snapshots</h2>
</div>

VolumeSnapshotMover relocates snapshots off of the cluster into an object store to be used during a restore process to recover stateful applications 
in instances such as cluster deletion or disaster. 

#### Design Proposal (in-progress): https://github.com/openshift/oadp-operator/pull/597/files

# Table of Contents

1. [Getting Started](#pre-reqs)
2. Running the Controller:
    1. [Backup](#backup)
    2. [Restore](#restore)


<h2>Prerequisites:<a id="pre-reqs"></a></h2>

<hr style="height:1px;border:none;color:#333;">

To use the data mover controller, you will first need a volumeSnapshot. This can be achieved
by using the Velero CSI plugin during backup of the stateful application.

- Install OADP Operator using [these steps](https://github.com/openshift/oadp-operator/blob/master/docs/install_olm.md).

- Have a stateful application running in a separate namespace. 

- Create a Velero backup using CSI snapshotting following the steps specified [here](https://github.com/openshift/oadp-operator/blob/master/docs/examples/csi_example.md).

- [Install](https://volsync.readthedocs.io/en/stable/installation/index.html) the VolSync controller.

- We will be using VolSync's Restic option, hence configure a [restic secret](https://volsync.readthedocs.io/en/stable/usage/restic/index.html#id2)

- Install the VolumeSnapshotMover CRDs `DataMoverBackup` and `DataMoverRestore` using: `oc create -f config/crd/bases/`

### Run the controller:

<hr style="height:1px;border:none;color:#333;">

<h4> For backup: <a id="backup"></a></h4>

- Create a Restic secret named `restic-secret` in the protected namespace, following the above steps.

- Create a `DataMoverBackup` CR in the app namespace. This will create several resources in the protected namespace, such as a PVC, volumeSnapshot, volumeSnapshotContent, and a Volsync `ReplicationSource`.
  - Use the `volumeSnapshotContent` name that was created during the Velero backup using CSI.
    This is the snapshot that will be moved to object storage:

```
apiVersion: pvc.oadp.openshift.io/v1alpha1
kind: DataMoverBackup
metadata:
  name: datamoverbackup-sample
spec:
  volumeSnapshotContent:
    name: <VOLUMESNAPSHOTCONTENT_NAME>
```

- Run the controller by executing `make run`

<h4> For restore: <a id="restore"></a></h4>

- Create a Restic secret named `restic-secret` in the protected namespace (if this no longer exists) following the above steps.

- Create an empty PVC in the protected namespace with the name as the PVC used by DMB.

- Create a `DataMoverRestore` CR. This will create a VolSync `ReplicationDestination` which will then create a new 
volumeSnapshot and volumeSnapshotContent in the cluster.
  - *Note:* in the future this will be created by the controller.

```
apiVersion: pvc.oadp.openshift.io/v1alpha1
kind: DataMoverRestore
metadata:
  name: datamoverrestore-sample
spec:
  resticSecretRef: 
    name: restic-secret
  destinationClaimRef: 
    name: <PVC_NAME>
    namespace: <APP_NS>
```

- Run the controller by executing `make run`

- To complete the restore process, create a Velero restore.
  Make sure `restorePVs` is set to `true`.

```
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: <Restore_Name>
  namespace: <Protected_NS>
spec:
  backupName: <Backup_From_CSI_Steps>
  restorePVs: true
```
