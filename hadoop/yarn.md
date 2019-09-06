# YARN #

Hadoop的分布式资源管理框架（Yet Another Resource Negotiator）

## 基本架构 ##

![yarn_architecture](images/yarn_architecture.gif)

YARN的基本思想是分离资源管理和任务调度/监控，分别由ResourceManager和ApplicationMaster负责。

YARN上的应用是一个作业或多个作业的DAG图。

主要包含以下几种角色：

+ ResourceManager（RM）：全局的资源管理器，负责整个系统的资源管理和分配。包含主要两个组件：Scheduler和ApplicationsManager。
  * Scheduler基于应用对资源的需求和Container进行资源分配，YARN支持三种调度器：FIFO Scheduler、Capacity Scheduler和FairScheduler。FIFO Scheduler把应用按提交的顺序排成一个队列，在进行资源分配的时候，先给队列中的第一个应用分配资源，第一个应用需求满足后再给下一个分配，以此类推。调度器通过`yarn.resourcemanager.scheduler.class`配置设置。
  * ApplicationsManager负责处理客户端请求，协商获取应用的第一个Container并启动ApplicationMaster，并且负责ApplicationMaster失败时重启它。

+ NodeManager（NM）：每个节点上的资源和任务管理器，定时向RM汇报本节点上的资源使用情况和各个Container的运行状态，接收并处理来自AM的Container启动/停止等各种请求

+ ApplicationMaster（AM）：用户提交的每个应用程序均包含一个AM，主要功能包含切分数据，与RM调度器协商以获取资源并分配给内部的任务，与NM通信启动/停止任务，监控任务的运行状态

+ Container：是YARN中资源的抽象，封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等。当AM向RM申请资源时，RM为AM返回的资源便是用Container表示的。YARN会为每个任务分配一个Container，且该任务只能使用该Container中描述的资源。

ResourceManager和NodeManager组成了数据计算框架。

### CapacityScheduler ###

CapacityScheduler用于一个集群中运行多个应用的情况，目标是最大化吞吐量和集群利用率。

CapacityScheduler将整个集群的资源分成多个部分，每个组织使用其中的一部分，每个组织有一个专门的队列，每个组织的队列还可以进一步划分成层次结构，从而允许组织内部的不同用户组的使用。
每个队列内部，按照FIFO的方式调度Applications。当某个队列的资源空闲时，可以将它的剩余资源共享给其他队列。

CapacityScheduler中有一个专门的队列用来运行小任务。

### FairScheduler ###

FairScheduler允许应用在一个集群中公平地共享资源。默认情况下FairScheduler的公平调度只基于内存，也可以配置成基于内存和CPU。当集群中只有一个应用时，它独占集群资源。当有新的应用提交时，空闲的资源被新的应用使用，这样最终每个应用就会得到大约相同的资源。可以为不同的应用设置优先级，决定每个应用占用的资源百分比。FairScheduler可以让短的作业在合理的时间内完成，而不必一直等待长作业的完成。

FairScheduler默认让所有的应用都运行，但是也可以通过配置文件限制每个用户以及每个queue运行的应用数量。

FairScheduler支持抢占，即允许调度器杀掉占用超过其应占份额资源队列的Container，这些Container资源便可被分配到应该享有这些份额资源的队列中。设置`yarn.scheduler.fair.preemption=true`来启用抢占功能。

## 工作机制 ##

1. 客户端向ResourceManager提交应用；
2. ResourceManager将应用的资源路径（hdfs://.../.staging和application_id）返回给客户端；
3. 客户端将运行所需资源提交到HDFS上的指定路径；
4. 应用资源提交完毕后，客户端向ResourceManager申请运行ApplicationMaster；
5. ResourceManager将客户端请求初始化成一个任务，放入相应队列；
6. NodeManager领取任务后创建Container并启动ApplicationMaster。
7. Container从HDFS上复制应用资源到本地；
8. ApplicationMaster首先向ResourceManager注册，这样客户端可以直接通过ResourceManager查看应用程序的运行状态，然后ApplicationMaster将为各个任务申请资源；
9. ResourceManager将任务分配给NodeManager，NodeManager领取任务后创建容器；
10. ApplicationMaster向NodeManager发送任务启动脚本，NodeManager启动Container运行任务；
11. 各个任务向ApplicationMaster汇报自己的状态和进度，以让ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务。
12. 应用完成后，ApplicationMaster向ResourceManager申请注销并关闭自己。

## 配置项 ##

YARN同时支持内存和CPU两种资源的调度。

`yarn.nodemanager.resource.memory-mb`：该结点YARN可以使用的内存总量。

`yarn.scheduler.minimum-allocation-mb`：YARN分配内存的最小值。

`yarn.scheduler.maximum-allocation-mb`：YARN分配内存的最大值。

YARN分配给Container的内存和CPU核数必须介于最大值和最小值之间。

`yarn.nodemanager.vmem-check-enabled`：NodeManager是否监控虚拟内存使用。

`yarn.nodemanager.pmem-check-enabled`：NodeManager是否监控物理内存使用。

`yarn.nodemanager.vmem-pmem-ratio`：使用的虚拟内存和物理内存比值的最大值。使用的虚拟内存超过这个值并且`yarn.nodemanager.vmem-check-enabled`为true时，YARN将终止Container。

CPU被划分成虚拟CPU，虚拟CPU是YARN引入的概念，不同节点CPU性能可能不同，可以设置各个节点的虚拟CPU。

`yarn.nodemanager.resource.cpu-vcores`：设置该节点上YARN可使用的虚拟CPU数目，推荐该值与物理CPU核数数目相同。

`yarn.scheduler.minimum-allocation-vcores`：YARN分配虚拟CPU核数最小值。

`yarn.scheduler.maximum-allocation-vcores`：YARN分配虚拟CPU核数最大值。

## 命令 ##

## REST API ##

## 编写YARN应用 ##

需要编写两个组件：客户端和ApplicationMaster。客户端负责向ResourceManager提交ApplicationMaster，并查询应用状态。ApplicationMaster负责向ResourceManager申请资源，与NodeManager通信启动/停止Container，监控任务运行状态，并在失败的时候重新申请资源。JobClient和MRAppMaster是MapReduce使用的两个组件。

### 客户端组件 ###

1. 初始化并启动一个YarnClient：

    ```Java
    YarnClient yarnClient = YarnClient.createYarnClient();
    yarnClient.init(conf);
    yarnClient.start();
    ```

2. 创建应用并获取应用id：

    ```Java
    YarnClientApplication app = yarnClient.createApplication();
    GetNewApplicationResponse appResponse = app.getNewApplicationResponse();
    ```

3. YarnClientApplication的响应中也包含集群的信息，比如集群分配资源的最大/最小值。根据这些信息正确设置用于运行ApplicationMaster的Container。

4. 编写客户端的关键在于设置ApplicationSubmissionContext，ApplicationSubmissionContext中包含了ResourceManager运行ApplicationMaster所需的所有信息。客户端需要在上下文中设置以下信息：
    * id
    * 名称
    * 所属队列
    * 优先级
    * 用户名
    * ContainerLaunchContext，即ApplicationMaster加载运行的Container信息，包含了运行ApplicationMaster需要的所有信息，包括各种文件资源（二进制文件、jar包、文本文件等），环境设置（CLASSPATH等），运行的命令和安全token。

    ```Java
    // 设置应用提交上下文
    ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
    ApplicationId appId = appContext.getApplicationId();
    appContext.setKeepContainersAcrossApplicationAttempts(keepContainers);
    appContext.setApplicationName(appName);

    // 设置ApplicationMaster需要的本地资源
    Map<String, LocalResource> localResources = new HashMap<String, LocalResource>();

    LOG.info("从本地文件系统复制jar包并添加到本地环境");
    // 复制ApplicationMaster的jar包，创建指向目标jar路径的本地资源
    FileSystem fs = FileSystem.get(conf);
    addToLocalResources(fs, appMasterJar, appMasterJarPath, appId.toString(), localResources, null);

    // 根据需要设置log4j属性
    if (!log4jPropFile.isEmpty()) {
        addToLocalResources(fs, log4jPropFile, log4jPath, appId.toString(), localResources, null);
    }

    // 复制shell脚本，保证其在Container上可用
    String hdfsShellScriptLocation = "";
    long hdfsShellScriptLen = 0;
    long hdfsShellScriptTimestamp = 0;
    if (!shellScriptPath.isEmpty()) {
        Path shellSrc = new Path(shellScriptPath);
        String shellPathSuffix = appName + "/" + appId.toString() + "/" + SCRIPT_PATH;
        Path shellDst = new Path(fs.getHomeDirectory(), shellPathSuffix);
        fs.copyFromLocalFile(false, true, shellSrc, shellDst);
        hdfsShellScriptLocation = shellDst.toUri().toString();
        FileStatus shellFileStatus = fs.getFileStatus(shellDst);
        hdfsShellScriptLen = shellFileStatus.getLen();
        hdfsShellScriptTimestamp = shellFileStatus.getModificationTime();
    }

    if (!shellCommand.isEmpty()) {
        addToLocalResources(fs, null, shellCommandPath, appId.toString(), localResources, shellCommand);
    }

    if (shellArgs.length > 0) {
        addToLocalResources(fs, null, shellArgsPath, appId.toString(), localResources, StringUtils.join(shellArgs, " "));
    }

    // 设置ApplicationMaster运行时的环境变量
    LOG.info("设置ApplicationMaster运行时的环境变量");
    Map<String, String> env = new HashMap<String, String>();

    // 将shell脚本位置放到env中，通过evn信息，ApplicationMaster将会为最终运行shell脚本的Container创建正确的本地资源
    env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTLOCATION, hdfsShellScriptLocation);
    env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTTIMESTAMP, Long.toString(hdfsShellScriptTimestamp));
    env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTLEN, Long.toString(hdfsShellScriptLen));

    // 向classpath添加AppMaster.jar位置
    StringBuilder classPathEnv = new StringBuilder(Environment.CLASSPATH.$$())
      .append(ApplicationConstants.CLASS_PATH_SEPARATOR).append("./*");
    for (String c : conf.getStrings(YarnConfiguration.YARN_APPLICATION_CLASSPATH, YarnConfiguration.DEFAULT_YARN_CROSS_PLATFORM_APPLICATION_CLASSPATH))  {
        classPathEnv.append(ApplicationConstants.CLASS_PATH_SEPARATOR); 
        classPathEnv.append(c.trim());
    }
    classPathEnv.append(ApplicationConstants.CLASS_PATH_SEPARATOR)
      .append("./log4j.properties");

    // 设置运行ApplicationMaster的命令
    Vector<CharSequence> vargs = new Vector<CharSequence>(30);

    // 设置Java可执行命令
    LOG.info("设置ApplicationMaster命令");
    vargs.add(Environment.JAVA_HOME.$$() + "/bin/java");
    // 设置Xmx
    vargs.add("-Xmx" + amMemory + "m");
    // 设置类名
    vargs.add(appMasterMainClass);
    // 设置ApplicationMaster参数
    vargs.add("--container_memory " + String.valueOf(containerMemory));
    vargs.add("--container_vcores " + String.valueOf(containerVirtualCores));
    vargs.add("--num_containers " + String.valueOf(numContainers));
    vargs.add("--priority " + String.valueOf(shellCmdPriority));

    for (Map.Entry<String, String> entry : shellEnv.entrySet()) {
        vargs.add("--shell_env " + entry.getKey() + "=" + entry.getValue());
    }
    if (debugFlag) {
        vargs.add("--debug");
    }

    vargs.add("1>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/AppMaster.stdout");
    vargs.add("2>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/AppMaster.stderr");

    // 获取最终结果
    StringBuilder command = new StringBuilder();
    for (CharSequence str : vargs) {
        command.append(str).append(" ");
    }

    LOG.info("ApplicationMaster完全命令 " + command.toString());
    List<String> commands = new ArrayList<String>();
    commands.add(command.toString());

    // 为ApplicationMaster设置Container加载上下文
    ContainerLaunchContext amContainer = ContainerLaunchContext.newInstance(localResources, env, commands, null, null, null);

    // 设置资源类型要求
    // 目前支持内存和虚拟CPU
    Resource capability = Resource.newInstance(amMemory, amVCores);
    appContext.setResource(capability);

    // 服务数据是传给应用的二进制数据
    // 此处不需要
    // amContainer.setServiceData(serviceData);

    // 设置安全令牌
    if (UserGroupInformation.isSecurityEnabled()) {
        Credentials credentials = new Credentials();
        String tokenRenewer = conf.get(YarnConfiguration.RM_PRINCIPAL);
        if (tokenRenewer == null | | tokenRenewer.length() == 0) {
            throw new IOException("Can't get Master Kerberos principal for the RM to use as renewer");
        }

        final Token<?> tokens[] = fs.addDelegationTokens(tokenRenewer, credentials);
        if (tokens != null) {
            for (Token<?> token : tokens) {
                LOG.info("Got dt for " + fs.getUri() + "; " + token);
            }
        }
        DataOutputBuffer dob = new DataOutputBuffer();
        credentials.writeTokenStorageToStream(dob);
        ByteBuffer fsTokens = ByteBuffer.wrap(dob.getData(), 0, dob.getLength());
        amContainer.setTokens(fsTokens);
    }

    appContext.setAMContainerSpec(amContainer);
    ```

5. 完成设置后，客户端就可以提交应用

    ```Java
    // 设置ApplicationMaster优先级
    Priority pri = Priority.newInstance(amPriority);
    appContext.setPriority(pri);

    // 设置应用所属队列
    appContext.setQueue(amQueue);

    // 向ResourceManager提交应用
    // SubmitApplicationResponse submitResp = applicationsManager.submitApplication(appRequest);

    yarnClient.submitApplication(appContext);
    ```

6. 接着，RM接受应用并开始分配满足要求的Container，然后在分配的Container上设置并启动ApplicationMaster

7. 客户端可以通过以下几种方式追踪任务的状态

    * 通过YarnClient的`getApplicationReport()`方法向ResourceManager请求应用情况报告

        ```Java
        // 获取appId指定的应用情况报告
        ApplicationReport report = yarnClient.getApplicationReport(appId);
        ```

        从ResourceManager获取的应用报告包含以下信息：

        + 应用一般信息：ApplicationId，所属队列，提交用户以及开始时间
        + ApplicationMaster详情：ApplicaitonMaster运行的主机，监听客户端请求的rpc端口以及客户端和ApplicationMaster通信的安全令牌
        + 应用追踪信息：如果应用支持进度追踪，可以设置一个追踪URL，通过ApplicationReport的getTrackingUrl()方法获取，客户端查看URL监控进度
        + 应用状态：
    * 如果ApplicationMaster支持的话，客户端可以通过从ApplicationReport获取的`host:rpcport`信息直接向ApplicationMaster查询进度信息，也可以使用ApplicationReport中的追踪链接

8. 客户端可以终止应用。YarnClient的`killApplication()` 方法用于从客户端通过ResourceManager向ApplicationMaster发送终止信号。ApplicationMaster也可能支持rpc的终止调用

    ```Java
    yarnClient.killApplication(appId);
    ```

### ApplicationMaster组件 ##
