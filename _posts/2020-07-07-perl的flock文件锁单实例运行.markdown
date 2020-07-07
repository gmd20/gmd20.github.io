bash有个flock的命令，perl里面好像也有一个 flock函数，就对应c语言的flock函数的吧。

https://perldoc.perl.org/functions/flock.html
官方的例子要把这行改为这样，或者像下面那样直接给出
 use Fcntl qw(:flock SEEK_END LOCK_EX LOCK_UN);

```perl
sub lock {
  my ($fh) = @_;
  my $LOCK_EX = 2;
  flock($fh, $LOCK_EX) or die "Cannot lock file - $!\n";
}
sub unlock {
  my ($fh) = @_;
  my $LOCK_UN = 8;
  flock($fh, $LOCK_UN) or die "Cannot unlock file - $!\n";
}

open(my $config_lock, ">>", "/tmp/config.lock") or die "Can't open config.lock $!";
lock($config_lock);


unlock($config_lock);

```
