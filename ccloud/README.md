# CCloud RHOSP 13 Deployment
## RHOSP 13 Deployment with 1 Controller/Compute Node in 5 ports Configurations

**DISCLAIMER** The code has been forked from [Dell EMC Ready Architecture For Red Hat OpenStack Platform 13](https://github.com/dsp-jetpack/JetPack/tree/JS-13.2) and modified to fulfill our requirements.

This installation includes steps after Undercloud Installation i.e steps till downloading and uploading of overcloud images are done in automated fashion through JetPack code. The rest of the steps which have been performed are mentioned below:

------------------------------------------------------------------------------------

### Code Cloning into Director VM
After completion of undercloud installation, clone the code into Director VM as follows:

```bash 
$ cd $HOME
$ git clone https://github.com/xFlowResearch/ccloud.git
$ cd ccloud
$ chmod +x ccloud/pilot/*.sh
$ chmod +x ccloud/pilot/*/*.sh
$ chmod +x ccloud/pilot/*.py
$ chmod +x ccloud/pilot/*/*.py
```

### Nodes Preparation

- Discover the nodes as for overcloud installation:
  ```bash
  $ cd $HOME/ccloud/pilot/
  $ ./discover_nodes/discover_nodes.py -u <idrac_user> -p <idrac_password> [Controller_iDRAC_IP] [Controller_iDRAC_IP] |tee $HOME/instackenv.json
  ```
  **NOTE: Make iDRAC username/password for all nodes consistent, otherwise each node needs to be discovered separately through the above mentioned command.**

- Configure nodes through iDRAC for clearing BIOS queues and reconfiguring NICs for PXE boot:
  ```bash
  $ ./config_idracs.py
  ```
- Import nodes using the following command:
  ```bash 
  $ ./import_nodes.py
  ```
- Turn off the nodes and set PXE NIC boot order using the following command:
  ```bash
  $ ./prep_overcloud_nodes.py
  ```
- Introspect nodes using the following command:
  ```bash
  $ ./introspect_nodes.py # for Out-of-Band introspection
  $ ./introspect_node.py -i <Node_iDRAC_IP> # for In-Band introspection
  ```
  **NOTE: In-band introspection is done for one node at a time and node is needed to be provided manually whereas out-of-band introspection takes values from ```ironic node-list``` and introspect them respectively.**

- Assign roles to the nodes as follows:
  ```bash
  $ ./assign_role.py -s <Controller_iDRAC_IP> controller-0
  $ ./assign_role.py -s <Compute_iDRAC_IP> compute-0
  ```
  **NOTE: The ```-s``` flag is used to skip RAID configurations. If RAID in not configured manually, run the script without -s flag and let the scripts configure RAID for relevant role.**

- Deploy overcloud using the following command:
  ```bash
  cd ;source ~/stackrc; openstack overcloud deploy \
  --log-file ~/pilot/overcloud_deployment.log \
  -t 300  --stack ccloud \
  --templates ~/pilot/templates/overcloud \
  -r ~/pilot/templates/roles_data.yaml \
  -e ~/pilot/templates/overcloud/environments/network-isolation.yaml \
  -e ~/pilot/templates/network-environment.yaml \
  -e ~/pilot/templates/nic-configs/5_port/nic_environment.yaml \
  -e ~/pilot/templates/static-ip-environment.yaml \
  -e ~/pilot/templates/static-vip-environment.yaml \
  -e ~/pilot/templates/node-placement.yaml \
  -e ~/pilot/overcloud_images.yaml \ # use ~/pilot/overcloud-local.yaml in case of local docker registry
  -e ~/pilot/templates/dell-environment.yaml \
  --libvirt-type kvm \
  --ntp-server 10.0.120.18
  ```

------------------------------------------------------------------------------------------------------

### APPENDIX A. Creating Local Docker Registry

Incase of slow internet connection, the deployment gets failed due to connection timeouts. It is recommended to host docker containers on the undercloud in this case to avoid the deployment failures.
- Generate Configuration file for downloading containers from RedHat Registry as follows:
  ```bash
  $ openstack overcloud container image prepare \
  --namespace=registry.access.redhat.com/rhosp13 \
  -e /usr/share/openstack-tripleo-heat-templates/environments/ceph-ansible/ceph-ansible.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/ironic.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services/barbican.yaml \
  -e /usr/share/openstack-tripleo-heat-templates/environments/services-docker/octavia.yaml \
  --set ceph_namespace=registry.access.redhat.com/rhceph \
  --set ceph_image=rhceph-3-rhel7 \
  --tag-from-label {version}-{release} \
  --push-destination=10.0.120.13:8787 \
  --output-env-file ~/overcloud-local.yaml \
  --output-images-file ~/local_registry_images.yaml 
  ```
- Create the Local Registry using the following command:
```bash
$ sudo openstack overcloud container image upload --config-file  ~/local_registry_images.yaml --verbose
```

------------------------------------------------------------------------------------------------------

### APPENDIX B. Clean Heat Stack Manually

Sometimes Heat Stack deletion fails and stack gets stuck there. Follwoing steps can help in deleting heat stack manually through DB:
```bash
$ export $project=ccloud
$ mysql -hlocalhost -uroot -p
> use heat;
> delete from resource_data where resource_id in (select id from resource where stack_id in (select id from stack where name = \"$project\"));
> delete from resource where stack_id in (select id from stack where name = \"$project\");
> delete from event where stack_id in (select id from stack where name = \"$project\");
> delete from stack where name = \"$project\";
```

------------------------------------------------------------------------------------------------------