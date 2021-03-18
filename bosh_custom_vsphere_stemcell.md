### How to Customize a BOSH vSphere Stemcell

This is a follow-on to [How to Customize a BOSH
Stemcell](https://engineering.pivotal.io/post/bosh-customize-stemcell/), which
has stale/wrong information (and which is frozen, so I can't update it):

```bash
  # create a directory where we'll place the stemcell and
  # which will be mounted w/in Docker container
HOST_DIR=/tmp/custom
mkdir $HOST_DIR
cd $HOST_DIR
wget https://bosh-core-stemcells.s3-accelerate.amazonaws.com/621.113/bosh-stemcell-621.113-vsphere-esxi-ubuntu-xenial-go_agent.tgz
  # run docker in privileged mode to allow mounts w/in container
  # also, mount /dev w/in container to access blkdevs
  # on Fedora hosts, run `podman` instead of `docker`
docker run -it --rm --privileged=true -v $PWD:/host -v /dev/:/dev fedora
  # we'll need qemu-img to convert .vmdk into mountable img
dnf install -y qemu parted
cd /host
mkdir -p stemcell image
cd stemcell
tar xvf ../bosh-stemcell*.tgz
cd ../image
tar xvf ../stemcell/image
qemu-img convert image-disk1.vmdk -O raw root.img
  # Start with a higher loop device to avoid `losetup` returning a
  # "root.img: failed to set up loop device: Device or resource busy"
  # Ubuntu snaps use loop devices.
LOOP=loop7
sudo losetup /dev/$LOOP root.img
partprobe /dev/$LOOP
mkdir -p /mnt/stemcell
mount /dev/${LOOP}p1 /mnt/stemcell
chroot /mnt/stemcell /bin/bash
  # Hygiene: change stemcell number to avoid confusion
echo -n 621.114 > /var/vcap/bosh/etc/stemcell_version
```

Congrats, you're now inside the stemcell image. Make the changes you need
(probably replacing `/var/vcap/bosh/bin/bosh-agent`, or adding a special user).
If you add a user, remember to add them to the `admin`, `bosh_sudoers`, and
`bosh_sshers` groups.  When you're done, do the following:

```bash
  # exit the chroot'ed bash
exit
umount /mnt/stemcell
losetup -d /dev/$LOOP
rm image-disk1.vmdk
qemu-img convert -o subformat=streamOptimized,adapter_type=lsilogic root.img -O vmdk image-disk1.vmdk
rm root.img
SHASUM=($(sha1sum image-disk1.vmdk))
sed -i "s/vmdk.= .*\$/vmdk)= $SHASUM[1]/" image.mf
tar czvf ../stemcell/image *
cd ../stemcell
vi stemcell.MF
tar czvf ../custom_stemcell_621.114.tgz *
  # exit the container
exit
```

Voilà, you're done, and your new stemcell is available in
`/tmp/custom/custom_stemcell_621.114.tgz`
