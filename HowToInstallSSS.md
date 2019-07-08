# How to Install EXPRESSCLUSTER X SingleServerSafe
## Index
- [Prerequisite](#Prerequisite)
- [Install SingleServerSafe on a Base Container](#Install-SingleServerSafe-on-a-Base-Container)
## Prerequisite
- We have evaluated on the following environment.
  - OpenShift v3.11.51
    - Docker 1.13.1
    - kubernetes v1.11.0
  - docker.io/openshift/base-centos7
  - EXPRESSCLUSTER X SingleServerSafe 4.1 for Linux (4.1.1-1)
- Copy the library files for monitoring target. For example, if you use MariaDB, install MariaDB on a temporary containr and copy the library file to the container host.
- (FIXME) Allow root account for SingleServerSafe container.

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

## MariaDB
### Setup MariaDB
1. (FIXME) Pull MariaDB image or install MariaDB to your container. And create a database for monitoring.

### Modify the Configuration for MariaDB
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
   clpcl -s
   tail -f /dev/null
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

### Create a YAML File
```yaml

```
