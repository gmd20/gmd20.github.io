```perl
#!/usr/bin/perl

use Net::SSH::Perl;

my %params;
$params{"debug"} = true;

my $ssh = Net::SSH::Perl->new("192.168.1.1", %params);
$ssh->login("admin", "admin");
my($stdout, $stderr, $exit) = $ssh->cmd("df -h");

print $stdout;

```

```python
import paramiko

paramiko.util.log_to_file("paramiko.log")

client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

client.connect(hostname="192.168.1.166",  port=22, username="admin", password='admin')
stdin, stdout, stderr = client.exec_command('df -h')

print(stdout.read().decode('utf-8'))

client.close()

```
