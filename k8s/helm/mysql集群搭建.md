# mysql集群部署



### 方案一：使用helm

1、Add repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```



2、download chart

```shell
helm fetch bitnami/mysql
```

3、解压 

```shell
tar -zxvf mysql-8.8.20.tgz
```

4、默认启动 

```shell
helm install baichuan-mysql -n ha mysql # 此处的ha表示提前创建号的namespace
```

out

```shell
** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace ha

Services:

  echo Primary: baichuan-mysql.ha.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace ha baichuan-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run baichuan-mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.27-debian-10-r63 --namespace ha --command -- bash

  2. To connect to primary service (read/write):

      mysql -h baichuan-mysql.ha.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"



To upgrade this helm chart:

  1. Obtain the password as described on the 'Administrator credentials' section and set the 'root.password' parameter as shown below:

      ROOT_PASSWORD=$(kubectl get secret --namespace ha baichuan-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)  
      
      helm upgrade --namespace ha baichuan-mysql bitnami/mysql --set auth.rootPassword=$ROOT_PASSWORD
```


##### Plan b

```shell
helm install bc-mysql -n ha mysql \
--set architecture=replication \
--set auth.rootPassword='tsl@2018' \
--set primary.service.type=NodePort \
--set primary.service.nodePort=43308 \
--set secondary.replicaCount=2 \
--set secondary.service.type=NodePort \
--set secondary.service.nodePort=43307 
```

out

```shell
** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace ha

Services:

  echo Primary: bc-mysql-primary.ha.svc.cluster.local:3306
  echo Secondary: bc-mysql-secondary.ha.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace ha bc-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run bc-mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.27-debian-10-r63 --namespace ha --command -- bash

  2. To connect to primary service (read/write):

      mysql -h bc-mysql-primary.ha.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"

  3. To connect to secondary service (read-only):

      mysql -h bc-mysql-secondary.ha.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"



To upgrade this helm chart:

  1. Obtain the password as described on the 'Administrator credentials' section and set the 'root.password' parameter as shown below:

      ROOT_PASSWORD=$(kubectl get secret --namespace ha bc-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)
      helm upgrade --namespace ha bc-mysql bitnami/mysql --set auth.rootPassword=$ROOT_PASSWORD
```



### 方案二：自行搭建success

1、configmap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  namespace: ha
  labels:
    app: mysql
data:
  master.cnf: |
    # 主节点MySQL的配置文件
    [mysqld]
    log-bin
  slave.cnf: |
    # 从节点MySQL的配置文件
    [mysqld]
    super-read-only
```

2、service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: ha
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  namespace: ha
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```

statefulset

```yaml
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: mysql
  namespace: ha
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: mysql
    spec:
      volumes:
        - name: conf
          emptyDir: {}
        - name: config-map
          configMap:
            name: mysql
            defaultMode: 420
      initContainers:
        - name: init-mysql
          image: 'mysql:5.7'
          command:
            - bash
            - '-c'
            - |
              set -ex
              # Generate mysql server-id from pod ordinal index.
              [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
              ordinal=${BASH_REMATCH[1]}
              echo [mysqld] > /mnt/conf.d/server-id.cnf
              # Add an offset to avoid reserved server-id=0 value.
              echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
              # Copy appropriate conf.d files from config-map to emptyDir.
              if [[ $ordinal -eq 0 ]]; then
                cp /mnt/config-map/master.cnf /mnt/conf.d/
              else
                cp /mnt/config-map/slave.cnf /mnt/conf.d/
              fi
          resources: {}
          volumeMounts:
            - name: conf
              mountPath: /mnt/conf.d
            - name: config-map
              mountPath: /mnt/config-map
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
        - name: clone-mysql
          image: 'gcr.io/google-samples/xtrabackup:1.0'
          command:
            - bash
            - '-c'
            - >
              set -ex

              # Skip the clone if data already exists.

              [[ -d /var/lib/mysql/mysql ]] && exit 0

              # Skip the clone on master (ordinal index 0).

              [[ `hostname` =~ -([0-9]+)$ ]] || exit 1

              ordinal=${BASH_REMATCH[1]}

              [[ $ordinal -eq 0 ]] && exit 0

              # Clone data from previous peer.

              ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C
              /var/lib/mysql

              # Prepare the backup.

              xtrabackup --prepare --target-dir=/var/lib/mysql
          resources: {}
          volumeMounts:
            - name: nfs-pvc
              mountPath: /var/lib/mysql
              subPath: mysql
            - name: conf
              mountPath: /etc/mysql/conf.d
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
      containers:
        - name: mysql
          image: 'mysql:5.7'
          ports:
            - name: mysql
              containerPort: 3306
              protocol: TCP
          env:
            - name: MYSQL_ALLOW_EMPTY_PASSWORD
              value: '1'
          resources:
            requests:
              cpu: 50m
              memory: 50Mi
          volumeMounts:
            - name: nfs-pvc
              mountPath: /var/lib/mysql
              subPath: mysql
            - name: conf
              mountPath: /etc/mysql/conf.d
          livenessProbe:
            exec:
              command:
                - mysqladmin
                - ping
            initialDelaySeconds: 30
            timeoutSeconds: 5
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          readinessProbe:
            exec:
              command:
                - mysql
                - '-h'
                - 127.0.0.1
                - '-e'
                - SELECT 1
            initialDelaySeconds: 5
            timeoutSeconds: 1
            periodSeconds: 2
            successThreshold: 1
            failureThreshold: 3
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
        - name: xtrabackup
          image: 'gcr.io/google-samples/xtrabackup:1.0'
          command:
            - bash
            - '-c'
            - >
              set -ex

              cd /var/lib/mysql


              # Determine binlog position of cloned data, if any.

              if [[ -f xtrabackup_slave_info && "x$(<xtrabackup_slave_info)" !=
              "x" ]]; then
                # XtraBackup already generated a partial "CHANGE MASTER TO" query
                # because we're cloning from an existing slave. (Need to remove the tailing semicolon!)
                cat xtrabackup_slave_info | sed -E 's/;$//g' > change_master_to.sql.in
                # Ignore xtrabackup_binlog_info in this case (it's useless).
                rm -f xtrabackup_slave_info xtrabackup_binlog_info
              elif [[ -f xtrabackup_binlog_info ]]; then
                # We're cloning directly from master. Parse binlog position.
                [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
                rm -f xtrabackup_binlog_info xtrabackup_slave_info
                echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                      MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
              fi


              # Check if we need to complete a clone by starting replication.

              if [[ -f change_master_to.sql.in ]]; then
                echo "Waiting for mysqld to be ready (accepting connections)"
                until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done

                echo "Initializing replication from clone position"
                mysql -h 127.0.0.1 \
                      -e "$(<change_master_to.sql.in), \
                              MASTER_HOST='mysql-0.mysql', \
                              MASTER_USER='root', \
                              MASTER_PASSWORD='', \
                              MASTER_CONNECT_RETRY=10; \
                            START SLAVE;" || exit 1
                # In case of container restart, attempt this at-most-once.
                mv change_master_to.sql.in change_master_to.sql.orig
              fi


              # Start a server to send backups when requested by peers.

              exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
                "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
              tail -f /dev/null
          ports:
            - name: xtrabackup
              containerPort: 3307
              protocol: TCP
          resources:
            requests:
              cpu: 50m
              memory: 50Mi
          volumeMounts:
            - name: nfs-pvc
              mountPath: /var/lib/mysql
              subPath: mysql
            - name: conf
              mountPath: /etc/mysql/conf.d
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext: {}
      schedulerName: default-scheduler
  volumeClaimTemplates:
    - kind: PersistentVolumeClaim
      apiVersion: v1
      metadata:
        name: nfs-pvc
        creationTimestamp: null
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
        storageClassName: nfs-client
        volumeMode: Filesystem
      status:
        phase: Pending
```

##### 说明：
如果拉取gcr.io/google-samples/xtrabackup:1.0 镜像有困难，按下面操作
```shell
docker pull ist0ne/xtrabackup
docker tag ist0ne/xtrabackup:latest gcr.io/google-samples/xtrabackup:1.0
```

##### 集群内客户端连接数据库

```shell
 kubectl run baichuan-mysql-client --rm --tty -i --restart='Never' --image mysql:5.7 --namespace ha --command -- bash
 # 连接写库
mysql -h mysql-0.mysql.ha -uroot
# 创建测试数据库
CREATE DATABASE test;
# 创建表
CREATE TABLE test.messages (message VARCHAR(250));
# 插入数据
 INSERT INTO test.messages VALUES ('hello');
 
 #连接读库 1
 mysql -h mysql-1.mysql.ha -uroot 
 # 连接读库 2
 mysql -h mysql-2.mysql.ha -uroot 
 # 通过读库的service连接
  mysql -h mysql.ha -uroot 
 # 或
   mysql -h mysql-read.ha -uroot 
```

istio代理mysql的读库port

参考：https://zhuanlan.zhihu.com/p/393749663