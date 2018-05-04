## Lab 5.2: Restore

### Restore a Project

We will now delete the logging project and try to restore it from the backup.
```
[ec2-user@master0 ~]$ oc delete project logging
```

Check if the project is being deleted
```
[ec2-user@master0 ~]$ oc get project logging
```

Restore the logging project from the backup. Some objects still exist, because they are not namespaced and therefore not deleted. You will see during the restore, that these object will not be replaced.
```
[ec2-user@master0 ~]$ oc adm new-project logging --node-selector=""

[ec2-user@master0 ~]$ oc create -f /home/ec2-user/openshift_backup_[date]/projects/logging/serviceaccount.json
[ec2-user@master0 ~]$ oc create -f /home/ec2-user/openshift_backup_[date]/projects/logging/secret.json
[ec2-user@master0 ~]$ oc create -f /home/ec2-user/openshift_backup_[date]/projects/logging/configmap.json
[ec2-user@master0 ~]$ oc create -f /home/ec2-user/openshift_backup_[date]/projects/logging/rolebindings.json
[ec2-user@master0 ~]$ oc create -f /home/ec2-user/openshift_backup_[date]/projects/logging/project.json
[ec2-user@master0 ~]$ oc create -f /home/ec2-user/openshift_backup_[date]/projects/logging/daemonset.json
```

Scale the logging components.
```
[ec2-user@master0 ~]$ oc get dc
NAME                  REVISION   DESIRED   CURRENT   TRIGGERED BY
logging-curator       5          1         1         config
logging-es-a4nhrowo   5          1         1         config
logging-kibana        7          1         0         config


[ec2-user@master0 ~]$ oc scale dc logging-kibana --replicas=0
[ec2-user@master0 ~]$ oc scale dc logging-curator --replicas=0
[ec2-user@master0 ~]$ oc scale dc logging-es-[HASH] --replicas=0
[ec2-user@master0 ~]$ oc scale dc logging-kibana --replicas=1
[ec2-user@master0 ~]$ oc scale dc logging-curator --replicas=1
[ec2-user@master0 ~]$ oc scale dc logging-es-[HASH] --replicas=1
``` 

Check if the pods are coming up again
```
[ec2-user@master0 ~]$ oc get pods -w
```

If all the pods are ready, Kibana should be receiving logs again.
```
https://logging.app[X].lab.openshift.ch
```


### Restore the etcd Cluster

First, we need to stop etcd on the first master.
```
[ec2-user@master0 ~]$ sudo systemctl stop etcd
```

The cluster is now down and you can't get any resources through the console. We are now copying the files back from the backup and set the right permissions.
```
[ec2-user@master0 ~]$ ETCD_DIR=/var/lib/etcd/
[ec2-user@master0 ~]$ sudo mv $ETCD_DIR /var/lib/etcd.orig
[ec2-user@master0 ~]$ sudo cp -Rp etcd.bak $ETCD_DIR
[ec2-user@master0 ~]$ sudo chcon -R --reference /var/lib/etcd.orig/ $ETCD_DIR
[ec2-user@master0 ~]$ sudo chown -R etcd:etcd $ETCD_DIR
```

Add the "--force-new-cluster" parameter to the etcd unit file, start etcd and check if it's running. This is needed, because initially it will create a new cluster with the existing data from the backup.
```
[ec2-user@master0 ~]$ sudo sed -i '/ExecStart/s/"$/  --force-new-cluster"/' /usr/lib/systemd/system/etcd.service
[ec2-user@master0 ~]$ sudo systemctl daemon-reload
[ec2-user@master0 ~]$ sudo systemctl start etcd
[ec2-user@master0 ~]$ sudo systemctl status etcd
```

The cluster is now initialized, so we need to remove the "--force-new-cluster" parameter again and restart etcd.
```
[ec2-user@master0 ~]$ sudo sed -i '/ExecStart/s/ --force-new-cluster//' /usr/lib/systemd/system/etcd.service
[ec2-user@master0 ~]$ sudo systemctl daemon-reload
[ec2-user@master0 ~]$ sudo systemctl restart etcd
[ec2-user@master0 ~]$ sudo systemctl status etcd
```

Check if etcd is healthy and check if "/openshift.io" exists in etcd
```
[ec2-user@master0 ~]$ sudo etcdctl -C https://master0.user[X].lab.openshift.ch:2379 --ca-file=/etc/etcd/ca.crt --cert-file=/etc/etcd/peer.crt --key-file=/etc/etcd/peer.key cluster-health
member 92c764a37c90869 is healthy: got healthy result from https://127.0.0.1:2379
cluster is healthy
[ec2-user@master0 ~]$ sudo etcdctl -C https://master0.user[X].lab.openshift.ch:2379 --ca-file=/etc/etcd/ca.crt --cert-file=/etc/etcd/peer.crt --key-file=/etc/etcd/peer.key ls /
/openshift.io
```

We need to change the peerURL of the etcd to it's private ip. Make sure to
correctly copy the member_id and private_ip.
```
[ec2-user@master0 ~]$ sudo etcdctl -C https://master0.user[X].lab.openshift.ch:2379 --ca-file=/etc/etcd/ca.crt --cert-file=/etc/etcd/peer.crt --key-file=/etc/etcd/peer.key member list
[member_id]: name=master0.user[X].lab.openshift.ch peerURLs=https://localhost:2380 clientURLs=https://[private_ip]:2379 isLeader=true

[ec2-user@master0 ~]$ sudo etcdctl -C https://master0.user[X].lab.openshift.ch:2379 --ca-file=/etc/etcd/ca.crt --cert-file=/etc/etcd/peer.crt --key-file=/etc/etcd/peer.key member update [member_id] https://[private_ip]:2379
Updated member with ID 6248d01c5701 in cluster

[ec2-user@master0 ~]$ sudo etcdctl -C https://master0.user[X].lab.openshift.ch:2379 --ca-file=/etc/etcd/ca.crt --cert-file=/etc/etcd/peer.crt --key-file=/etc/etcd/peer.key member list
6248d01c5701: name=master0.user[X].lab.openshift.ch peerURLs=https://172.31.46.201:2379 clientURLs=https://172.31.46.201:2379 isLeader=true
```


---

**End of Lab 5.2**

<p width="100px" align="right"><a href="60_monitoring_troubleshooting.md">6. Monitoring and Troubleshooting →</a></p>

[← back to the Chapter Overview](50_backup_restore.md)
