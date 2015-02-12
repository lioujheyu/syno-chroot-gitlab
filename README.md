# gitlab on Synology NAS

This is an install note about how to setup gitlab inside the chroot environment of Synology NAS. I experienced this with Xpenology in VM but I think it could work on any synology nas with intel x64 architecture including 415+ or higher 
model. Cause gitlab requires 64-bit support, it's too bad 214play won't work while I planed to use this tiny piece of NAS originally.

So here is the procedure.

1.  Add custom synocommunity package source in DSM package center. You can refer [synocommunity web](https://synocommunity.com/) to do this.
2.  Install debian-chroot from synocommunity in DSM package center. We will use its script but not debian environment because of its 32-bit.
3.  Create your own debian-chroot environment using x64 disto. You could refer [this page](https://sites.google.com/a/courville.org/courville/home/synology-debian-chroot)
    to create, or simply download my version from [here](https://drive.google.com/open?id=0B4a_0yuNmR_FLVlkYXZJcFYxSWc&authuser=0) if your're lazy :)
4.  Replace the whole chroot directory in DSM under /volume1/@appstore/debian-chroot/var/chroottarget. However, it will be deleted once you uninstall debian-chroot from DSM package center. So I suggest putting this environment elsewhere
    and use soft-link or mount-bind on this.
5.  Enter chroot env and setup all necessary thing.

```bash
    /var/packages/debian-chroot/scripts/start-stop-status chroot
```
        