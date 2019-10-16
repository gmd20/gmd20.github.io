```text
1.
wget http://vault.centos.org/7.7.1908/os/Source/SPackages/iproute-4.11.0-25.el7.src.rpm

2.
rpm -i iproute-4.11.0-25.el7.src.rpm

yum install iptables-devel libmnl-devel linuxdoc-tools psutils tex texlive-preprint

3.
cd /root/rpmbuild/SPECS
[root@localhost SPECS]# ls
iproute.spec

[root@localhost SPECS]# rpmbuild -bp --target=$(uname -m) iproute.spec


4.
cd /root/rpmbuild/BUILD/iproute-4.11.0-0.el7

./configure --help

5.
make

```
