:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml


   **This technote is a work-in-progress.**

Abstract
========

Operative system updates are important because they offer security patches, introduces new features in software, protects the data and improves the performance overall.
The following document consist in several steps to do this updates on our system.

OS Update Playbook
==================

1. Updates should generally occur first in dev, then TTS (stage), before being rolled out to LS/CP.  CP should never be ahead of TTS.

2. Look for hosts in foreman which the puppet agent hasn’t phoned in recently and resolve problems prior to starting OS update. (`os = CentOS and last_report < "2 hours ago" <https://foreman.ls.lsst.org/hosts?search=os+%3D+CentOS+and+last_report+%3C+%222+hours+ago%22&page=1>`__).

   *needs to be revised to work with almalinux*

3. Check for nodes which yum thinks already need an update – this should be “OK” but might be a sign of a past update which failed. (`facts.yum_reboot_required = true <https://foreman.ls.lsst.org/hosts?search=facts.yum_reboot_required+%3D+true&page=1>`__)

4. - Select “live centos nodes” in foreman. (`os = CentOS and last_report > "2 hours ago" <https://foreman.ls.lsst.org/hosts?search=os+%3D+CentOS+and+last_report+%3E+%222+hours+ago%22&page=1>`__)

   - Search for live centos nodes that possibly failed when running. (`os = CentOS and last_report > "2 hours ago" and (status.failed > 0 or status.failed_restarts > 0 or status.skipped > 0) <https://foreman.ls.lsst.org/hosts?search=os+%3D+CentOS+and+last_report+%3E+%222+hours+ago%22+and+%28status.failed+%3E+0+or+status.failed_restarts+%3E+0+or+status.skipped+%3E+0%29&page=1>`__)

      *needs to be revised to work with almalinux*

5. Announce on relevant slack that OS update is starting

6. Run the misc / os update prepare job templates on all nodes. (this will use lot of bandwidth and take some time)
   - This cleans up yum transactions, forces a puppet agent run, and pre-downloads rpms.
   
   NOTE: sometimes some host will fail, check logs for dependency problems on puppet and run command `yum erase -y puppet7-release puppet6-release puppet-release` on failing host.

7. Run :command:`yum update -y` with `os update no-reboot` job template on all nodes

8. Reboot libvirt hypervisors first (Cores). This will cause the VMs on them to be restarted as well and avoid having the VMs down twice (once for the hypervisor and again to apply kernel updates).  The core nodes must only be down one at a time.  Verify that the VMs are accessible again for each node before rebooting the next.

  * Select Core3 (once done choose core 2 then core 1) and schedule Job "Brutal reboot" (usually ping the host to check recovery) then Run the following checklist to verify:

  - Check VM state with :command:`virsh list`

  - Check that DNS is working for resolver VMs. (:command:`dig +short google.com @dns3.tu.lsst.org`)

  - Check that IPA console is accessible for each IPA VM. (*ipa console can take several minutes to come up after a boot*) *add icinga checks for this*

  - Procedure for checking IPA replication. (:command:`ipa-replica-manage list -v <host>`)

  - After all core nodes have been rebooted, check on the rancher cluster

    - :command:`kubectl get nodes` to shows all nodes
    - Rancher dashboard is accessible
    - *add icinga checks for this*

  - Make sure that DHCP is running on the foreman VM

  - Do the same for the next Core till done.

9. Reboot any other VMs (non-core node VMs) 

   *query for the VMs that are on ESXI and reboot them separately is needed*

10. Select “live centos nodes baremetal" in foreman (`os = CentOS and last_report > "2 hours ago" and facts.virtual = physical <https://foreman.ls.lsst.org/hosts?search=os+%3D+CentOS+and+last_report+%3E+%222+hours+ago%22+and+facts.virtual+%3D+physical&page=1>`__)

  -  Select all nodes except for the **core nodes and VMs running on core nodes AND auxtel/comcam/lsstcam nodes** which will be rebooted by the camera/commissioning team (on CP Only).

  -  Reboot all k8s nodes running ceph at the same time to avoid ceph wasting time on recovery.

11. Run misc/brutal reboot scheduled task in Foreman.

  - This is heavy handed but avoids issues in particular with ceph nodes hanging during shutdown.

  - Use foreman rex to run `ls` on nodes as a sanity check that they are back online.

  - Check BMC (for nodes that have a BMC…) for any nodes that fail to come back up

12. As a final check, look for nodes which are flagged as needing a reboot.

13. Announce os update is finished on slack - auxtel-operations (*reboot needed see 10a*)
