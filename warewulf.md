## Setup the machine
- master 
    - change hostname of the machine
        - `hostnamectl set-hostname master`
    - disable firewall
        - `systemctl stop firewalld`
        - `systemctl disable firewalld`
    - to ssh into the master node
        - `ssh -l root 192.168.6.130`
- node
    - change hostname of the machine
        - `hostnamectl set-hostname node`
    - disable firewall
        - `systemctl stop firewalld`
        - `systemctl disable firewalld`
    - `ssh -l root 192.168.6.131`

## to install yamllint (optional)
- install epel9 package
    - `subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms && dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm`
- install yamllint
    - `yum install -y yamllint`

## To remove Warewulf (Optional)
- stop and disable warewulfd
    - `systemctl stop warewulfd`
    - `systemctl disable warewulfd`
- remove warewulf 
    - `sudo dnf remove -y warewulf*`
    - `sudo rm -rf /usr/local/bin/wwctl`
- remove warewulf configuration
    - `sudo rm -rf /etc/warewulf`
    - `sudo rm -rf /var/lib/warewulf`
- cleaning left over logs
    - `sudo rm -rf /var/log/warewulf*`
- remove associated services
    - `sudo dnf remove -y dhcp-server tftp-server nfs-utils`
- remove leftover files by searching for them
    - `find / -name "*warewulf*" 2>/dev/null`
    - `rm -rdf /var/lib/tftpboot/warewulf/`

## to install warewulfd
- installing from rpm
    - `dnf install -y https://github.com/warewulf/warewulf/releases/download/v4.6.0rc2/warewulf-4.6.0rc2-1.el9.x86_64.rpm`
- configure the config file
    ```bash
    ```
- if we geterror related to yaml array we have to make changes in nodes.conf file
    ```yaml
        # file: /etc/warewulf/nodes.conf
            # change this single line  
            args: quiet crashkernel=no
            # change this into list
            args:
            - quiet
            - crashkernel=no
    ```
- another fix we have to apply is in warewulf.conf file
    ```yaml 
        # change subnet mask in line number 2
        # from netmask: 255.255.252.0
        netmask: 255.255.255.0
    ```
- configure the warewulf
    - `wwctl configure --all`
- check if all services are running 
    - `systemctl is-active dhcpd tftp nfs-server warewulfd.service`
    - restart if warewulfd is not running
        - `systemctl start warewulfd`

## configure node 
- to configure node we first need to pull image 
    - `wwctl image import docker://ghcr.io/warewulf/warewulf-rockylinux:9 rockylinux-9 --build`
    - `wwctl profile set default --image rockylinux-9`
- set profile gateway 
    - `wwctl profile set -y default --netmask=255.255.255.0 --gateway=192.168.6.1`
- add node  
    - `wwctl node add n1 --ipaddr=192.168.6.141 --discoverable=true`
- edit node 
    - `wwctl node edit n1`
- node details
    ```yaml
    n1:
    profiles:
        - default
    image name: rockylinux-9
    network devices:
      element:
        hwaddr: 00:0c:29:2c:94:bb
        ipaddr: 192.168.6.141
    resources:
      fstab:
        - file: /home
          freq: 0
          mntops: defaults
          passno: 0
          spec: warewulf:/home
          vfstype: nfs
        - file: /opt
          freq: 0
          mntops: defaults,ro
          passno: 0
          spec: warewulf:/opt
          vfstype: nfs
        - file: /nfs
          freq: 0
          mntops: defaults
          passno: 0
          spec: warewulf:/nfs
          vfstype: nfs
    ```
## installing SLURM
- visit link : `https://slurm.schedmd.com/quickstart_admin.html#quick_start`
- install munge
    - `yum install munge`
- download the slurm package
    - `wget https://download.schedmd.com/slurm/slurm-24.11.1.tar.bz2`
    - if wget is not availble the install it using this command
        - `yum install -y wget`
- build the rpm files using rpmbuild
    - to install rpm build and it's dependencies
        - `yum install -y rpm-build`
        - `yum install -y autoconf automake gcc make mariadb-devel munge-libs pam-devel perl perl-devel readline-devel munge-devel`
    - to build rpms use this command
        - `rpmbuild -ta slurm-24.11.1.tar.bz2`
- make a slurm user
    ```bash
        export SLURMUSER=900;
        # add group
        groupadd -g $SLURMUSER slurm;
        # create user
        useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm;
        # check if user exist in the file
        cat /etc/passwd | grep slurm;
    ```
- create log files and directories
    ```bash
        # create spool directory
        mkdir /var/spool/slurm
        
        # transfer ownership to slurm user
        chown slurm:slurm /var/spool/slurm

        # change permissions of the spool directory
        chmod 755 /var/spool/slurm

        # create log directory
        mkdir /var/log/slurm

        # transfer ownership of the log folder
        chown slurm:slurm /var/log/slurm

        # create log files
        touch /var/log/slurm_jobacct.log /var/log/slurm_jobcomp.log

        # transfer ownership of the log files to slurm user
        chown slurm:slurm /var/log/slurm_jobacct.log /var/log/slurm_jobcomp.log
    ```
- install the packages on master
    - `cd /root/rpmbuild/RPMS/x86_64/`
    - `dnf localinstall ./*`
- edit the slurm-config file    
    - `cp slurm.conf.example slurm.conf`
    - `vim /etc/slurm/slurm.config`
        - ```yaml
            #
            ClusterName=warewulf-cluster
            SlurmctldHost=master
            JobAcctGatherFrequency=30
            JobAcctGatherType=jobacct_gather/none
            SlurmctldDebug=info
            SlurmctldLogFile=/var/log/slurmctld.log
            SlurmdDebug=info
            SlurmdLogFile=/var/log/slurmd.log
         ```
- start the controller demon
    - `systemctl start slurmctld`

## setting up slurm user 
```bash
# setting up the munge authentication
export MUNGEUSER=1010
groupadd -g $MUNGEUSER munge
useradd -m -c "MUNGE uid N gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge
export SLURMUSER=1002
useradd -m -c "MUNGE uid N gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge
yum install munge -y
# copy munge key
chown munge:munge /etc/munge/munge.key
chmod 400 /etc/munge/munge.key
systemctl enable munge
systemctl start munge

