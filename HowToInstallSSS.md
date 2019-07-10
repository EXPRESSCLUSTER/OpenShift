# How to Install EXPRESSCLUSTER X SingleServerSafe
## Index
- [Prerequisite](#Prerequisite)
- [Install SingleServerSafe on a Base Container](#Install-SingleServerSafe-on-a-Base-Container)
- [Sidecar Pattern](#Sidecar-Pattern)
  - [MariaDB](#MariaDB)

## Prerequisite
- We have evaluated on the following environment.
  - OpenShift v3.11.51
    - Docker 1.13.1
    - kubernetes v1.11.0
  - docker.io/openshift/base-centos7
  - EXPRESSCLUSTER X SingleServerSafe 4.1 for Linux (4.1.1-1)
- Copy the library files for monitoring target. For example, if you use MariaDB, install MariaDB on a temporary containr and copy the library file to the container host.
- Allow root user account for the containers.
  ```bash
  # oc project <your project>
  # oc adm policy add-scc-to-user anyuid -z default -n <your project name>
  ```

## Install SingleServerSafe on a Base Container
1. Save EXPRESSCLUSTER rpm file and license files on a directory (e.g. /root/work) of container host.
1. Check if the base-centos7 image is available on worker nodes.
   ```bash
   # docker images |grep base-centos7
    :
   docker.io/openshift/base-centos7                                 latest              4842f0bd3d61        2 years ago         383 MB   
   ```
1. Create a temporary container.
   ```bash
   # docker run -it -d -p 29003:29003 -v /root/work/:/mydata --name base-centos7-work base-centos7:latest /bash
   ```
1. Attach the temporary container.
   ```bash
   # docker exec -it base-centos7-work bash
   ```
1. If your environment behind a proxy server, edit yum.conf.
   ```bash
   # vi /etc/yum.conf
     :
   proxy=http://<your proxy server>:<port number>
   sslverify=false
   ```
1. Install iproute and sysvinit-tools.
   ```bash
   # yum -y install iproute
   # yum -y install sysvinit-tools
   ```
   - iproute is needed to run **ip** command.
   - sysinit-tools is need to run **pidof** command.
1. Install EXPRESSCLUSTER X SingleServerSafe.
   ```bash
   # cd /mydata
   # rpm -i clusterprosss-4.1.1-1.x86_64.rpm
   ```
1. Register the licenses of EXPRESSCLUSTER X SingleServerSafe and Agent.
1. Exit from the container and stop it.
   ```bash
   # exit
   # docker stop base-centos7-work
   ```

## Sidecar Pattern
### MariaDB
1. Pull MariaDB image. 
   ```bash
   # docker pull mariadb
   ```
1. Create YAML file (e.g. pv-mariadb.yml) for a persistent volume.
   ```yaml
   # pv-mariadb.yml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: pv-mariadb1
   spec:
     capacity:
       storage: 10Gi
     accessModes:
     - ReadWriteOnce
     persistentVolumeReclaimPolicy: Retain
     nfs:
       path: /exports/mariadb1
       server: <nfs server name or IP address>
       readOnly: false
   ```
   Apply the YAML file.
   ```bash
   # oc apply -f pv-mariadb.yml
   # oc get pv
   NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM   STORAGECLASS   REASON    AGE
    :
   pv-mariadb1   10Gi       RWO            Retain           Bound                                       12m
   ```
1. Create a YAML (e.g. pvc-mariadb1.yml) for a persisten volume claim.
   ```yaml
   # pvc-mariadb1.yml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: pvc-mariadb1
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 10Gi
   ```
   Apply the YAML file.
   ```bash
   # oc apply -f pvc-mariadb1.yml
   # oc get pv,pvc
   NAME                                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                         STORAGECLASS   REASON    AGE
    :
   persistentvolume/pv-mariadb1         10Gi       RWO            Retain           Bound      <your project>/pvc-mariadb1                            18m
   
   NAME                                 STATUS    VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
   persistentvolumeclaim/pvc-mariadb1   Bound     pv-mariadb1   10Gi       RWO                           17m
   ```
1. Create a YAML file for MariaDB.
   ```yaml
   # mariadb1.yml
   apiVersion: v1
   kind: Pod
   metadata:
     name: mariadb1
     labels:
       app: mariadb1
   spec:
     containers:
     - name: mariadb1
       image: mariadb:latest
       imagePullPolicy: Never
       env:
       - name: MYSQL_USER
         value: foo
       - name: MYSQL_PASSWORD
         value: <password>
       - name: MYSQL_ROOT_PASSWORD
         value: <password>
       ports:
       - containerPort: 3306
       volumeMounts:
       - name: pvc-mariadb1
         mountPath: /var/lib/mysql
     volumes:
     - name: pvc-mariadb1
       persistentVolumeClaim:
         claimName: pvc-mariadb1  
   ```
   Apply the YAML file.
   ```bash
   # oc apply -f mariadb1.yml
   # oc get pod
   NAME                         READY     STATUS    RESTARTS   AGE
    :
   mariadb1                     1/1       Running   0          4s  
   ```
1. Attach the container and create a database for monitoring.
   ```bash
   # oc exec -it mariadb1 bash
   # mysql -u root -p
   Enter password:
   Welcome to the MariaDB monitor.  Commands end with ; or \g.
    :
   MariaDB [(none)]> show databases;
   +--------------------+
   | Database           |
   +--------------------+
   | information_schema |
   | mysql              |
   | performance_schema |
   +--------------------+
   3 rows in set (0.001 sec)
   
   MariaDB [(none)]> select user,host from mysql.user;
   +----------+-----------+
   | User     | Host      |
   +----------+-----------+
   | foo      | %         |
   | root     | %         |
   | root     | localhost |
   +----------+-----------+
   3 rows in set (0.002 sec)
   
   MariaDB [(none)]> create database testdb;
   Query OK, 1 row affected (0.002 sec)
   
   MariaDB [(none)]> show databases;
   +--------------------+
   | Database           |
   +--------------------+
   | information_schema |
   | mysql              |
   | performance_schema |
   | testdb             |
   +--------------------+
   4 rows in set (0.001 sec)
   
   MariaDB [(none)]> exit    
   ```
1. Delete the MariaDB pod.
   ```bash
   # oc delete -f mariadb1.yml
   ```
1. Save the library file (e.g. libmysqlclient.so.18.0.0) on a work directory (e.g. /root/work).
1. Create a container to modify configuration.
   ```bash
   # docker commit base-centos7-work base-centos7-sss4mariadb
   # docker images base-centos7-sss4mariadb
   REPOSITORY                 TAG                 IMAGE ID            CREATED              SIZE
   base-centos7-sss4mariadb   latest              3ea95d3c0eb8        About a minute ago   539 MB
   # docker run -it -d -p 29103:29003 -v /root/work:/mydata --name base-centos7-sss4mariadb-work base-centos7-sss4mariadb:latest bash
   ```
1. Copy the library file from /mydata to /root.
   ```bash
   # cp /mydata/libmysqlclient.so.18.0.0 /root
   # ls -l /root
   total 3072
    :
   -rwxr-xr-x. 1 root root 3135712 Jul  8 07:45 libmysqlclient.so.18.0.0
   ```
1. Run the following command.
   ```bash
   cd /opt/nec/clusterpro/etc/systemd
   ./clusterpro_evt.sh start
   ./clusterpro_trn.sh start
   ./clusterpro_webmgr.sh start
   ```
1. On container host or the other machine, start web browser and access to the following URL.
   ```
   http://<IP address of the container host>:29103
   ```
1. Remove **user monitor resource** and add the following group, resource and monitor resource at least.
   - Failover Group
     - Add a failover group and set the default parameters.
   - Resource
     - Add exec resource and set the default parameters.
   - Monitor Resource
     - Add MySQL monitor resource and set the following parameters.

       |Parameter      |Value|Remarks|
       |---------------|-----|-------|
       |Target Resource|exec |       |
       |Database Name  |testdb|      |
       |IP Address     |127.0.0.1|Default|
       |Port           |3306|Default|
       |User Name      |<your user account>||
       |Password       |<your user password>||
       |Table          |mysqlwatch|Default|
       |Library Path   |/root/libmysqlclient.so.18.0.0||

1. Create the script file as below and save it on some directory on the container (e.g. /root/start.sh).
   ```bash
   cd /opt/nec/clusterpro/etc/systemd
   ./clusterpro_evt.sh start
   ./clusterpro_trn.sh start
   ./clusterpro_alertsync.sh start
   ./clusterpro_webmgr.sh start
   clpcfctrl --push
   clpcl -s
   ```
   And change file mode bits as below.
   ```bash
   # chmod 755 start.sh
   ```
1. Stop the container and commit the image.
   ```bash
   # docker stop base-centos7-sss4mariadb-work
   ```
1. Save the container image and load the image on worker nodes.
   ```bash
   (On master node)
   # docker save base-centos7-sss4mariadb-work > base-centos7-sss4mariadb.tar

   (On worker node)
   # docker load < base-centos7-sss4mariadb.tar
   ```
1. Create a YAML file (e.g. sss4mariadb1.yml) as below for a pod of MariaDB and SigleServerSafe.
   ```yaml
   # sss4mariadb1.yml
   apiVersion: v1
   kind: Pod
   metadata:
     name: sss4mariadb1
     labels:
       app: sss4mariadb1
   spec:
     containers:
     - name: mariadb1
       image: mariadb:latest
       imagePullPolicy: Never
       env:
       - name: MYSQL_USER
         value: foo
       - name: MYSQL_PASSWORD
         value: <password>
       - name: MYSQL_ROOT_PASSWORD
         value: <password>
       ports:
       - containerPort: 3306
       volumeMounts:
       - name: pvc-mariadb1
         mountPath: /var/lib/mysql
     - name: sss
       image: base-centos7-sss4mariadb
       imagePullPolicy: Never
       command: ['sh', '-c', 'sh /root/start.sh && tail -f /dev/null']
       ports:
       - containerPort: 29003
     volumes:
     - name: pvc-mariadb1
       persistentVolumeClaim:
         claimName: pvc-mariadb1
   ```
   Apply the YAML file.
   ```bash
   # oc apply -f sss4mariadb1.yml
   # oc get pod 
   NAME                         READY     STATUS    RESTARTS   AGE
    :
   sss4mariadb1                 2/2       Running   0          7m
   ```
1. Attach the each container and check if MariaDB and EXPRESSCLUSTER X SingleServerSafe are running.
   ```bash
   (For MariaDB)
   # oc exec -it sss4mariadb1 -c mariadb1 bash

   (For EXPRESSCLUSTER X SingleServerSafe) 
   # oc exec -it sss4mariadb1 -c sss1 bash
   ```
1. Create a YAML file (e.g. svc-sss4mariadb1.yml) as below for 
   ```yaml
   # svc-sss4mariadb1.yml
   apiVersion: v1
   kind: Service
   metadata:
     name: sss4mariadb1
   spec:
     type: NodePort
     ports:
       - name: mariadb1
         protocol: TCP
         port: 9106
         targetPort: 3306
         nodePort: 30106
       - name: sss1
         protocol: TCP
         port: 9103
         targetPort: 29003
         nodePort: 30103
     selector:
       app: sss4mariadb1 
   ```
   Apply the YAML file.
   ```bash
   # oc apply -f svc-sss4mariadb.yml
   # oc get svc
   NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
    :
   sss4mariadb1         NodePort    172.30.14.190   <none>        9106:30106/TCP,9103:30103/TCP   4s
   ```
1. Check if you can access MariaDB and Cluster WebUI.
   - MariaDB
     ```bash
     (On MariaDB client machine)
     # mysql -h <IP address of master or worker node> -p 30106 -u root -p
     ```
   - Cluster WebUI
     - Start a web browser and enter the following URL.
       ```
       http://<IP address of master or worker node>:30103
       ```
