```text
cat /etc/shadow
[root@centos7 ming]# perl -e 'print crypt("password123","\$6\$saltsalt\$") . "\n"'
$6$saltsalt$qFmFH.bQmmtXzyBY0s9v7Oicd2z4XSIecDzlB5KiA2/jctKu9YterLp8wwnSq.qc.eoxqOmSuNp2xS0ktL3nh/
[root@centos7 ming]# python -c 'import crypt; print crypt.crypt("password123", "$6$saltsalt123$")'
$6$saltsalt$qFmFH.bQmmtXzyBY0s9v7Oicd2z4XSIecDzlB5KiA2/jctKu9YterLp8wwnSq.qc.eoxqOmSuNp2xS0ktL3nh/
```
