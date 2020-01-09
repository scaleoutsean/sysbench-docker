# sysbench on Docker #

Sysbench 1.0.17 in a container environment.

## Installation Procedure

- Install a recent version of K8s (no special features are required)

```
# kubectl get nodes -o wide
NAME                     STATUS   ROLES    AGE   VERSION         INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
ubuntu1804.localdomain   Ready    master   53d   v1.16.3-k3s.1   10.0.2.15     <none>        Ubuntu 18.04.3 LTS   4.15.0-70-generic   containerd://1.3.0-k3s.4

```

- Install NetApp Trident v19.10 and configure a backend such as SolidFire
- Use the same namespace for all pods (examples below use namespace sysbench, but you can use any, just stick with it)
- Ensure you have a SC that uses your back-end array, such as below

```

# kubectl get sc

Name:                  basic
IsDefaultClass:        No
Annotations:           <none>
Provisioner:           csi.trident.netapp.io
Parameters:            IOPS=200,backendType=solidfire-san,fsType=xfs
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     Immediate
Events:                <none>
```

- Create a PVC to use RWO volume based on your storage class

```
# cat pvc-sysbench.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: basic
```

- Now create MySQL pod that uses this PVC. Below MySQL root password is set to "password"

```
# cat sysbench-create-mysql.yaml 
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

- Next we create a DB in this MySQL pod. The quickest way is to install mysql-client on host and connect to MySQL pod's IP address:

```
# mysql --host=10.42.0.144 --user=root --password
```

- Create stuff:

```sql
mysql> CREATE SCHEMA sbtest;
mysql> CREATE USER sbtest@'%' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON sbtest.* to sbtest@'%';
mysql> END;
```

- Fill the DB with data. Note that this table and DB is very small - feel free to increase oltp-table-size, oltp-tables-count, threads. This container may run a while and then it stop/exit.

```
# cat sysbench-prep-mysql.yaml 
apiVersion: batch/v1
kind: Job
metadata:
  name: sysbench-prepare
spec:
  template:
    metadata:
      name: sysbench-prepare
    spec:
      containers:
      - name: sysbench-prepare
        image: severalnines/sysbench
        command:
        - sysbench
        - --db-driver=mysql
        - --oltp-table-size=1000
        - --oltp-tables-count=2
        - --threads=1
        - --mysql-host=10.42.0.144
        - --mysql-port=3306
        - --mysql-user=root
        - --mysql-password=password
        - /usr/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua
        - run
      restartPolicy: Never
```

- Once the test DB has been populated with data we can run sysbench by creating a sysbench workload pod in the same namespace. Feel free to increase oltp-table-size, oltp-tables-count, threads, but not to a higher value than above. Use `sbtest` if your DB is named as per above SQL example. Use mySQL pod IP for hostname if you don't have DNS/FQDN for it.

```
# cat sysbench-run-mysql.yaml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: sysbench
  name: sysbench
spec:
  containers:
  - command:
    - sysbench
    - --db-driver=mysql
    - --report-interval=2
    - --mysql-table-engine=innodb
    - --oltp-table-size=1000
    - --oltp-tables-count=2
    - --threads=4
    - --time=99999
    - --mysql-host=10.42.0.144
    - --mysql-port=3306
    - --mysql-user=sbtest
    - --mysql-password=password
    - /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua
    - run
    image: severalnines/sysbench
    name: sysbench
  restartPolicy: Never

```

## Example from above:

### PVC & PV 

MySQL is utilizing PVC `27ed`

```
root@ubuntu1804:~# kubectl get pvc -n sysbench
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pvc   Bound    pvc-fc1fced2-a73b-4b9c-87ac-b19a6abc27ed   1Gi        RWO            basic          99m
root@ubuntu1804:~# kubectl get pv -n sysbench
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
pvc-10dad67d-2ebb-44ac-9769-242cc9ecac75   1Gi        RWO            Delete           Bound    default/pvc1          basic                   53d
pvc-09acaccf-e8e2-447b-8a70-e464569ff17c   1Gi        RWO            Delete           Bound    sean/pvc1-from-snap   basic                   53d
pvc-b87f2696-1608-4b16-9a9e-ffc6b292670a   1Gi        RWO            Delete           Bound    default/pvc1-test2    basic                   53d
pvc-fc1fced2-a73b-4b9c-87ac-b19a6abc27ed   1Gi        RWO            Delete           Bound    sysbench/mysql-pvc    basic                   99m

```
### Pods

- Note that mySQL IP is 10.42.0.144, sysbench workload generator connects to that IP:

```
root@ubuntu1804:~# kubectl get pods -n sysbench
NAME                    READY   STATUS    RESTARTS   AGE
mysql-dbb9f67f9-2xsc7   1/1     Running   0          96m
sysbench                1/1     Running   0          52m

root@ubuntu1804:~# kubectl get pods -o wide -n sysbench
NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE                     NOMINATED NODE   READINESS GATES
mysql-dbb9f67f9-2xsc7   1/1     Running   0          98m   10.42.0.144   ubuntu1804.localdomain   <none>           <none>
sysbench                1/1     Running   0          54m   10.42.0.154   ubuntu1804.localdomain   <none>           <none>

root@ubuntu1804:~# kubectl describe pod mysql-dbb9f67f9-2xsc7 -n sysbench
Name:         mysql-dbb9f67f9-2xsc7
Namespace:    sysbench
Priority:     0
Node:         ubuntu1804.localdomain/10.0.2.15
Start Time:   Thu, 09 Jan 2020 14:24:53 +0000
Labels:       app=mysql
              pod-template-hash=dbb9f67f9
Annotations:  <none>
Status:       Running
IP:           10.42.0.144
IPs:
  IP:           10.42.0.144
Controlled By:  ReplicaSet/mysql-dbb9f67f9
Containers:
  mysql:
    Container ID:   containerd://15d90b9abcaa28a35df30d5a0b59ea94b76e2af4bb4a87f223c840c3ea414b2a
    Image:          mysql:5.6
    Image ID:       docker.io/library/mysql@sha256:82a505551c0243ca04df445f1287b2c4da3b23463b1a9c0bc2b2476760179950
    Port:           3306/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 09 Jan 2020 14:25:58 +0000
    Ready:          True
    Restart Count:  0
    Environment:
      MYSQL_ROOT_PASSWORD:  password
    Mounts:
      /var/lib/mysql from mysql-persistent-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-mvvjx (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  mysql-persistent-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  mysql-pvc
    ReadOnly:   false
  default-token-mvvjx:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-mvvjx
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>

root@ubuntu1804:~# kubectl describe pod sysbench -n sysbench
Name:         sysbench
Namespace:    sysbench
Priority:     0
Node:         ubuntu1804.localdomain/10.0.2.15
Start Time:   Thu, 09 Jan 2020 15:08:46 +0000
Labels:       app=sysbench
Annotations:  <none>
Status:       Running
IP:           10.42.0.154
IPs:
  IP:  10.42.0.154
Containers:
  sysbench:
    Container ID:  containerd://dd3144967b4dfc5321e595995b8cdc6058a276f19704926f3bba50ce930a701e
    Image:         severalnines/sysbench
    Image ID:      docker.io/severalnines/sysbench@sha256:64cd003bfa21eaab22f985e7b95f90d21a970229f5f628718657dd1bae669abd
    Port:          <none>
    Host Port:     <none>
    Command:
      sysbench
      --db-driver=mysql
      --report-interval=2
      --mysql-table-engine=innodb
      --oltp-table-size=1000
      --oltp-tables-count=2
      --threads=4
      --time=99999
      --mysql-host=10.42.0.144
      --mysql-port=3306
      --mysql-user=sbtest
      --mysql-password=password
      /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua
      run
    State:          Running
      Started:      Thu, 09 Jan 2020 15:08:53 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-mvvjx (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-mvvjx:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-mvvjx
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From                             Message
  ----    ------     ----       ----                             -------
  Normal  Scheduled  <unknown>  default-scheduler                Successfully assigned sysbench/sysbench to ubuntu1804.localdomain
  Normal  Pulling    53m        kubelet, ubuntu1804.localdomain  Pulling image "severalnines/sysbench"
  Normal  Pulled     53m        kubelet, ubuntu1804.localdomain  Successfully pulled image "severalnines/sysbench"
  Normal  Created    53m        kubelet, ubuntu1804.localdomain  Created container sysbench
  Normal  Started    53m        kubelet, ubuntu1804.localdomain  Started container sysbench

```

