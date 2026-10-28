# SOP: Enable and Troubleshoot QEMU Guest Agent for Windows VMs in OpenStack

## 1. Purpose

This SOP describes how to enable QEMU Guest Agent support for Windows
Server VMs in OpenStack and troubleshoot the error:

``` text
error: argument unsupported: QEMU guest agent is not configured
```

It covers image preparation, Windows service verification, libvirt
channel verification, and end-to-end testing.

## 2. Scope and environment

Validated in the following environment during troubleshooting:

-   OpenStack Nova: `29.2.0`
-   Hypervisor: KVM/libvirt
-   Guest OS: Windows Server (64-bit; tested with Windows Server 2025)
-   Glance image: `Windows-Server-2022-RAW`
-   Guest agent service name: `QEMU-GA`

> **Note:** The image name may say Windows Server 2022 even when the
> installed guest OS is Windows Server 2025. Confirm the actual guest OS
> before applying this SOP elsewhere.

## 3. How it works

QEMU Guest Agent requires all of the following:

1.  The QEMU Guest Agent software is installed and running inside
    Windows.
2.  The Glance image has the property `hw_qemu_guest_agent=yes` before a
    new instance is created.
3.  Nova/libvirt creates a virtio-serial controller and a channel named
    `org.qemu.guest_agent.0`.
4.  The agent responds to commands from the hypervisor.

Setting the Glance property does **not** reliably add the channel to an
already-existing VM. In the observed environment, creating a new
instance from the updated image caused Nova to add the channel
automatically. Verify this behavior in your own deployment.

## 4. Prerequisites and safety

-   OpenStack CLI credentials with permission to inspect and update
    Glance images.
-   Shell access to the compute node hosting the VM.
-   Administrator access inside Windows.
-   A maintenance window if the VM must be recreated.
-   A verified backup or recovery plan before deleting or rebuilding any
    existing VM.

**Do not delete the existing VM until you have verified backups and
recorded its boot source, attached volumes, networks, fixed/floating
IPs, security groups, flavor, metadata, and other required
configuration.** Recreating a VM can cause data loss or change its
networking identity.

## 5. Step A --- Verify the Windows Guest Agent

Inside the Windows VM, open PowerShell as Administrator:

``` powershell
Get-Service QEMU-GA
```

Expected status:

``` text
Status   Name      DisplayName
------   ----      -----------
Running  QEMU-GA   QEMU Guest Agent
```

If the service is installed but stopped:

``` powershell
Set-Service -Name QEMU-GA -StartupType Automatic
Start-Service -Name QEMU-GA
Get-Service QEMU-GA
```

If the service is not installed, mount the trusted VirtIO-Win ISO, open
its `guest-agent` directory, and run the appropriate 64-bit MSI. Obtain
the ISO from the official VirtIO-Win project source and verify
organizational software policies before installing it.

## 6. Step B --- Set the Glance image property

List or inspect the image:

``` bash
openstack image show "Windows-Server-2022-RAW" -c id -c name -c properties
```

Set the property:

``` bash
openstack image set \
  --property hw_qemu_guest_agent=yes \
  "Windows-Server-2022-RAW"
```

Verify it through the `properties` field:

``` bash
openstack image show "Windows-Server-2022-RAW" -c properties
```

Expected property in the output:

``` text
hw_qemu_guest_agent='yes'
```

Some OpenStack CLI versions do not expose image properties as individual
selectable columns. If `-c hw_qemu_guest_agent` returns "No recognized
column names," use `-c properties`; this does not mean setting the
property failed.

## 7. Step C --- Identify the VM and compute host

On a controller or a host with OpenStack CLI credentials:

``` bash
openstack server show <INSTANCE_UUID> \
  -c name \
  -c status \
  -c image \
  -c OS-EXT-SRV-ATTR:host
```

Record the VM UUID, name, status, image, and compute host.

SSH to the reported compute host. Find the libvirt domain name using the
instance UUID if needed:

``` bash
virsh list --all
```

In many deployments, the domain name is similar to `instance-000000b6`;
do not assume this name is the same on every host.

## 8. Step D --- Check the libvirt guest-agent channel

On the compute host:

``` bash
virsh dumpxml <DOMAIN_NAME> | grep -A6 -B2 'org.qemu.guest_agent'
```

A configured channel should resemble:

``` xml
<channel type='unix'>
  <source mode='bind' path='/var/lib/libvirt/qemu/org.qemu.guest_agent.0.<DOMAIN_NAME>.sock'/>
  <target type='virtio' name='org.qemu.guest_agent.0' state='connected'/>
  <alias name='channel0'/>
  <address type='virtio-serial' controller='0' bus='0' port='1'/>
</channel>
```

The exact socket path, alias, address, and state may vary. The important
item is the target name:

``` text
org.qemu.guest_agent.0
```

Also inspect controllers:

``` bash
virsh dumpxml <DOMAIN_NAME> | grep -E "controller type=|channel type=|org.qemu.guest_agent"
```

If no guest-agent channel appears, continue to Step E.

## 9. Step E --- If the channel is missing

1.  Confirm that `hw_qemu_guest_agent=yes` is set on the image used for
    instance creation.
2.  Confirm that the instance was created after the property was set.
3.  Do not assume a stop/start or a guest OS reboot will add a missing
    channel to an existing domain.
4.  Check Nova compute configuration and logs if a newly created
    instance still lacks the channel.
5.  Do not manually edit the live libvirt XML as a first-line fix. Nova
    may overwrite manual changes, and live hardware changes can be
    disruptive.

For an existing VM, prefer a controlled, supported Nova remediation or
create a replacement instance from the updated image. If recreating:

-   Back up and verify important data first.
-   Record flavor, boot source, volumes, networks, security groups,
    addresses, metadata, and any special settings.
-   Create a parallel test instance when possible.
-   Verify guest-agent communication before cutting over.
-   Keep the original VM until the replacement is fully validated.

### Example Nova configuration inspection

On the compute host, if Nova is installed directly on the host:

``` bash
sudo grep -n -E '^\[libvirt\]|^virt_type|^compute_driver' /etc/nova/nova.conf
nova-manage --version
```

If Nova runs in containers, inspect the effective configuration and logs
inside the relevant Nova compute container using your deployment's
standard tooling. Do not change `nova.conf` without evidence that a
configuration change is needed.

## 10. Step F --- Test communication from the compute host

Run:

``` bash
virsh qemu-agent-command <DOMAIN_NAME> \
  '{"execute":"guest-ping"}'
```

Expected response:

``` json
{"return":{}}
```

Optionally request agent details:

``` bash
virsh qemu-agent-command <DOMAIN_NAME> \
  '{"execute":"guest-info"}'
```

A successful `guest-ping` confirms that the host can communicate with
the guest agent.

## 11. Troubleshooting reference

  ----------------------------------------------------------------------------------------------------------------
  Symptom                                Likely cause               Next action
  -------------------------------------- -------------------------- ----------------------------------------------
  `QEMU guest agent is not configured`   Guest-agent channel is     Verify image property and channel; for
                                         missing from libvirt       existing VM, use a supported remediation or
                                         domain XML                 recreate safely

  `guest-ping` times out or agent is     Channel exists, but agent  Check Windows `QEMU-GA` service, guest OS
  unresponsive                           is not responding          health, and channel state

  `QEMU-GA` service not found            Agent is not installed, or Install the correct VirtIO-Win guest-agent MSI
                                         service name differs       and verify service name

  Image property is not shown by         CLI does not expose custom Use
  `-c hw_qemu_guest_agent`               properties as standalone   `openstack image show <IMAGE> -c properties`
                                         columns                    

  New instance still has no channel      Image property may not be  Check the actual image ID used, Nova compute
                                         applied, or Nova           logs/configuration, and deployment-specific
                                         configuration/deployment   support
                                         behavior differs           
  ----------------------------------------------------------------------------------------------------------------

## 12. Final validation checklist

-   [ ] Windows `QEMU-GA` service exists.
-   [ ] Windows `QEMU-GA` service is running and set to automatic
    startup.
-   [ ] Glance image property `hw_qemu_guest_agent=yes` is present.
-   [ ] VM was created from the intended image after the property was
    set, or an approved remediation was applied.
-   [ ] Libvirt XML contains `org.qemu.guest_agent.0`.
-   [ ]
    `virsh qemu-agent-command <DOMAIN_NAME> '{"execute":"guest-ping"}'`
    returns `{"return":{}}`.
-   [ ] VM data, networking, and workload are validated after any
    replacement or maintenance.

## 13. Incident notes template

Record these details during future troubleshooting:

``` text
Date/time:
Operator:
Project/region:
VM name:
VM UUID:
Guest OS/version:
Glance image name and ID:
Compute host:
Nova version:
QEMU Guest Agent service status:
Image property hw_qemu_guest_agent:
Libvirt channel present (yes/no):
guest-ping result:
Changes made:
Backup/recovery verified:
Final status:
```

## 14. Official references

-   VirtIO-Win project:
    https://github.com/virtio-win/virtio-win-pkg-scripts
-   VirtIO-Win QEMU Guest Agent documentation:
    https://virtio-win.github.io/Knowledge-Base/Qemu-ga-win.html
-   OpenStack image CLI documentation:
    https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/image.html
