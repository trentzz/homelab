# Setting Up Your Services VM

For the majority of services, we are going to use a single VM that runs all of the docker containers/services/scripts. This way we can have more dynamic resource paritioning across all of our services.

I would recommend using something stable like `debian` or `fedora`, whatever you're most familiar with. Then make sure you provision enough resources, this will depend on how many services and what kind of resources you are planning on running. For my VM I gave it 6 cores and 42GB of memory. YMMV.

## Setting up a new service

For each service, we are going to make a new shared folder in OMV and share it with this VM over nfs. This way each of the services' data are separated and the setup is more flexible.

For this suppose we're making a shared folder under the parent `SERVICES` folder in OMV: `/SERVICES/EXAMPLE`.

1. Make a new folder in OMV (and make it read/write for  most of the cases)
2. Share it the VM using NFS.
3. Check the connection works on the services VM using `sudo mount -t nfs 10.10.1.XX:/SERVICES/EXAMPLE /mnt/example`
4. Update your `/etc/fstab` to add that entry so that it auto mounts on boot.
