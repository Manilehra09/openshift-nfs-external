# OpenShift SNO — External NFS Storage Setup

## Overview
This guide configures **OpenShift Single Node (SNO)** to use an **external NFS server** as RWX storage.

---

## 1️⃣ Install NFS on external server

### RHEL / Fedora / CentOS
```bash
sudo dnf install -y nfs-utils
```

### Ubuntu / Debian
```bash
sudo apt install -y nfs-kernel-server
```

---

## 2️⃣ Create export directory
```bash
sudo mkdir -p /srv/nfs/data
sudo chmod 777 /srv/nfs/data
```

---

## 3️⃣ Configure NFS export
Edit:
```bash
sudo nano /etc/exports
```

Add:
```
/srv/nfs/data *(rw,sync,no_root_squash,no_subtree_check)
```

Apply:
```bash
sudo exportfs -rav
```

Verify:
```bash
showmount -e
```

---

## 4️⃣ Start and enable NFS service
```bash
sudo systemctl enable --now nfs-server
```

---

## 5️⃣ Firewall (optional)
```bash
sudo firewall-cmd --add-service=nfs --permanent
sudo firewall-cmd --reload
```

---

## 6️⃣ Create PersistentVolume on OpenShift

**pv-nfs.yaml**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /srv/nfs/data
    server: <NFS_SERVER_IP>
```

Apply:
```bash
oc apply -f pv-nfs.yaml
```

---

## 7️⃣ PVC (Static — do NOT set StorageClass)

**pvc-nfs.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs
  namespace: default
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
  volumeName: pv-nfs
```

Apply:
```bash
oc apply -f pvc-nfs.yaml
```

---

## 8️⃣ Test Deployment

**nfs-test.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-test
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-test
  template:
    metadata:
      labels:
        app: nfs-test
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'sleep 3600']
        volumeMounts:
        - name: test-data
          mountPath: /mnt/data
      volumes:
      - name: test-data
        persistentVolumeClaim:
          claimName: pvc-nfs
```

---

## Verify
```bash
oc exec -it deploy/nfs-test -- sh
cd /mnt/data
touch hello.txt
ls -l
```

Then check on NFS server:
```bash
ls -l /srv/nfs/data
```
