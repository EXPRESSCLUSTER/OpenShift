# How to Install EXPRESSCLUSTER X SingleServerSafe
## Index
- [Prerequisite](#Prerequisite)

## Prerequisite
- Allow root user account for a pod.
  ```bash
  $ oc new-project <your project>
  $ oc project <your project>
  $ oc create sa with-anyuid
  $ oc adm policy add-scc-to-user anyuid -z with-anyuid --as system:admin
  ```
## Deploy SingleServerSafe Container
- Please refer to the following page.
  - https://github.com/EXPRESSCLUSTER/CodeReady-Containers/blob/master/DeployApp.md#singleserversafe