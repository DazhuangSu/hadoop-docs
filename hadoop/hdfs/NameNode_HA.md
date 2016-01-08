#NameNode HA
##Purpose
在hadoop中，namenode始终存在着单点故障问题，一旦namenode宕机，集群服务即终止，HA的目标就是在Active NameNode宕机时可以自动切换到Standby NameNode，切换这个步骤很简单，关键在于namenode存储了HDFS中所有的INode信息（存储于FsImage和EditLog中），如何维持两个namenode之间INode信息的同步是Namenode HA需要解决的最主要问题。
实现NameNode HA的最明显的两个好处是

1. active namenode宕机可以自动切换到standby namenode
2. 既然active namenode关闭不会造成服务空窗期，那么与namenode的相关的打patch升级和更改配置都会变的更加容易

##Hardware resources
1. 两台namenode，硬件配置对等
2. 至少三台JournalNodes，要求奇数如3、5、7等
3. 目前的second namenode的主要工作是merge fsimage和editlog，保证fsimage的更新。在ha中，由standby namenode来完成该工作，因此不再需要second namenode。

##Deployment
参考[NameNode HA部署](http://hadoop.apache.org/docs/r2.5.2/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html#Deployment)

##Code
###Secondary Namenode
目前second namenode的功能主要是提供文件元信息的备份存储，以及定期合并fsimage、editslog两个功能。  
代码实现上second namenode类实现了runnable，通过一个while(true)循环来定期合并fsimage并回传给namenode。
  
![Secondary Namenode](https://raw.githubusercontent.com/StrongSu/research_pics/master/namenodeHA/secondary%20namenode.png)
###Start NameNodeHA
namenode HA需要JournalNodes来对editslog进行存储、同步等功能。  
Active Namenode通过对editslog进行修改来响应用户对hdfs中文件修改请求。两个不同的namenode会对journalnodes持有不同的状态。通过状态来保证同时只有一个namenode对editslog进行修改。  
editslog具有以下几个状态：  

```
private enum State {  
	UNINITIALIZED,  
	BETWEEN_LOG_SEGMENTS,  
	IN_SEGMENT,  
	OPEN_FOR_READING,  
	CLOSED;  
  } 
```
* non-HA  
editslog初始时处于UNINITIALIZED状态，初始化完成后进入IN\_SEGMENT状态，表明namenode现在可以更改editslog。  
在需要合并fsimage和editslog时，namenode首先关闭当前segment，然后开启一个新segment继续记录hdfs变化请求。在关闭旧segment和打开新segement的中间，editslog处于BETWEEN\_LOG\_SEGMENTS状态。  
* HA  
EditsLog在开始namenode刚启动时处于UNINITIALIZED状态，初始化完成后会首先进入OPEN\_FOR\_READING状态，表明namenode会首先进入standby模式。在接收到进入active namenode命令后，editslog先进入CLOSED状态，随后进入BETWEEN\_LOG\_SEGMENTS状态，最后进入IN\_SEGMENT状态，namenode激活完成。

####Start Standby Namenode
如上所述，启动一个Active Namenode会首先进入standyby状态，随后再正式激活。  
启动standby namenode时，会首先将journals置为只读模式，随后创建EditLogTailerTherad线程。该线程会根据配置的时间定时从journalnodes拉取editslog的更新数据，并将拉取到的editslog合并入fsimage。
随后，根据standbyShouldCheckpoint参数确定standby namenode是否需要定期合并fsimage并回传给active namenode，该参数由DFS\_HA\_STANDBY\_CHECKPOINTS\_KEY配置，默认为true。  
若为true，则另新建一个CheckpointerThread线程，执行doCheckpoint()和ThransferFsImage.uploadImageFromStorage()。  

![Start Standby Namenode](https://raw.githubusercontent.com/StrongSu/research_pics/master/namenodeHA/start%20standby%20namenode.png)
####Start Active Namenode
激活namenode的过程很简单。接收到激活namenode的命令后，namenode将sharedJournals置为可写状态，随后开始recovery journalnodes还未完成的segment，同时还需要catchup切换过程中editlog发生的改动，最后将editlog置为可写状态即可。此时，启动active namenode对于文件元信息的操作完成，随后会依次启动leaseManager、NameNodeResourceMonitor、NameNodeEditLogRoller、cacheManager和blockManager。  

![Start Active Namenode](https://raw.githubusercontent.com/StrongSu/research_pics/master/namenodeHA/start%20active%20namenode.png)
