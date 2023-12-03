# Setting up a shared folder

---

## Host configuration

### 1. Enable shared memory
In `virt-manager`, navigate to `Memory` and enable the option `Enable shared memory`.

### 2. Add filesystem device
In `virt-manager`, use `Add Hardware > Filesystem`, change the driver to `virtiofs`, and configure the given options. Alternatively, attach the below XML:
```xml
<filesystem type="mount" accessmode="passthrough">
  <driver type="virtiofs"/>
  <source dir="HOST_DIRECTORY_PATH"/>
  <target dir="GUEST_MOUNT_NAME"/>
  <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
</filesystem>
```
where `HOST_DIRECTORY_PATH` is the directory on the host to passthrough, e.g. `/home/virtio-shared`, and `GUEST_MOUNT_NAME` is the drive name that will appear in the guest.

---

## Guest configuration -- Linux

**If guest is linux** -- Drivers should come with the kernel, if not, install `virtiofs` or the relevant package for your distro. Then, simply mount `GUEST_MOUNT_NAME` with type `virtiofs` with:
```bash
sudo mount -t virtiofs GUEST_MOUNT_NAME /mount/point/path
```

That's all there is to do. Have a blast.

---

## Guest configuration -- Windows -- pt.1: Getting the drivers

**If guest is windows** -- drivers need to be attached to the system somehow. There are two files you will need to download, either on the host or guest, depending on which option you decide to follow. WinFSP can be downloaded [from the WinFSP github](https://github.com/winfsp/winfsp/releases/) and the virtio windows guest tools installer can be downloaded [from RedHat](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/). Select the most recent version and download the `virtio-win-guest-tools.exe` file.

- Option 1 -- Bundle both into an iso. The steps to do this are as follows:

First, make a tmpdir and copy all the files to it. Then, make the tmpdir into an iso (see [mkisofs(8)](https://linux.die.net/man/8/mkisofs) for documentation and [this StackOverflow answer](https://unix.stackexchange.com/a/760651) for flags):
```bash
ISOWRITE=$(mktemp -d)
cp "/path/to/virtio-win-guest-tools.exe" "$ISOWRITE"
cp "/path/to/winfsp.msi" "$ISOWRITE"
mkisofs -v -J -V "SharedFolderGuestTools" -o shared-folder-guest-tools.iso "$ISOWRITE"
```
Attach this iso to the guest in `virt-manager`.

This may seem like a more convoluted method, but for me personally, it is much easier to bundle a single iso and attach it any time I fire up a new Windows vm.

- Option 2 -- Enable spice USB passthrough or similar. Pause the guest, mount a USB to the host, copy these files to that USB, unmount it, then physically disconnect it. Then, unpause the guest, and plug the USB back in. It should be passed through automatically to the guest. Mount it and there should be your files.

- Option 3 -- **Requires an internet connection in the guest.** Download these two files directly in the guest. This might be the easiest, but I often prefer to not let my testbed windows VMs access the internet.

| Method | Requires host internet? | Requires guest internet? |
| ------ | ----------------------- | ------------------------ |
| 1      | Yes*                    | No                       |
| 2      | Yes*                    | No                       |
| 3      | No                      | Yes                      |

*\*Just get the files onto the host somehow. Download directly, or transfer from another computer.*

## Guest configuration -- Windows -- pt.2: Installing and setting up the drivers

Once these files somehow are accessible in the guest, via whatever method you choose, follow the following steps:
1. Boot into the guest.
2. Run the WinFSP installer (`winfsp-[vers].msi`) and the virtio driver installer (`virtio-win-[vers]/virtio-win-guest-tools.exe`).
3. Reboot the guest.
4. When logged back in, open *Device Manager* (`devmgmt.msc`), and under *System Devices*, open the properties panel for the `VirtIO FS Device` device. Verify that the device is "functioning properly" and that the driver is correctly installed and loaded.
5. Open *Services* (`services.msc`), and find `VirtIO-FS Service`. Right-click and select "Start" to start the service.
6. If you wish to automatically start this service at boot, right-click and select "Properties" and set "Startup type" to `Automatic`.
7. The folder should now show up as a new drive, at label `Z:\`, in File Explorer.
