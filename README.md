# Container Storage Interface (CSI) Driver for SSHFS

This repository contains a CSI driver for SSHFS.

This is a fork of the original [`chr-fritz/csi-sshfs`](https://github.com/chr-fritz/csi-sshfs) CSI
driver that has been revamped and rewritten to work on Kubernetes 1.22 and later. Both the 
`deploy/kubernetes` and `deploy/terraform` folder can be used as a guide to the proper new 
deployment structure.

## Known Issues

- The `deploy/kubernetes-debug` folder wasn't updated. Its contents will definitely fail on recent 
  versions of kubernetes.
- The `volumeHandle` must be unique per `PersistentVolume`.
- Expect to have file ownership/permissions weirdness, as SSHFS doesn't handle that for you.
- The original driver was only a proof of concept. YMMV in a production environment.

## Usage

Deploy the whole directory `deploy/kubernetes`.
This installs the csi controller and node plugin and an appropriate storage class for the csi driver.
```bash
kubectl apply -f deploy/kubernetes
```

To use the csi driver create a persistent volume and persistent volume claim like the example one:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-sshfs
  labels:
    name: data-sshfs
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 100Gi
  storageClassName: sshfs
  csi:
    driver: co.p4t.csi.sshfs
    volumeHandle: data-id
    volumeAttributes:
      server: "<HOSTNAME|IP>"
      port: "22"
      share: "<PATH_TO_SHARE>"
      privateKey: "<NAMESPACE>/<SECRET_NAME>"
      user: "<SSH_CONNECT_USERNAME>"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-sshfs
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: sshfs
  selector:
    matchLabels:
      name: data-sshfs
```

Then mount the volume into a pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx 
spec:
  containers:
  - image: maersk/nginx
    imagePullPolicy: Always
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    volumeMounts:
      - mountPath: /var/www
        name: data-sshfs
  volumes:
  - name: data-sshfs
    persistentVolumeClaim:
      claimName: data-sshfs
```

```
TODO:
add more things from the spec: https://github.com/container-storage-interface/spec/releases
Ensure everything is idempotent. check first if things exist before creating?
figure out what capabilities to report

plan adding controller support for making new volumes in subfolders
https://github.com/kubernetes-csi/csi-driver-nfs/issues/70#issuecomment-714845727
https://github.com/kubernetes-csi/livenessprobe
https://github.com/kubernetes-csi/external-snapshotter
https://github.com/kubernetes-csi/external-resizer
https://github.com/kubernetes-csi/external-health-monitor

https://kubernetes.io/blog/2020/12/18/kubernetes-1.20-pod-impersonation-short-lived-volumes-in-csi/
https://kubernetes.io/blog/2020/01/08/testing-of-csi-drivers/
https://kubernetes.io/blog/2020/01/21/csi-ephemeral-inline-volumes/
```
