```
1. Check vm.swappiness on all your nodes 
   - Set the value to 1 if necessary
$ sudo more /etc/sysctl.conf
...
vm.swappiness = 1
...

2. Show the mount attributes of all volumes
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        30G  1.2G   29G   4% /
devtmpfs        6.9G     0  6.9G   0% /dev
tmpfs           6.9G     0  6.9G   0% /dev/shm
tmpfs           6.9G  8.4M  6.9G   1% /run
tmpfs           6.9G     0  6.9G   0% /sys/fs/cgroup
/dev/sda1       497M   62M  436M  13% /boot
/dev/sdb1        28G   45M   26G   1% /mnt/resource
tmpfs           1.4G     0  1.4G   0% /run/user/1000
/dev/sdc1      1007G   72M  956G   1% /disk1

3. Show the reserve space of any non-root, ext-based volumes 
   - XFS volumes do not use reserve space
   
/dev/sdc1      1007G   72M  956G   1% /disk1
   
4. Disable transparent hugepages

5. List your network interface configuration
$ sudo ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500        
inet 10.5.0.4  netmask 255.255.255.128  broadcast 10.5.0.127        
inet6 fe80::20d:3aff:fea1:595a  prefixlen 64  scopeid 0x20<link>        
ether 00:0d:3a:a1:59:5a  txqueuelen 1000  (Ethernet)        
RX packets 10535  bytes 4814250 (4.5 MiB)        
RX errors 0  dropped 0  overruns 0  frame 0        
TX packets 11827  bytes 1754074 (1.6 MiB)        
TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536        
inet 127.0.0.1  netmask 255.0.0.0        
inet6 ::1  prefixlen 128  scopeid 0x10<host>        
loop  txqueuelen 1  (Local Loopback)        
RX packets 64  bytes 5568 (5.4 KiB)        
RX errors 0  dropped 0  overruns 0  frame 0        
TX packets 64  bytes 5568 (5.4 KiB)        
TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

6. List forward and reverse host lookups using getent or nslookup
7. Show the nscd service is running
8. Show the ntpd service is running


```
