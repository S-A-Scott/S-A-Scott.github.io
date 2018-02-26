---
layout:     post
title:      "MapReduce源码分析（一）Job提交过程"
subtitle:   "Source Code of MR Job Commit"
date:       2017-05-10 12:00:00
author:     "Spencer"
header-img: "img/post-bg-2015.jpg"
tags:
    - MapReduce
    - Hadoop
---

> 说明：
> * 源码版本：hadoop2.6.5
> * 资源调度模式：Local
> * 客户端：本机（Mac）
> * 文件系统：HDFS
> * 使用debug的模式跟踪代码，获取变量值，文本使用{variable value }表示debug模式下获得的变量值。客户端代码是最简单的wordcount

### 一. Job提交过程
* 在客户端代码中，是通过job.waitForCompletion提交的作业的。
```java
public boolean waitForCompletion(boolean verbose
    ) throws IOException, InterruptedException,
            ClassNotFoundException {
        if (state == Job.JobState.DEFINE) {
            // 通过submit方法来提交作业
            submit();
        }
        if (verbose) {
            // 打印作业的进度
            monitorAndPrintJob();
        } else {
            // get the completion poll interval from the client.
            int completionPollIntervalMillis =
                    Job.getCompletionPollInterval(cluster.getConf());
            while (!isComplete()) {
                try {
                    Thread.sleep(completionPollIntervalMillis);
                } catch (InterruptedException ie) {
                }
            }
        }
        return isSuccessful();
    }
```

* 代码是通过submit方法来提交的
```java
 public void submit()
            throws IOException, InterruptedException, ClassNotFoundException {
    ensureState(Job.JobState.DEFINE);
    // 方法中检测是否手动设置了使用旧的API
    // 默认是使用新的API
    setUseNewAPI();
    // connect方法中会根据配置创建出cluster对象
    connect();
    // 根据文件系统和客户端创建出submitter(提交者)对象
    // submitter中持有2个重要的对象
    // 1. jtFS --> 文件系统 {LocalFileSystem}
    // 2. submitClient --> 和jobTracker通信的对象 {LocalJobRunner}
    final JobSubmitter submitter =
            getJobSubmitter(cluster.getFileSystem(), cluster.getClient());
    status = ugi.doAs(new PrivilegedExceptionAction<JobStatus>() {
        public JobStatus run() throws IOException, InterruptedException,
                ClassNotFoundException {
            // 提交作业
            return submitter.submitJobInternal(Job.this, cluster);
        }
    });
    state = Job.JobState.RUNNING;
    LOG.info("The url to track the job: " + getTrackingURL());
}
```
submitClient是实现ClientProtocol接口的对象，ClientProtocol规定了客户端和资源调度器之间的RPC通信协议，关于RPC通信，可以看之前写的另一篇[博客](https://s-a-scott.github.io/2017/01/21/HadoopRPC/)。ClientProtocol的实现类有LocalJobRunner和YARNRunner，分别对应了两种资源调度框架。

---

* 进入submitter.submitJobInternel方法中
```java
JobStatus submitJobInternal(Job job, Cluster cluster)
        throws ClassNotFoundException, InterruptedException, IOException {

    //validate the jobs output specs
    // 检查作业的输出路径，保证输出路径不为空并且 not already there
    checkSpecs(job);

    // getConfiguration会默认读取以下几个文件
    // core-default.xml core-site.xml mapred-default.xml mapred-site.xml
    // yarn-default.xml yarn-site.xml hdfs-default.xml hdfs-site.xml
    Configuration conf = job.getConfiguration();
    addMRFrameworkToDistributedCache(conf);

    // 获得Job提交的部分路径（后面会和JobId拼接成完整路径)
    // {file:/tmp/hadoop-spencer/mapred/staging/root1128794791/.staging}
    Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);
    //configure the command line options correctly on the submitting dfs
    // 获得本地hostname和ip {Spencers-MacBook-Pro.local/192.168.254.55}
    InetAddress ip = InetAddress.getLocalHost();
    if (ip != null) {
        submitHostAddress = ip.getHostAddress();
        submitHostName = ip.getHostName();
        conf.set(MRJobConfig.JOB_SUBMITHOST,submitHostName);
        conf.set(MRJobConfig.JOB_SUBMITHOSTADDR,submitHostAddress);
    }
    // JobClient和JobTracker RPC通信 获得jobId {job_local159099713_0001}
    JobID jobId = submitClient.getNewJobID();
    job.setJobID(jobId);
    // 获得Job提交的路径 {file:/tmp/hadoop-spencer/mapred/staging/root1128794791/.staging/job_local1128794791_0001}
    Path submitJobDir = new Path(jobStagingArea, jobId.toString());
    JobStatus status = null;
    try {
        // 在配置文件中设置job路径，用户名
        conf.set(MRJobConfig.USER_NAME,
                UserGroupInformation.getCurrentUser().getShortUserName());
        conf.set("hadoop.http.filter.initializers",
                "org.apache.hadoop.yarn.server.webproxy.amfilter.AmFilterInitializer");
        conf.set(MRJobConfig.MAPREDUCE_JOB_DIR, submitJobDir.toString());
        LOG.debug("Configuring job " + jobId + " with " + submitJobDir
                + " as the submit dir");
        // get delegation token for the dir
        TokenCache.obtainTokensForNamenodes(job.getCredentials(),
                new Path[] { submitJobDir }, conf);

        populateTokenCache(conf, job.getCredentials());

        // generate a secret to authenticate shuffle transfers
        if (TokenCache.getShuffleSecretKey(job.getCredentials()) == null) {
            KeyGenerator keyGen;
            try {

                int keyLen = CryptoUtils.isShuffleEncrypted(conf)
                        ? conf.getInt(MRJobConfig.MR_ENCRYPTED_INTERMEDIATE_DATA_KEY_SIZE_BITS,
                        MRJobConfig.DEFAULT_MR_ENCRYPTED_INTERMEDIATE_DATA_KEY_SIZE_BITS)
                        : SHUFFLE_KEY_LENGTH;
                keyGen = KeyGenerator.getInstance(SHUFFLE_KEYGEN_ALGORITHM);
                keyGen.init(keyLen);
            } catch (NoSuchAlgorithmException e) {
                throw new IOException("Error generating shuffle secret key", e);
            }
            SecretKey shuffleKey = keyGen.generateKey();
            TokenCache.setShuffleSecretKey(shuffleKey.getEncoded(),
                    job.getCredentials());
        }

        // 将作业运行时需要的资源（作业jar文件，配置文件和计算所得的输入分片）
        // 复制到jobtracker的文件系统中（路径为submitJobDir）
        copyAndConfigureFiles(job, submitJobDir);

        Path submitJobFile = JobSubmissionFiles.getJobConfPath(submitJobDir);

        // Create the splits for the job
        LOG.debug("Creating splits at " + jtFs.makeQualified(submitJobDir));
        // 写入map的切片信息到submitJobDir
        // 会在submitJobFile {file:/tmp/hadoop-spencer/mapred/staging/root1128794791/.staging/job_local1128794791_0001}
        // 路径下生成job.split  .job.split.crc job.splitmetainfo .job.splitmeatainfo.crc 四个文件
        int maps = writeSplits(job, submitJobDir);
        conf.setInt(MRJobConfig.NUM_MAPS, maps);
        LOG.info("number of splits:" + maps);

        // write "queue admins of the queue to which job is being submitted"
        // to job file.
        String queue = conf.get(MRJobConfig.QUEUE_NAME,
                JobConf.DEFAULT_QUEUE_NAME);
        AccessControlList acl = submitClient.getQueueAdmins(queue);
        conf.set(toFullPropertyName(queue,
                QueueACL.ADMINISTER_JOBS.getAclName()), acl.getAclString());

        // removing jobtoken referrals before copying the jobconf to HDFS
        // as the tasks don't need this setting, actually they may break
        // because of it if present as the referral will point to a
        // different job.
        TokenCache.cleanUpTokenReferral(conf);

        if (conf.getBoolean(
                MRJobConfig.JOB_TOKEN_TRACKING_IDS_ENABLED,
                MRJobConfig.DEFAULT_JOB_TOKEN_TRACKING_IDS_ENABLED)) {
            // Add HDFS tracking ids
            ArrayList<String> trackingIds = new ArrayList<String>();
            for (Token<? extends TokenIdentifier> t :
                    job.getCredentials().getAllTokens()) {
                trackingIds.add(t.decodeIdentifier().getTrackingId());
            }
            conf.setStrings(MRJobConfig.JOB_TOKEN_TRACKING_IDS,
                    trackingIds.toArray(new String[trackingIds.size()]));
        }

        // Set reservation info if it exists
        ReservationId reservationId = job.getReservationId();
        if (reservationId != null) {
            conf.set(MRJobConfig.RESERVATION_ID, reservationId.toString());
        }

        // Write job file to submit dir
        // 将job的配置信息写入submitJobFile路径下
        // 会在submitJobFile {file:/tmp/hadoop-spencer/mapred/staging/root1128794791/.staging/job_local1128794791_0001}
        // 路径下生成job.xml和 .job.xml.crc 两个文件
        writeConf(conf, submitJobFile);

        //
        // Now, actually submit the job (using the submit name)
        //
        printTokens(jobId, job.getCredentials());
        // 前面提过submitClient是ClientProtocol实现类的对象，这里submitClient是LocalJobRunner
        //（YARNRunner代码量太大了，不好调试，等以后更全面了解Yarn资源调度细节再分析源码吧QAQ）
        status = submitClient.submitJob(
                jobId, submitJobDir.toString(), job.getCredentials());
        if (status != null) {
            return status;
        } else {
            throw new IOException("Could not launch job");
        }
    } finally {
        if (status == null) {
            LOG.info("Cleaning up the staging area " + submitJobDir);
            if (jtFs != null && submitJobDir != null)
                jtFs.delete(submitJobDir, true);

        }
    }
}
```
有关map是如何切片的可以参考另一篇(博客)[http://www.todo.com]

---
* 继续查看LocalJobRunner是如何submitJob的
```java
public org.apache.hadoop.mapreduce.JobStatus submitJob(
        org.apache.hadoop.mapreduce.JobID jobid, String jobSubmitDir,
        Credentials credentials) throws IOException {
    job = new Job(JobID.downgrade(jobid), jobSubmitDir);
    job.setCredentials(credentials);
    return job.status;
}
```

* 继续查看new Job中做了哪些事情
```java
public Job(JobID jobid, String jobSubmitDir) throws IOException {
    // 获得job提交的路径 systemJobDir {file:/tmp/hadoop-spencer/mapred/staging/root1128794791/.staging/job_local1128794791_0001}
    this.systemJobDir = new Path(jobSubmitDir);
    // 获得job.xml路径 systemJobFile {file:/tmp/hadoop-spencer/mapred/staging/root1128794791/.staging/job_local1128794791_0001/job.xml}
    this.systemJobFile = new Path(systemJobDir, "job.xml");
    // id {job_local1128794791_0001}
    this.id = jobid;
    // 根据job.xml的内容生成JobConf
    JobConf conf = new JobConf(systemJobFile);
    this.localFs = FileSystem.getLocal(conf);
    String user = UserGroupInformation.getCurrentUser().getShortUserName();
    // localJobDir {file:/tmp/hadoop-spencer/mapred/local/localRunner/root/job_local1128794791_0001}
    this.localJobDir = localFs.makeQualified(new Path(
            new Path(conf.getLocalPath(jobDir), user), jobid.toString()));
    // localJobFile {file:/tmp/hadoop-spencer/mapred/local/localRunner/root/job_local1128794791_0001/job_local1128794791_0001.xml}
    this.localJobFile = new Path(this.localJobDir, id + ".xml");

    // Manage the distributed cache.  If there are files to be copied,
    // this will trigger localFile to be re-written again.
    localDistributedCacheManager = new LocalDistributedCacheManager();
    localDistributedCacheManager.setup(conf);

    // Write out configuration file.  Instead of copying it from
    // systemJobFile, we re-write it, since setup(), above, may have
    // updated it.
    // 将配置文件job.xml 拷贝到localJobDir下
    // 将在file:/tmp/hadoop-spencer/mapred/local/localRunner/root/job_local1128794791_0001下生成以下两个文件
    // job_local1128794791_0001.xml 和 .job_local1128794791_0001.xml.crc
    OutputStream out = localFs.create(localJobFile);
    try {
        conf.writeXml(out);
    } finally {
        out.close();
    }
    // 根据.xml文件生成新的JobConf配置对象job，这里名字起为job真的好吗:)
    this.job = new JobConf(localJobFile);

    // Job (the current object) is a Thread, so we wrap its class loader.
    if (localDistributedCacheManager.hasLocalClasspaths()) {
        setContextClassLoader(localDistributedCacheManager.makeClassLoader(
                getContextClassLoader()));
    }

    profile = new JobProfile(job.getUser(), id, systemJobFile.toString(),
            "http://localhost:8080/", job.getJobName());
    status = new JobStatus(id, 0.0f, 0.0f, JobStatus.RUNNING,
            profile.getUser(), profile.getJobName(), profile.getJobFile(),
            profile.getURL().toString());

    // jobs是一个hashMap
    // 目前size为1  {"job_local1128794791_0001" -> "Thread[Thread-17,5,main]"}
    jobs.put(id, this);

    // 开启线程启动job的run方法
    this.start();
}
```

* 接着看下Job的run方法
