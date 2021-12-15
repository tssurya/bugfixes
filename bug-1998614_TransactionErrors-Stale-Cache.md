# Transaction Errors caused due to stale cache in OVN-K

BZ link: https://bugzilla.redhat.com/show_bug.cgi?id=1998614  
Component: OVN-Kubernetes  
OCP version: 4.7.24  
OVN Version: 20.12.0-24.el8fdp.x86_64  
OVS Version: 2.13.0-79.el8fdp.x86_64  
Number of Nodes: 120 (+3 masters)  

## Problem Statement

During OVN-K scale testing, it was observed that pod creations started failing with the error:
```
  Warning  ErrorAddingLogicalPort  39m                    controlplane  error while creating logical port f5-served-ns-30_pod-served-1-6-served-job-30 error: Transaction Failed due to an error: constraint violation details: Transaction causes multiple rows in "Logical_Switch_Port" table to have identical values (f5-served-ns-30_pod-served-1-6-served-job-30) for index on column "name".  First row, with UUID 8b534f6b-23a8-4984-863f-7b391b301067, was inserted by this transaction.  Second row, with UUID 25107606-b83c-420b-a903-ae7f6278ecaf, existed in the database before this transaction, which modified some of the row's columns but not any columns in this index. in committing transaction

  Warning  FailedCreatePodSandBox  4m22s (x139 over 56m)  kubelet       (combined from similar events): Failed to create pod sandbox: rpc error: code = Unknown desc = failed to create pod network sandbox k8s_pod-served-1-6-served-job-30_f5-served-ns-30_0825a91a-27ab-4a66-9712-5dfceff04331_0(d4e1ad6adfb4efa06f5e41faeb11037679fc7d5e85c5744d0a39951f31211c07): error adding pod f5-served-ns-30_pod-served-1-6-served-job-30 to CNI network "multus-cni-network": [f5-served-ns-30/pod-served-1-6-served-job-30:ovn-kubernetes]: error adding container to network "ovn-kubernetes": CNI request failed with status 400: '[f5-served-ns-30/pod-served-1-6-served-job-30 d4e1ad6adfb4efa06f5e41faeb11037679fc7d5e85c5744d0a39951f31211c07] [f5-served-ns-30/pod-served-1-6-served-job-30 d4e1ad6adfb4efa06f5e41faeb11037679fc7d5e85c5744d0a39951f31211c07] failed to configure pod interface: error while waiting on flows for pod: timed out waiting for OVS flows

```

## Process

- Looking at the message it was clear that for some reason OVN-K was trying to create the lsp (logical switch port) for a pod that already had an existing logical switch port entry in the northbound database.
- We checked the ovnkube-master logs and saw this error message repeating over and over for all new pods that were trying to get created and each time the UUID hash value for the lsp was different.
- Next we tried to delete the existing port entry for that pod from the northbound database. This would stop this error message for a brief period, but as soon as a new lsp for that pod would get created, the error would start again.
- Looking at the logs of all other components like `ovn-controller`, `nbdb`, `sbdb`, `northd` we couldn't find anything fishy expect high poll intervals and cpu consumption but that was because the cluster was havily loaded (This bug seemed to happen only at scale on loaded clusters).
- We enabled debug mode for jsonrpc logs on northd and nbdb components to get more information:
  ```
  ovn-appctl -t <ovn-component> vlog/set jsonrpc:dbg
  ```
- We created a new pod `client-on-ovn-worker` in `bar` namespace to track step by step what was happening.
  - We saw the `addLogicalPort` happening on master but failing with the transaction error:
  ```
  I0831 18:15:39.788465       1 event.go:282] Event(v1.ObjectReference{Kind:"Pod", Namespace:"bar", Name:"client-on-ovn-worker", UID:"a04870f6-ca9e-4398-87b9-44688233b48a", APIVersion:"v1", ResourceVersion:"4213537", FieldPath:""}): type: 'Warning' reason: 'ErrorAddingLogicalPort' error while creating logical port bar_client-on-ovn-worker error: Transaction Failed due to an error: constraint violation details: Transaction causes multiple rows in "Logical_Switch_Port" table to have identical values (bar_client-on-ovn-worker) for index on column "name".  First row, with UUID e9b5fd13-664c-470f-90ee-cf0500ab4dc4, was inserted by this transaction.  Second row, with UUID 85c48bd1-8824-4b9c-bb5e-11a658be4bc1, existed in the database before this transaction, which modified some of the row's columns but not any columns in this index. in committing transaction
  ```
  - We checked the nbdb for the port entry:
  ```
  sh-4.4# ovn-nbctl lsp-list worker012-r640
  85c48bd1-8824-4b9c-bb5e-11a658be4bc1 (bar_client-on-ovn-worker) 
  ```
  - Clearly OVN-K master was trying to create lsp `e9b5fd13-664c-470f-90ee-cf0500ab4dc4` when `85c48bd1-8824-4b9c-bb5e-11a658be4bc1` already existed for that pod.
  - Again from ovnkube-master logs we could see the first requested for `addLogicalPort` actually succeeded but even then for some reason ovn-k was trying to create the second port. This clearly smelled like a cache inconsistency issue in OVN-K master:
  ```
  I0831 18:15:39.774158       1 kube.go:63] Setting annotations map[k8s.ovn.org/pod-networks:{"default":{"ip_addresses":["10.130.17.33/23"],"mac_address":"0a:58:0a:82:11:21","gateway_ips":["10.130.16.1"],"ip_address":"10.130.17.33/23","gateway_ip":"10.130.16.1"}}] on pod bar/client-on-ovn-worker
  I0831 18:15:39.788344       1 pods.go:302] [bar/client-on-ovn-worker] addLogicalPort took 17.058603ms
  E0831 18:15:39.788361       1 ovn.go:635] error while creating logical port bar_client-on-ovn-worker error: Transaction Failed due to an error: constraint violation details: Transaction causes multiple rows in "Logical_Switch_Port" table to have identical values (bar_client-on-ovn-worker) for index on column "name".  First row, with UUID e9b5fd13-664c-470f-90ee-cf0500ab4dc4, was inserted by this transaction.  Second row, with UUID 85c48bd1-8824-4b9c-bb5e-11a658be4bc1, existed in the database before this transaction, which modified some of the row's columns but not any columns in this index. in committing transaction
  I0831 18:15:39.788465       1 event.go:282] Event(v1.ObjectReference{Kind:"Pod", Namespace:"bar", Name:"client-on-ovn-worker", UID:"a04870f6-ca9e-4398-87b9-44688233b48a", APIVersion:"v1", ResourceVersion:"4213537", FieldPath:""}): type: 'Warning' reason: 'ErrorAddingLogicalPort' error while creating logical port bar_client-on-ovn-worker error:Transaction Failed due to an error: constraint violation details: Transaction causes multiple rows in "Logical_Switch_Port" table to have identical values (bar_client-on-ovn-worker) for index on column "name".  First row, with UUID e9b5fd13-664c-470f-90ee-cf0500ab4dc4, was inserted by this transaction.  Second row, with UUID 85c48bd1-8824-4b9c-bb5e-11a658be4bc1, existed in the database before this transaction, which modified some of the row's columns but not any columns in this index. in committing transaction  
  ```
  - It looked like goovn cache for port bindings that ovnkube-master used was stale that was causing master to retry port creation.
  - When looking at the jsonrpc logs for nbdb, we saw the request and reply messages for `lsp-add` but no update messages were getting sent after the entry was created in the nbdb. That explained why the `cacheLogicalSwitchPort` was not getting called since cache update happens only after receiving the update message from ovsdb server.
  - Then we tried to find out why update messages weren't coming and saw that there were 0 monitors running on the pod that was the nbdb raft leader:
  ```
  [root@master-0 ~]# ovn-appctl -t /var/run/ovn/ovnnb_db.ctl memory/show
  cells:100055 monitors:0 raft-connections:4 raft-log:3650 sessions:3
  ```
  - Since there were no monitors running we were unable to see any updates in the messages. We were also not able to see any `monitor_cancel` messages from nbdb. It seemed to be silently drop the monitor (https://bugzilla.redhat.com/show_bug.cgi?id=2000375).
  - Conclusion: Since the monitor was getting dropped, we stopped getting update messages when an entry got created in nbdb, and thus the goovn cache became stale and ovnkube-master kept trying to recreate the lsp thinking it doesn't exist in the db and this caused the transaction error with duplicate entries.

## Fix

We couldn't figure out why ovsdbserver was just dropping the monitor without sending the cancel message (maybe a load thing?), anyhow we did a fix on OVN-K side: https://github.com/openshift/ovn-kubernetes/pull/717 to triggger a `reconnect` to the server in case we hit a transaction error (which is not a common thing to hit). The reconnect would then recreate the monitors and we'd start receiving the updates.


