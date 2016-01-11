#NameNode HA

&copy;苏大壮  
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
  
![Secondary Namenode](https://raw.githubusercontent.com/DazhuangSu/research_pics/master/namenodeHA/secondary_nn.png)
###NameNodeHA
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
若为true，则另新建一个CheckpointerThread线程，执行doCheckpoint()和TransferFsImage.uploadImageFromStorage()。  

![Start Standby Namenode](https://raw.githubusercontent.com/DazhuangSu/research_pics/master/namenodeHA/startStandbyService.png)
####Start Active Namenode
激活namenode的过程很简单。接收到激活namenode的命令后，namenode将sharedJournals置为可写状态，随后开始recovery journalnodes还未完成的segment，同时还需要catchup切换过程中editlog发生的改动，最后将editlog置为可写状态即可。此时，启动active namenode对于文件元信息的操作完成，随后会依次启动leaseManager、NameNodeResourceMonitor、NameNodeEditLogRoller、cacheManager和blockManager。  

![Start Active Namenode](https://raw.githubusercontent.com/DazhuangSu/research_pics/master/namenodeHA/startActiveService.png)
###User Side
上面简单介绍了secondary namenode、start standby namenode以及start active namenode在server进程收到命令后整个执行流程的最后几步。下面将会介绍执行流程的前几步是如何处理请求并保证后续顺利进行的。
####TransitionToActive(req)
namenode从standby状态到active状态的入口有三个，但是这三个入口最后均从transitionToActive(req)来实现。

* ZKFailoverCtrl  
由ElectorCallbacks调用becomeActive
* FailoverCtrl  
通过HAAdmin可以调用该方法，该方法的主要参数有两个，一个是fromSvc，一个是toSvc。参数的语义是当前的active namenode从fromSvc切换到toSvc。执行该方法需要三个步骤：首先，将fromSvc从active状态切换至standby状态；其次，fence所有发送至fromSvc的请求。最后，将toSvc切换至active状态。
* HAAdmin  
提供手动切换namenode active、standby功能。

####TransitionToStandby(req)
namenode从active状态到standby状态的入口有三个，但是这三个入口最后均从transitionToStandby(req)来实现。

* ZKFailoverCtrl  
由ElectorCallbacks调用becomeStandby
* doCedeActive  
* HAAdmin  
同上  

PS:TransitionTo*可以由RPC call直接调用。

####SetStateInternal
server接受RPC请求后开时处理namenode状态切换，在上述步骤完成之后，通过HAState.SetStateInternal开始切换。

![Start Active Namenode](https://raw.githubusercontent.com/DazhuangSu/research_pics/master/namenodeHA/setStateInternal.png)  
SetStateInternal总体上可分为四个步骤，prepareToExitState、prepareToEnterState、exitState以及enterState。enterState就是前文介绍的start(Active/Standby)Service。  
prepareToExitState做的事情不多，主要是在从standby切换至active时，需要准备退出stanby状态，此时需要暂停当前的checkpoint过程，并取消新的checkpoint。  
prepareToEnterState do nothing.
exitState调用了stop(Active/Standby)Service。  
stopActiveService:  
stopActiveService的过程就是startActiveService的反过程，包括stop leaseManager、interrupt NameNodeResourceMonitor和NameNodeEditLogRoller、cacheManager和blockManager的停止和清理工作。此外还需要讲将editlog置为CLOSED并获取最新的改动写入editlog。  
stopStandbyService:  
该过程比较简单，将startStandbyService时创建的两个线程（standbyCheckpointer和editLogTailer）停止，并将editlog置为CLOSED。  

_For feedback or questions, please contact <dazhuang.su@dianping.com>_