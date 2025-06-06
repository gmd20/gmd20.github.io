
# 挂载cgroup子系统（若未挂载）：
mount -t cgroup2 none /sys/fs/cgroup
# chmod o+wt /sys/fs/cgroup

# 创建控制组：
mkdir /sys/fs/cgroup/mygroup
echo "+cpu" > /sys/fs/cgroup/cgroup.subtree_control
echo "+memory" > /sys/fs/cgroup/cgroup.subtree_control

# 表示每 100ms 最多运行 50ms（即 50%）
echo "50000 100000" > /sys/fs/cgroup/mygroup/cpu.max

# 设置最大内存使用为 512MB
echo $((512 * 1024 * 1024)) > /sys/fs/cgroup/mygroup/memory.max

# 用低优先级模式运行
nice -n 19 your_command

# 添加 进程pid 到 grous 组里面
PID=$(pidof your_command)
echo $PID  > /sys/fs/cgroup/mygroup/cgroup.procs

# 查看当前 cgroup 状态
cat /sys/fs/cgroup/mygroup/cpu.stat
cat /sys/fs/cgroup/mygroup/memory.current

# 测试完 删除 cgroup（必须先清空）
  rmdir /sys/fs/cgroup/mygroup


使用 systemd-run  命令也可达到同样的目的
