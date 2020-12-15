ocp46_etcd_restore
=========

OCP ETCD Restore role performs a ETCD cluster data restore in an OpenShift 4.2 environment from a previous backup.


Prerequisites
--------------

- Installed `oc` command-line tool for OCP v`4.6` on the client machine
- Logged in via `oc` command-line tool to your v`4.6` cluster on the client machine


Role Variables
--------------

As is generally known, Ansible variables could be defined in different files. The following subsections include default variables and required variables which have to be defined in order to perform an ETCD cluster data backup.

### Required Vars

The following required variables have to be defined when role is triggered.

|Variable|Comment|Type|
|---|---|---|
|`backup_src`|ETCD backup folder in client instance which contains files created by `ocp46_etcd_backup` Ansible role|String|

Example Inventory
-----------------

```ini
[masters]
master0 ansible_host=10.0.0.1
master1 ansible_host=10.0.0.2
master2 ansible_host=10.0.0.3

[recovery]
master3 ansible_host=10.0.0.4

[localhost]
localhost ansible_host=localhost ansible_connection=local

[ocp:children]
masters
recovery

[ocp:vars]
ansible_user=core
http_proxy=""
https_proxy=""
```

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```yaml
##
# Example:
# $ ansible-playbook -i inventory ocp46-restore-etcd.yml
# $ ansible-playbook -i inventory ocp46-restore-etcd.yml --extra-vars="backup_src=/tmp/etcd_backup"
##
- hosts: masters
  gather_facts: False
  become: True
  environment:
    http_proxy: "{{ http_proxy }}"
    https_proxy: "{{ https_proxy }}"
  tasks:
    - name: Move the existing etcd pod file out of the kubelet manifest directory
      shell: mv /etc/kubernetes/manifests/etcd-pod.yaml /tmp
      ignore_errors: True

    - name: Move the existing Kubernetes API server pod file out of the kubelet manifest directory
      shell: mv /etc/kubernetes/manifests/kube-apiserver-pod.yaml /tmp
      ignore_errors: True

    - name: Move the etcd data directory to a different location
      shell: mv /var/lib/etcd/ /tmp
      ignore_errors: True

- hosts: recovery
  gather_facts: False
  environment:
    http_proxy: "{{ http_proxy }}"
    https_proxy: "{{ https_proxy }}"
  tasks:
    - name: Perform ETCD 4.6 Restore
      import_role:
        name: ocp46_etcd_restore
      vars:
        backup_src: "~/Downloads/etcd_backup"

- hosts: recovery
  gather_facts: False
  become: True
  environment:
    http_proxy: "{{ http_proxy }}"
    https_proxy: "{{ https_proxy }}"
  tasks:
    - name: Restart the kubelet service on a recovery host
      service:
        name: kubelet.service
        state: restarted

- pause:
    seconds: 30

- hosts: masters
  gather_facts: False
  become: True
  serial: 1
  environment:
    http_proxy: "{{ http_proxy }}"
    https_proxy: "{{ https_proxy }}"
  tasks:
    - name: Restart the kubelet service on all master hosts
      service:
        name: kubelet.service
        state: restarted

- hosts: localhost
  gather_facts: False
  become: False
  tasks:
    - name: Force etcd redeployment
      shell: >
        oc patch etcd cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge

    - name: Verify all nodes are updated to the latest revision
      shell: >
        oc get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
      register: verify_lates_rev

    - name: Review the output of `AllNodesAtLatestRevision`
      debug:
        msg: "{{ verify_lates_rev.stdout }}"

    - pause:
        seconds: 30

    - name: Update the `kubeapiserver`
      shell: >
        oc patch kubeapiserver cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge

    - name: Update the `kubecontrollermanager`
      shell: >
        oc patch kubecontrollermanager cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge

    - name: Update the `kubescheduler`
      shell: >
        oc patch kubescheduler cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$( date --rfc-3339=ns )"'"}}' --type=merge

    - name: Verify that all master hosts have started and joined the cluster
      shell: >
        oc get pods -n openshift-etcd | grep etcd
      register: etcd_cluster_pods

    - name: Review the output of etcd pods
      debug:
        msg: "{{ etcd_cluster_pods.stdout }}"
```

License
-------

MIT

Author Information
------------------

- Lucian Maly [@Red Hat](https://github.com/redhatofficial)
