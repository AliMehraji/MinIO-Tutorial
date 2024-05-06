# Minio Cluster With Vagrant

The `qemu/kmv` virtualization and `vagrant` need to be installed in Host.

## Download `Minio` Package

Download `mino` package in Host, It will be copied to Vagrant VMs.

```bash
  curl  -L https://dl.min.io/server/minio/release/linux-amd64/minio_20240501011110.0.0_amd64.deb -o /tmp/minio_20240501011110.0.0_amd64.deb
```

  > For the latest version check the Minio Download page for Linux [Minio-Download-Page][1]
  >
  > If You can't download , set proxy for `curl` `--proxy "socks5h://<PROXY-SERVER-IP>:<PROXY-SERVER-PORT>"`
  >
example :

  ```bash
  curl --proxy "socks5h://127.0.0.1:10808" -L https://example.com/package.deb -o /path/to/package
  ```

## Vagrant

Vagrant VMs IPs :

> It Depends On Host Virtualization `qemu/kvm` IP Range. it could be changed.
> 
> To find the ip range (by default the link name is `virbr0`) , invoke `ip addr show virbr0`.

- 192.168.122.31
- 192.168.122.32
- 192.168.122.33
- 192.168.122.34

### Vagrantfile

This Vagrantfile is for `libvirt` via `qemu/kvm` Virtualization.

Please Read the Comments in Vagrantfile before applying it.
Invoke `vagrant up` and wait all VMs be provisioned.

```vagrantfile
# Latest Version Of Minio Is Present in Host
MINIO_DEB_LATEST_VERSION="minio_20240501011110.0.0_amd64.deb"

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.ssh.insert_key = false
  config.vm.synced_folder ".", "/vagrant", disabled: true
  # Copy Minio-Package From Host To VMs
  minio_deb_package_path = "/tmp/#{MINIO_DEB_LATEST_VERSION}"
  config.vm.provision "file", source: minio_deb_package_path , destination: minio_deb_package_path 
  # Step by Step Shell commands to prepare the VMs 
  # The Vagrant User is in Sudoer Group by default.
  config.vm.provision "shell", inline: <<-SHELL
    # Add Your Host SSH Key To VMs to connect them via ssh 
    echo '<HOST-PUBLIC-SSH-KEY>' >> /home/vagrant/.ssh/authorized_keys
    # Changing Mod and OwnerShip of authorized key to be readable just for vagrant user
    chmod 600 /home/vagrant/.ssh/authorized_keys
    chown vagrant:vagrant /home/vagrant/.ssh/authorized_keys
    # Adding Proxy Server Address for Apt Because of Filterring If You Need because Of Not Free Internet
    # If You Have Free Internet , Comment below line
    echo 'Acquire::http::Proxy "socks5h://<PROXY-SERVER-IP>:<PROXY-SERVER-PORT>";' >> /etc/apt/apt.conf.d/proxy
    # Update And Upgrade VMs
    sudo apt update -y && sudo apt upgrade -y
    # Adding Minio servers IP to /ets/host to implement DNS Server.
    # The Static IPs which going to be assigned to VMs.
    sudo echo -en "192.168.122.31 minio-01\n192.168.122.32 minio-02\n192.168.122.33 minio-03\n192.168.122.34 minio-04\n" >> /etc/hosts
    # Adding User and group for minio
    sudo useradd -M -r -s /usr/sbin/nologin minio-user
    # changing TimeZone To Asia/Tehran
    sudo timedatectl set-timezone Asia/Tehran
    # Installing Minio Which it's package is downloaded in host and copied to VMs.
    sudo dpkg -i #{minio_deb_package_path }
    # Creating Directory For Mounting disks in vms from their /etc/fstab after provisioning the VMs.
    sudo mkdir /mnt/disk{1..2}
  SHELL

  config.vm.provider :libvirt do |libvirt|
    libvirt.memory = 2048
    libvirt.cpus = 2  # Set the number of CPUs
    libvirt.storage :file, :size => '2G', :type => 'qcow2' # Additional Disk
    libvirt.storage :file, :size => '2G', :type => 'qcow2'
    # v.cpusize = 1024  # Set the CPU execution cap (in percentage)
  end

  # Minio Server 1
  config.vm.define "Minio-Server-1" do |app1|
    app1.vm.hostname = "minio-01"
    app1.vm.network :private_network, type: "static", ip: "192.168.122.31", bridge: "virbr0"
  end

  # Minio Server 2
  config.vm.define "Minio-Server-2" do |app2|
    app2.vm.hostname = "minio-02"
    app2.vm.network :private_network, type: "static", ip: "192.168.122.32", bridge: "virbr0"
  end

  # Minio Server 3
  config.vm.define "Minio-Server-3" do |app3|
    app3.vm.hostname = "minio-03"
    app3.vm.network :private_network, type: "static", ip: "192.168.122.33", bridge: "virbr0"
  end

  # Minio Server 4
  config.vm.define "Minio-Server-4" do |app4|
    app4.vm.hostname = "minio-04"
    app4.vm.network :private_network, type: "static", ip: "192.168.122.34", bridge: "virbr0"
  end
end
```

## Connect To VMs

There is two way to connect to Provisioned VMs:

1 - `ssh vagrant@VM-IP`</br></br>
  > The Host public ssh key has to be added to VMs via Vagrantfile.
  example:

  ```bash
  ssh vagrant@192.168.121.34
  ```

2 - `vagrant ssh <Vagrant-VM-Name>` </br></br>
  example:

  ```bash
  vagrant ssh Minio-Server-4
  ```

## Mount Disks 

- At first invoke `lsblk` to list the disks which is added to VMs via Vagrant
  
  The `vdb` and `vdc` disks were added.

  ```txt
  NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
  loop0                       7:0    0  63.4M  1 loop /snap/core20/1974
  loop1                       7:1    0  53.3M  1 loop /snap/snapd/19457
  loop2                       7:2    0 111.9M  1 loop /snap/lxd/24322
  loop3                       7:3    0  38.7M  1 loop /snap/snapd/21465
  loop4                       7:4    0    87M  1 loop /snap/lxd/28373
  vda                       252:0    0   128G  0 disk 
  ├─vda1                    252:1    0     1M  0 part 
  ├─vda2                    252:2    0     2G  0 part /boot
  └─vda3                    252:3    0   126G  0 part 
    └─ubuntu--vg-ubuntu--lv 253:0    0    63G  0 lvm  /
  vdb                       252:16   0     2G  0 disk
  vdc                       252:32   0     2G  0 disk
  ```

- become root/sudoer via `sudo su` and make `XFS` File system and add Labels `DISK1` and `DISK2` on the two disks (Minio Highly Recommends to use `XFS`)

  ```bash
  mkfs.xfs /dev/vdb -L DISK1
  mkfs.xfs /dev/vdc -L DISK2
  ```

- Add Two `XFS` Disk at the end of `/etc/fstab`  file to be permanently mounted in reboot , `vi /etc/fstab`
  
  ```conf
  LABEL=DISK1      /mnt/disk1     xfs     defaults,noatime  0 2
  LABEL=DISK2      /mnt/disk2     xfs     defaults,noatime  0 2
  ```

- Mount Disks from `/etc/fstab` , the `/mnt/disk1` and `/mnt/disk2` have been created via Vagrant in VMs.
  
  ```bash
  mount -av
  ```

- Change Ownership of those Directories to `minio-user`
  
  ```bash
  chown minio-user:minio-user /mnt/disk1/minio
  ```

- Create `minio` Directory in `/mnt/disk1` and `/mnt/disk2`.
  
  ```bash
  mkdir /mnt/disk1/minio
  mkdir /mnt/disk2/minio
  ```

## [ Minio Systemd Service][2]

After connecting to each VM, add a minio systemd service in `/etc/systemd/system`

```bash
vi /etc/systemd/system/minio.service
```

The User and Group in service file has to be `minio-user`

```conf
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio
AssertFileNotEmpty=/etc/default/minio

[Service]
Type=notify

WorkingDirectory=/usr/local/

User=minio-user
Group=minio-user
ProtectProc=invisible

EnvironmentFile=-/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

# Let systemd restart this service always
Restart=always

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=1048576

# Specifies the maximum number of threads this process can create
TasksMax=infinity

# Disable timeout logic and wait until process is stopped
TimeoutSec=infinity

SendSIGKILL=no

[Install]
WantedBy=multi-user.target
```

## Environments For Minio

in the above systemd service, Minio loads the environment variables file from `/etc/default/minio` , create that file and add below contents to it: 

```conf
MINIO_VOLUMES="http://minio-0{1...4}:9000/mnt/disk{1...2}/minio"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER=<USER-NAME>
MINIO_ROOT_PASSWORD=<STRONG-PASSWORD>
MINIO_SERVER_URL="http://minio-01:9000"
```

## Start And Enable Minio Service

```bash
systemctl start minio.service
```

Enable The service in Reboot/Restart.

```bash
systemctl enable minio.service
```

status of the minio service , it has to be in active and running status.

```bash
systemctl status minio.service
```

```txt
     Loaded: loaded (/etc/systemd/system/minio.service; disabled; vendor preset: enabled)
     Active: active (running) since Sun 2024-05-05 19:32:26 +0330; 2h 9min ago
       Docs: https://docs.min.io
    Process: 4198 ExecStartPre=/bin/bash -c if [ -z "${MINIO_VOLUMES}" ]; then echo "Variable MINIO_VOLUMES not set in /etc/default/minio"; exit 1; fi (code=exited, status=0/SUCCESS)
   Main PID: 4199 (minio)
      Tasks: 11
     Memory: 245.8M
        CPU: 16.618s
     CGroup: /system.slice/minio.service
             └─4199 /usr/local/bin/minio server --console-address :9001 http://minio-0{1...4}:9000/mnt/disk{1...2}/minio

May 05 19:32:26 minio-04 minio[4199]: All MinIO sub-systems initialized successfully in 2.583099ms
May 05 19:32:26 minio-04 systemd[1]: Started MinIO.
May 05 19:32:26 minio-04 minio[4199]: MinIO Object Storage Server
May 05 19:32:26 minio-04 minio[4199]: Copyright: 2015-2024 MinIO, Inc.
May 05 19:32:26 minio-04 minio[4199]: License: GNU AGPLv3 <https://www.gnu.org/licenses/agpl-3.0.html>
May 05 19:32:26 minio-04 minio[4199]: Version: RELEASE.2024-05-01T01-11-10Z (go1.21.9 linux/amd64)
May 05 19:32:26 minio-04 minio[4199]: API: http://minio-01:9000
May 05 19:32:26 minio-04 minio[4199]: WebUI: http://192.168.122.34:9001 http://192.168.122.9:9001 http://127.0.0.1:9001
May 05 19:32:26 minio-04 minio[4199]: Docs: https://min.io/docs/minio/linux/index.html
May 05 19:32:26 minio-04 minio[4199]: Status:         8 Online, 0 Offline.

```

## Login To Minio (Web UI)

in Browser type the minio UI URL , can be reached with VMs IP and its Port and Login via the Username And Password Provided in `/etc/default/mini` file.

```txt
192.168.121.31:9000
```

## Resources

- [`qemu/kvm` installation][3]
- [Vagrant Installation][4]
- [VagrantBoxes][5]


[1]: https://min.io/download?license=agpl&platform=linux#/linux
[2]: https://github.com/minio/minio-service/blob/master/linux-systemd/minio.service
[3]: https://docs.fedoraproject.org/en-US/quick-docs/virtualization-getting-started
[4]: https://developer.hashicorp.com/vagrant/install
[5]: https://app.vagrantup.com/boxes/search