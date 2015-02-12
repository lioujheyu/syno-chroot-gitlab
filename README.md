# gitlab on Synology NAS

I always dream to setup a gitlab service on my own NAS. At first, I used ipkg and try to recompie everything from scrach but quickly fall into dependency hell until I found chroot. Thanks to the chroot, I only have to modify configuration file without compiling package.

So I write down ihis install note about how to setup gitlab inside the chroot environment of Synology NAS. I install it on Xpenology in VM, but I think it could work on any synology nas with intel x64 architecture including 412+, 415+ or other higher model. Cause gitlab requires 64-bit support, it's too bad gitlab won't work on 214play while I planed to use this tiny piece of NAS originally.

Here the procedure goes.

1.  Add custom synocommunity package source in DSM package center. You can refer [synocommunity web](https://synocommunity.com/) to do this.
2.  Install debian-chroot from synocommunity in DSM package center. We will use its script but not debian environment because of its 32-bit.
3.  Create your own debian-chroot environment using x64 disto. You could refer [Courville's site](https://sites.google.com/a/courville.org/courville/home/synology-debian-chroot) to create, or simply download my version from [here](https://drive.google.com/open?id=0B4a_0yuNmR_FLVlkYXZJcFYxSWc&authuser=0) if your're lazy :)
4.  Replace the whole chroot directory in DSM under /volume1/@appstore/debian-chroot/var/chroottarget with our own chroot env in precious step. However, it will be deleted once you uninstall debian-chroot from DSM package center. So I suggest putting this environment elsewhere and use soft-link or mount-bind on this. I will use $CHROOT as your chroot location in the following steps.
5.  Enter chroot env and setup all necessary things.

    ```sh
        /var/packages/debian-chroot/scripts/start-stop-status chroot  # Enter chroot env
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
    
7.  First gitlab configuration
    
    ````sh
        gitlab-ctl reconfigure
    ````

    There is a big problem here. Configure process will detect your system service initial method and try to append initial script into /etc/inittab which doesn't exist in chroot env. This `gitlab-ctl reconfigure` is likely hang in somewhere or pops out some error messages. But don't panic. We only need this step to generate `gitlab.rb` under `/etc/gitlab`. So if you are hang in this step over 10 mins, just `ctrl-c` and continue next step.
    
8.  edit gitlab.rb. You have to change at least nginx web service's port and postgresql's port to avoid conflict with DSM's service.

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
    + postgresql['port'] = 5433                 # DSM has its own postgresql, change the gitlab's postgresql port allow two postgresql instance coexist without data base cross polution
    # postgresql['data_dir'] = "/var/opt/gitlab/postgresql/data"
    ````
    If your machine has less than 2GB ram, you will probably encounter some wield problem while activating gitlab's built-in postgresql. Please also find `postgresql['shared_buffers']` in gitlab.rb and change it to 1MB (This may hurt the performance of gitlab's postgresql but yet to verify).
    
    ````diff
    -# postgresql['shared_buffers'] = "256MB" # recommend value is 1/4 of total RAM, up to 14GB.
    + postgresql['shared_buffers'] = "1MB" # recommend value is 1/4 of total RAM, up to 14GB.
    ````

