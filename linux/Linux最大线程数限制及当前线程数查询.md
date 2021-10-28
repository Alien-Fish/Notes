1、总结系统限制有：
    /proc/sys/kernel/pid_max #查系统支持的最大线程数，一般会很大，相当于理论值

    
    /proc/sys/kernel/threads-max

    
    max_user_process #系统限制某用户下最多可以运行多少进程或线程，使用命令:ulimit -u

    

    注：修改max_user_process的数值，只需要修改/etc/security/limits.conf即可，但是这个参数需要修改/etc/security/limits.d/90-nproc.conf

    附录：
　　附录1：centos 6.*可以修改/etc/security/limits.d/90-nproc.conf
　　我这边是通过修改/etc/security/limits.conf，在最后添加
　　* soft nproc 65535
　　* hard nproc 65535

    查看默认的线程栈大小，单位是字节（Bytes），使用命令:ulimit -s

    
    /proc/sys/vm/max_map_count #硬件内存大小
    


2、Java虚拟机本身限制：
    -Xms  #intial java heap size
    -Xmx  #maximum java heap size
    -Xss  #the stack size for each thread


3、查询当前某程序的线程或进程数
# pstree -p `ps -e | grep java | awk '{print $1}'` | wc -l

上面用的是命令替换，关于命令替换，就是说用``括起来的命令会优先执行，然后以其输出作为其他命令的参数
或
# pstree -p 进程号 | wc -l

# top -H 进程号 | wc -l

上面用的是管道，关于管道：管道符号"|"左边命令的输出作为右边命令的输入


4、查询当前整个系统已用的线程或进程数
pstree -p | wc -l