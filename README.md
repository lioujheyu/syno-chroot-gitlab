# GitLab on Synology NAS

I always dream to setup a gitlab service on my own NAS. At first, I used ipkg and try to recompile everything from scratch but quickly fall into dependency hell until I found chroot. Thanks to the chroot, I only have to modify gitlab's configuration file without building the huge number of package. It also frees me from the old 4.2.3 gcc inside the ipkg and let the NAS be more closed to a real linux.

So I write down this installation note about how to setup gitlab inside the chroot environment of Synology NAS. I've try this on a XPEnology in VM, but think it could also work on any synology NAS with **x64 architecture CPU** including 412+, 415+ or other higher model. Cause gitlab requires 64-bit support, it's too bad gitlab won't work on 214play (32-bit Intel Atom) while I planned to use this tiny piece of NAS originally.


Here the procedure goes.

1.  Add custom synocommunity package source in DSM package center. You can refer [synocommunity web](https://synocommunity.com/) to do this.
2.  Install debian-chroot from synocommunity in DSM package center. We will use its script but not debian environment because of its 32-bit. You also have to install python 2.7 from synocommunity, not from official synology python package.
3.  Create your own debian-chroot environment using x64 disto. You could refer [Courville's site](https://sites.google.com/a/courville.org/courville/home/synology-debian-chroot) to do so, or simply download my version from [here](https://drive.google.com/open?id=0B4a_0yuNmR_FLVlkYXZJcFYxSWc&authuser=0).
4.  Replace the whole chroot directory in DSM under `/volume1/@appstore/debian-chroot/var/chroottarget` with our own chroot env from previous step. However, this directory will be deleted once you uninstall debian-chroot from DSM package center. So I suggest putting this environment elsewhere and use soft-link or mount-bind on this.
5.  Use debian-chroot script to mount every thing we need, and then enter chroot env to setup all necessary things.

    ```sh
    /var/packages/debian-chroot/scripts/start-stop-status start     # This script will automaticlly mount the resource chroot needs
    /var/packages/debian-chroot/scripts/start-stop-status chroot    # Enter chroot env
    unset LD_LIBRARY_PATH
    /debootstrap/debootstrap --second-stage
    passwd root                                # Set your root password
    vi /etc/apt/sources.list                   # Fill you own debian package repositories
    aptitude update
    aptitude install openssh-server vim ...... # and anything else you want to install
    aptitude install locales                   # Configure the right localesc
    dpkg-reconfigure locales
    vim /etc/ssh/sshd_config                   # Change the chroot ssh port to avoid conflict with DSM ssh server. ex: 2222
    service ssh restart                        # Now you could directly ssh into your chroot env.
    ```
6.  Download and install gitlab from [gitlab.com](https://about.gitlab.com/downloads/).
    
    ````sh
    wget https://downloads-packages.s3.amazonaws.com/debian-7.8/gitlab_7.7.2-omnibus.5.4.2.ci-1_amd64.deb
    dpkg -i gitlab_7.7.2-omnibus.5.4.2.ci-1_amd64.deb
    ````
    
7.  Edit `/etc/gitlab/gitlab.rb`. You have to change at least nginx web service's port and postgresql's port to avoid conflict with DSM's service.

    ````diff
    vim /etc/gitlab/gitlab.rb
    --- /opt/gitlab/etc/gitlab.rb.template  2015-01-20 18:13:07.000000000 +0800
    +++ gitlab.rb   2015-02-09 15:14:05.306926190 +0800
    @@ -1,7 +1,7 @@
    -external_url 'GENERATED_EXTERNAL_URL'
    +external_url 'http://192.168.8.131:8123'   # Fill your own server address, remember to change the port to other than 22

    @@ -252,11 +252,11 @@
    -# postgresql['enable'] = true
    + postgresql['enable'] = true
    # postgresql['listen_address'] = nil
    -# postgresql['port'] = 5432
    + postgresql['port'] = 5433                 # DSM has its own postgresql, changing the gitlab's postgresql port allow two postgresql instance coexist without data base pollution.
    # postgresql['data_dir'] = "/var/opt/gitlab/postgresql/data"
    ````
    Because of shared memory setting in Synology DSM I'm not sure, you will probably encounter some wield problems while activating gitlab's built-in postgresql. Please also find `postgresql['shared_buffers']` in gitlab.rb, uncomment it and change it to 1MB (This may hurt the performance of gitlab's postgresql but yet to verify. Sorry I'm not expert in postgresql).
    
    ````diff
    -# postgresql['shared_buffers'] = "256MB" # recommend value is 1/4 of total RAM, up to 14GB.
    + postgresql['shared_buffers'] = "1MB" # recommend value is 1/4 of total RAM, up to 14GB.
    ````

8.  First gitlab configuration
    
    ````sh
    gitlab-ctl reconfigure
    ````

    Here is a problem. Configure process will detect your system service initial method and try to append initial script into /etc/inittab which doesn't exist in chroot env. This `gitlab-ctl reconfigure` is very likely to hang in somewhere or pop out some error messages. If you are hang in this step over 10 mins, just `ctrl-c` and continue next step.
    
9. Manually start all gitlab related services. After that, perform second gitlab reconfigure. This time we expect no error message.

    ````sh
    /opt/gitlab/embedded/bin/runsvdir-start &
    gitlab-ctl reconfigure
    ````

10. Check if gitlab is running smoothly

    ````sh
    gitlab-ctl status
    ````
    
    It will show something like
    
    ````
    run: logrotate: (pid 1394) 76s; run: log: (pid 1393) 76s
    run: nginx: (pid 1374) 82s; run: log: (pid 1373) 82s
    run: postgresql: (pid 1274) 95s; run: log: (pid 1273) 95s
    run: redis: (pid 1190) 100s; run: log: (pid 1189) 100s
    run: sidekiq: (pid 1356) 84s; run: log: (pid 1355) 84s
    run: unicorn: (pid 1333) 85s; run: log: (pid 1332) 85s
    ````
    
    If there is still anything odd, use `gitlab-rake gitlab:check` to check the error message. Then google it !!
    
Congradulation!! Now use your web browser to bring up the gitlab. Don't forget the port you set in the gitlab.rb. If all look good, now it's time to add script let DSM running gitlab after every time it reboots.

1.  back into DSM shell and edit `/var/packages/debian-chroot/scripts/start-stop-status`.  

    ````diff
    exit                    # exit chroot
    vi /var/packages/debian-chroot/scripts/start-stop-status
    
    @@ -22,6 +22,7 @@

            # Start all services
            ${INSTALL_DIR}/app/start.py
    +       chroot $CHROOTTARGET/ /opt/gitlab/embedded/bin/runsvdir-start &
        fi
    }
    ````
    
2.  edit `/etc/rc.local`. Doing this will definitely result DSM Security Advisor complains about the malicious startup script. Just leave it or turn this checking item off.

    ````sh
    echo -e "#! bin/sh\n/var/packages/debian-chroot/scripts/start-stop-status start " > /etc/rc.local
    ````
    
Now reboot your NAS and see if gitlab is worlking after reboot.

## Reference

*   https://sites.google.com/a/courville.org/courville/home/synology-debian-chroot
*   http://www.rooot.net/en/geek-stuff/synology/39-chroot-debian-synology-debootstrap.html
*   http://www.hang321.net/2014/08/16/debian-chroot-on-dsm/
*   https://gitlab.com/gitlab-org/omnibus-gitlab/issues/129






