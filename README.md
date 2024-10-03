# CVP Enrollment Token Push
In networks where customers are operating multiple CVP (CloudVision Portal) clusters, additional steps are required to enroll switches to the 2nd CVP cluster.  This may be because the customer wants to run a pair of clusters for high availability, or to migrate from one cluster to another.

Enrollment to the first CVP cluster is typically carried out using ZTP (Zero Touch Provisioning).  Once the device is enrolled to the primary CVP cluster, the Ansible playbook in this repo can be used to deploy the enrollment token from the secondary cluster via the primary cluster, to individual devices.  Once this is complete, the primary cluster can be used to update the Terminattr configuration on each device to point to both clusters.

Various things are required to make this work (and more details can be found below):
  - The "Push Token" custom action must be installed on the primary CVP.
  - A service account should be created on the secondary CVP.
  - Variables in `deploy-token.yml` should be changed as required.

## Push Token Custom Action
The Push Token Custom Action can be found in the [cloudvision-python-actions](https://github.com/aristanetworks/cloudvision-python-actions) repo on Github.  Installation instructions for this custom action can be found in the [README.md](https://github.com/aristanetworks/cloudvision-python-actions/blob/trunk/README.md) file.  Please note that you should use the branch in the Github repository that corresponds to your CVP version.  For CVaaS please use `trunk`.

## Service Account
A service account should be created on the secondary CVP cluster and the hostname and service account token to access this cluster should be added to the variables in the playbook.  This can be created under Settings -> Service Accounts.

## Variables in deploy-token.yml
The following variables need to be set as below.  Note that we're making use of an Ansible lookup to retrieve the CVaaS service account token from an environment variable.

```yaml
  vars:
    # Set as required.  All devices in this group will have tokens deployed.
    fabric_name: FABRIC

    # Hostname and service account token of CVP Cluster
    # to fetch enrollment token from.
    cvp_src_cluster: www.arista.io
    cvp_src_token: "{{ lookup('ansible.builtin.env', 'CVAAS_TOKEN')}}"
    cvp_cc_output: cc_output.yml

    # This URL & token_request may only work in CVaaS currently.
    # Alternate URL/token_request are below.
    # The "Extract Token" task below also requires changing.
    url: "https://{{ cvp_src_cluster }}/api/resources/admin.Enrollment/AddEnrollmentToken"
    token_request: |
      {
        "enrollmentToken": {
          "validFor": "86400s",
          "reenrollDevices": ["*"]
        }
      }

    # Required for older CVP as above.
    # url: "https://{{ cvp_src_cluster }}/cvpservice/enroll/createToken"
    # token_request: |
    #   {
    #     "duration": "24h",
    #     "reenrollDevices": ["*"]
    #   }
```

In addition to the variable changes you'll need to change the following task too:

```yaml
  tasks:
    ...
    - name: Extract Token
      ansible.builtin.set_fact:
        token: "{{ token_response.json.enrollmentToken.token }}"
        # token: "{{ token_response.json.data }}"
```

## Running the Playbook
Typical output from the Playbook is shown below:
```sh
$ ansible-playbook deploy-token.yml

PLAY [Deploy CVP Token from CVP Cluster B via CVP Cluster A] **************************************************************************************************

TASK [Fetch Token from Existing CloudVision] ******************************************************************************************************************
ok: [cvaas]

TASK [Extract Token] ******************************************************************************************************************************************
ok: [cvaas]

TASK [Get Device Serial Numbers from CVP] *********************************************************************************************************************
ok: [cvaas] => (item=A2-JS)
ok: [cvaas] => (item=B1-JS)
ok: [cvaas] => (item=B2-JS)
ok: [cvaas] => (item=C1-JS)

TASK [Extract Device Serial Numbers] **************************************************************************************************************************
ok: [cvaas] => (item=15E720A82A6BD89E7521405D1F6F07B6)
ok: [cvaas] => (item=C3FA4A0F0BA1B5D3B6F55B633457414D)
ok: [cvaas] => (item=0DE9D45E17603E4890F6502587775B81)
ok: [cvaas] => (item=FD39181B449532F31BF894E70C5A6C43)

TASK [Template out Change Control] ****************************************************************************************************************************
changed: [cvaas]

TASK [Load Change Control vars] *******************************************************************************************************************************
ok: [cvaas]

TASK [Create a change control on CVP] *************************************************************************************************************************
changed: [cvaas]

PLAY RECAP ****************************************************************************************************************************************************
cvaas                      : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Reconfigure TerminAttr
Finally your `daemon TerminAttr` configuration should be changed on each device to either make use of both clusters simultaneously, or to migrate from the old cluster to the new.