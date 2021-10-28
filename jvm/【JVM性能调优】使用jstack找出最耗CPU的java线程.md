jstack可以定位到线程堆栈，根据堆栈信息我们可以定位到具体的代码，所以它在JVM性能调优中很常见。下面我们在找出某个java进程中最耗CPU的线程，并定位堆栈信息，使用到的命令有：ps、top、printf、jstack、grep。

1、使用top -c命令查看占用CPU最高的java进程

#top -c
1


2、查看CPU占用最高的线程

（1）#top -Hp 9179 -d 1 -n 1
（2）#ps -Lfp pid
（3）#ps -mp pid -o THREAD，tid，time
1
2
3
使用第一个输出如下：（time列就是各个java线程耗费的CPU时间）

PS：top命令参数说明：
-p PID：仅监视指定进程的ID，PID是进程号；
-c：显示命令行，而不仅仅是命令名
-h：当系统有多个CPU时，个别CPU的状态信息被隐藏，只显示平均状态值
-d N：显示两次刷新时间的间隔，比如：-d 5，表示两次刷新时间间隔为5秒；

3、占用CPU最高的线程9179换算成16进制到文档中寻找对象应线程

#printf "%x\n" 9224
1


4、打印占用CPU最高的java线程9224的堆栈信息

（1）#jstack 9224 > /root/mss/dump.txt
（2）#jstack 9179 | grep 2408n
1
2
使用第二个，用来输出进程9179的堆栈信息，然后根据线程ID的十六进制值grep，如下：

"PollIntervalRetrySchedulerThread" prio=10 tid=0x00007f950043e000 nid=0x54ee in Object.wait()
1
可以看到CPU消耗在PollIntervalRetrySchedulerThread这个类的Object.wait()这个方法。

有一个方便的定位脚本如下：

###### begin pid为86295的java进程中cpu>0的线程的堆栈######
pid=86295
sfile="/tmp/java.$pid.trace"
tfile="/tmp/java.$pid.trace.tmp"
rm -f $sfile  $tfile
echo "pid $pid"

jstack $pid > $tfile
ps -mp $pid -o THREAD,tid,time|awk '{if ($2>0 && $8 != "-") print $8,$2}'|while read line;
do
        nid=$(echo "$line"|awk '{printf("0x%x",$1)}')
        cpu=$(echo "$line"|awk '{print $2}')
        echo "nid: $nid, cpu: $cpu %">>$sfile
        lines=`grep $nid -A 100 $tfile |grep -n '^$'|head -1|awk -F':' '{print $1}'`
        ((lines=$lines-1))
        if [ "$lines" = "-1" ];
        then
             grep $nid -A 100 $tfile  >>$sfile
             echo '' >>$sfile
        else
             grep $nid -A $lines $tfile  >>$sfile
        fi
done
rm -f $tfile
echo "read msg in $sfile"
###########  end  ############

需要注意的是pid的获取依然需要多ps查找一次，建议pid的获取方式可以改为：

ps -ef|grep 你程序的关键字|grep -v 'grep'|awk '{print $2}'
