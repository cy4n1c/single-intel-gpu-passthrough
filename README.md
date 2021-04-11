# Single Intel GPU passthrough

This repository contains information on running a virtual machine with GPU
passthrough for systems with a single Intel GPU.


## Hardware and software

GPU: Intel  
Passthrough technology: GVT-d  
Host OS: GNU/Linux  
Host firmware: BIOS  
Guest OS: GNU/Linux  
Guest firmware: UEFI (OVMF)  
Virtualization software: QEMU (KVM)  
Guest software: QEMU with KVM, configured using virt-manager  


## QEMU libvirt hook

Since passhrough is done using GVT-d, the host cannot use the GPU while the
guest is using it. For this reason, before starting the guest virtual machine,
the GPU needs to be disassociated from the host. Also, after the guest shuts
down, this needs to be reversed so that the host can run properly again.

This can be achieved by unloading i915 kernel module. To do this, all other
software that is using the module must stop using it. This includes vtconsoles
and the desktop environment. Since Intel audio device also uses the module
(HDMI audio), it must also be unloaded, along with userspace processes like
pulseaudio. After unloading the module, the GPU can be bound to vfio-pci and
used by the guest.

This can be done automatically when starting the guest using a libvirt QEMU
hook script which can also revert the changes after guest shuts down.

## The script

Use the provided [script](qemu) by placing it in `/etc/libvirt/hooks/qemu` on
Debian based systems or integrating it into an existing script. Make sure you
set the name of the virtual machine which requires GPU passthrough in the
`gpu_domains` array.

The recommended way to test the configuration is to connect via SSH from another device and run the hooks manually.

```
# /etc/libvirt/hooks/qemu gpu prepare begin
# /etc/libvirt/hooks/qemu gpu release end
```


## Guest setup

Using virt-manager, create a new virtual machine. At the end of the process,
before pressing `Finish`, click on `Customize configuration before install`. In
the Overview section under Hypervisor Details make sure that Firmware is set to
UEFI (e.g. `UEFI x86_64: /usr/share/OVMF/OVMF_CODE_4M.fd`).

At this point you can install the system without GPU passthrough, but make sure
that the virtual machine name is not in the hook script as you will lose the
ability to control the guest using the host GUI.

In order to switch to using GPU passthrough, add the GPU as a PCI Host Device
and make sure that the hook script contains the virtual machine name. You
should also add input devices like a USB mouse and a USB keyboard. Finally,
make sure that all graphics related virtual hardware is removed from the guest
(Video and Display devices).
