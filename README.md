About
-----

This is a set of tools that can be used to create, enable or disable chroot jails for web and shell hosting purposes.

Requirements
------------

In order for jail tools to work it is necessary to have a server with Debian Wheezy or Debian Jessie installed. In the case of Debian Jessie it is also required to replace systemd with sysvinit since systemd makes a spectacular mess with bind mounts:

    apt-get install sysvinit sysvinit-core
    reboot

Installation
------------

Install mandatory packages. At the very least we need git libpam-chroot to install software and confine ssh users to their jails.

    apt-get install git libpam-chroot debootstrap nginx uwsgi uwsgi-core uwsgi-emperor uwsgi-plugins-python uwsgi-plugins-php uwsgi-emperor mysql-server python-jinja2

Fetch jail code and make necessary links and groups.

    git clone https://github.com/phalaaxx/jail
    cd jail
    python setup.py install
    jctl --groups-setup

Enable jail init script. This script is used to bind-mount jail directories at boot time and umount them before shutdown.

    update-rc.d jail defaults
    update-rc.d jail enable

Make necessary directories.

    mkdir -p /jail/{base,root,home}

Download chroot environment and install additional software within the chroot environment:

    jctl --chroot-setup
    chroot /jail/base apt-get install vim tmux zsh


GRSecurity
----------

This is a simple introduction how to configure and install grsecurity patched kernel under debian/ubuntu. This step is optional though it is highly recommended.
Make sure you have sufficient space in /usr/src for kernel compilation before you proceed with the next step.
First, download necessary packages.

    apt-get install kernel-package libncurses5-dev attr

Download grsecurity patch from [http://grsecurity.net/download_stable.php](http://grsecurity.net/download_stable.php) - at the time of writing latest stable version is 2.9.1-3.2.51-201309181906.

    cd /usr/src
    wget http://grsecurity.net/stable/grsecurity-2.9.1-3.2.51-201309181906.patch

Download and extract appropriate kernel version.

    wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.2.51.tar.xz
    tar Jxvf linux-3.2.51.tar.xz

Patch kernel sources with grsecurity patch.

    cd linux-3.2.51
    patch -p1 < ../grsecurity-2.9.1-3.2.51-201309181906.patch

Configure kernel sources.

    make menuconfig

Example setup has the following grsecurity options enabled (all options are under Security Options/Grsecurity/Customize Configuration). Options are for custom configuration method.

#### PaX ####
* Enable various PaX features
* PaX Control
  * Use filesystem extended attributes marking
* Non-executable pages
  * Enforce non-executable pages
  * Paging based non-executable pages
  * Emulate trampolines
  * Restrict mprotect()
* Address Space Layout Randomization
  * Address Space Layout Randomization
  * Randomize kernel stack base
  * Randomize user stack base
  * Randomize mmap() base
* Miscellaneous hardening features
  * Sanitize kernel stack
  * Prevent various kernel object reference counter overflows
  * Harden heap object copies between kernel and userland
  * Prevent various integer overflows in function size parameters
  * Generate some entropy during boot

#### Memory Protections ####
* Deny reading/writing to /dev/kmem, /dev/mem, and /dev/port
* Disable privileged I/O
* Harden BPF JIT against spray attacks
* Disable unprivileged PERF\_EVENTS usage by default
* Insert random gaps between thread stacks
* Harden ASLR against information leaks and entropy reduction
* Deter exploit bruteforcing
* Harden module auto-loading
* Hide kernel symbols
* Active kernel exploit response

#### Role Based Access Control Options ####

#### Filesystem Protections ####
* Proc restrictions
  * Restrict /proc to user only
* Additional restrictions
* Linking restrictions
* Kernel-enforced SymlinksIfOwnerMatch
  * GID for users with kernel-enforced SymlinksIfOwnerMatch (990)
* FIFO restrictions
* Sysfs/debugfs restriction
* Runtime read-only mount protection
* Eliminate stat/notify-based device sidechannels
* Chroot jail restrictions
  * Deny mounts
  * Deny double-chroots
  * Deny pivot\_root in chroot
  * Enforce chdir("/") on all chroots
  * Deny (f)chmod +s
  * Deny fchdir out of chroot
  * Deny mknod
  * Deny shmat() out of chroot
  * Deny access to abstract AF\_UNIX sockets out of chroot
  * Protect outside processes
  * Restrict priority changes
  * Deny sysctl writes
  * Capability restrictions

#### Kernel Auditing ####
* /proc/<pid>/ipaddr support

#### Executable Protections ####
* Dmesg(8) restriction
* Deter ptrace-based process snooping
* Require read access to ptrace sensitive binaries
* Enforce consistent multithreaded privileges
* Trusted Path Execution (TPE)
  * Partially restrict all non-root users
  * GID for TPE-untrusted users (991)

#### Network Protections ####
* Larger entropy pools
* TCP/UDP blackhole and LAST\_ACK DoS prevention
* Disable TCP Simultaneous Connect
* Socket restrictions
  * Deny any sockets to group
  * GID to deny all sockets for (992)
  * Deny client sockets to group
  * GID to deny client sockets for (993)
  * Deny server sockets to group
  * GID to deny server sockets for (994)

Make sure to tune these settings to your liking. These settings may not work for your case.
Finally compile grsecurity kernel.

    export CONCURRENCY_LEVEL=4
    make-kpkg --initrd --revision 1 kernel_image

Depending on hardware this may take up to several hours. Once the kernel is ready - install it.

    cd /usr/src
    dpkg -i linux-image-3.2.51-grsec_1_amd64.deb

At this point the server needs to be restarted. Once it boots make sure that the new kernel has been loaded.

    uname -r

If the server was booted into the new kernel. you should see something similar to _3.2.51-grsec_.
When the new kernel is running it is possible that some updates may fail due to grsecurity restrictions. The reason is because grub will not be able to update its configuration. In order to resolve this issue, two binaries from grub package need to be given correct permissions.

    setfattr -n user.pax.flags -v pme /usr/bin/grub-script-check
    setfattr -n user.pax.flags -v pme /usr/sbin/grub-probe


gradm2
------

Download required packages for compilation:

    apt-get install bison flex

Download, compile and install gradm2:

    cd /usr/src
    wget http://grsecurity.net/stable/gradm-2.9.1-201309161709.tar.gz
    tar zxf gradm-2.9.1-201309161709.tar.gz
    cd gradm2
    make
    make install


Usage
-----

In order to start using chroot jails you need some users in the jail group.

    juser --useradd test@example.com
    passwd test@example.com

It is also necessary to add www-data user to the new user's group:

    usermod -aG test@example.com www-data

Update and mount jail directories.

    juser --update
    juser --mount test@example.com
    juser --list


Sample vhost
------------

In order to make virtual hosts confined inside chroot jails, use nginx configuration similar to this.


    juser --vhost-json << EOF
    {
        "UserName"      : "test@example.com",
        "vhosts": [
            {
                "Name"  : "test.example.com",
                "Aliases" : []
            }
        ]
    }
    EOF

The above command will generate a configuration file /etc/nginx/sites-enabled/test@example.com.conf like this:

	# /etc/nginx/sites-enabled/test@example.com.conf
	server {
	    listen 80;

	    root /home/test@example.com/public_html;
	    server_name test.example.com;
	    index index.php index.html;

	    location ~ \.php$ {
	        include uwsgi_params;
	        uwsgi_modifier1 14;
	        uwsgi_pass unix:///jail/root/test@example.com/run/uwsgi/app/php-uwsgi.sock;
	    }
	}


LICENSE
-------

BSD License http://opensource.org/licenses/bsd-license
