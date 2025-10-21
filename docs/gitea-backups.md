# Gitea Backup and Restore

Quick reference for backing up and restoring Zarf Gitea data between k3d clusters. This example deals with two separate clusters, both deployed with the default gitea settings from the zarf init package; a single PVC mounted to the `/data` directory inside the pod. 

## Backup (Source Cluster)

### 1. Scale down Gitea
```bash
kubectl scale -n zarf deployment/zarf-gitea --replicas=0
kubectl wait --for=delete pod -l app.kubernetes.io/name=gitea -n zarf --timeout=60s
```

### 2. Find k3d container and PVC path
```bash
docker ps | grep k3d-lab-server
kubectl get pv -o yaml | grep hostPath -A 1
```

### 3. Create backup
```bash
docker exec -it k3d-lab-server-0 tar czf /tmp/gitea-backup.tar.gz \
  -C /opt/local-path-provisioner-rwx/pvc-[UUID]_zarf_data-zarf-gitea-0 .

docker cp k3d-lab-server-0:/tmp/gitea-backup.tar.gz \
  ./gitea-backup-$(date +%Y%m%d-%H%M%S).tar.gz
```

### 4. Scale up Gitea
```bash
kubectl scale -n zarf deployment/zarf-gitea --replicas=1
```

### 5. Transfer backup
```bash
scp ./gitea-backup-*.tar.gz [remote-host]:/tmp/
```

## Restore (Target Cluster)

### 1. Scale down target Gitea
```bash
kubectl scale -n zarf deployment/zarf-gitea --replicas=0
kubectl wait --for=delete pod -l app.kubernetes.io/name=gitea -n zarf --timeout=60s
```

### 2. Find target container and PVC path
```bash
docker ps | grep k3d-uds-server
kubectl get pv -o yaml | grep hostPath -A 1
```

### 3. Clear and restore data

```bash
docker cp /tmp/gitea-backup-*.tar.gz k3d-uds-server-0:/tmp/

docker exec -it k3d-uds-server-0 /bin/sh
cd /opt/local-path-provisioner-rwx/pvc-[UUID]_zarf_data-zarf-gitea-0
rm -rf ./*
tar xzf /tmp/gitea-backup-*.tar.gz
exit
```

### 4. Scale up target Gitea
```bash
kubectl scale -n zarf deployment/zarf-gitea --replicas=1
```

## Access Remote Gitea

```bash
# On remote machine
uds z connect git

# On local machine (new terminal)
ssh -L [port]:127.0.0.1:[port] -N [remote-host]
```

Then access `http://127.0.0.1:[port]` locally.

## Notes

- Backup includes SQLite database, Git repositories, and user data
- Scaling down prevents SQLite corruption during backup/restore
- Replace `[UUID]` with actual PVC UUID from kubectl output
- Replace `[port]` and `[remote-host]` with actual values