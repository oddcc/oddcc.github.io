---
title: gilab-runner源码阅读记录（一）
author: oddcc
type: post
date: 2020-12-23T08:21:09+00:00
url: /2020/12/gilab-runner源码阅读记录（一）/
categories:
  - Golang

---
## 背景

接之前的文章，单纯把整个流程调通了上到生产环境发现还是不满足所有需求。比如：

  1. 不支持指定多个subnet和instance-type，导致的结果就是如果指定了subnet和instance-type，就只能在一个zone中请求spot实例，经常会出现没有相关资源的情况（capacity-not-available）。这时候所有CI请求就被block住了，因为没有新的机器可以运行，只能等已有任务跑完，复用已有的机器
  2. 既然是为了节省成本，当然选比较低配的机型，只要能跑通CI即可。但是有一些项目需要的资源是比较高的，我们选的机型是2C2G的机型，某些项目跑CI就会出现OOM（out of memory）的情况。<!--more-->

对于问题1，我们需要gitlab-runner支持配置多个subnet和instance-type，当拿不到机器的时候，就自动按照配置进行尝试，而不是傻等。

对于问题2，出现了就很难短时间解决，一旦出现此问题，对工作效率影响还是很大的。因为就算此时修改配置，重启runner，也不能保证之后该项目的任务可以分配到高配机器上。

如果不修改代码，只有2个方案：

一个是统一使用高配机型，这显然是不经济的，而且不解决根本问题，谁知道后续还有没有要求更高的项目呢？

另一个是碰到这种项目，就在gitlab后台进行runner的绑定并在项目级停用默认的runner，把这些项目绑定到一个高配的runner上。但是我们的工作流程是参考开源项目的流程，提交代码都是fork项目之后提pr的方式来进行的，针对每一个项目都要提工单，手动分配runner，很麻烦，也遭到了运维的反对，毕竟是额外的工作量。

所以我们需要runner能根据项目名称来修改docker-machine的参数，去请求高配的机型，且之后有该项目的CI任务过来，也要分配到高配的机型上面来运行。

其中docker-machine部分的修改比较容易，但是gitlab-runner的逻辑就比较复杂，这篇记录下runner的启动流程。

## 启动流程

gitlab-runner的启动（gitlab-runner run）流程依赖两个项目

<https://github.com/urfave/cli> 

<https://github.com/ayufan/golang-kardianos-service>

可以稍微熟悉一下用法，其中golang-kardianos-service没啥详细文档，比较反人类，后面会着重提下

入口在项目目录的main.go中，Commands的注册是通过各个包的init()方法完成的，具体之后执行逻辑可根据运行的命令找到具体的包，我们要找的是 gitlab-runner run 命令，所以找到commands包

```go
// ./main.go
package main

import (
   ...
   _ "gitlab.com/gitlab-org/gitlab-runner/cache/azure"
   _ "gitlab.com/gitlab-org/gitlab-runner/cache/gcs"
   _ "gitlab.com/gitlab-org/gitlab-runner/cache/s3"
   _ "gitlab.com/gitlab-org/gitlab-runner/commands"
   _ "gitlab.com/gitlab-org/gitlab-runner/commands/helpers"
   _ "gitlab.com/gitlab-org/gitlab-runner/executors/custom"
   _ "gitlab.com/gitlab-org/gitlab-runner/executors/docker"
   _ "gitlab.com/gitlab-org/gitlab-runner/executors/docker/machine"
   _ "gitlab.com/gitlab-org/gitlab-runner/executors/kubernetes"
   _ "gitlab.com/gitlab-org/gitlab-runner/executors/parallels"
   _ "gitlab.com/gitlab-org/gitlab-runner/executors/shell"
   _ "gitlab.com/gitlab-org/gitlab-runner/executors/ssh"
   _ "gitlab.com/gitlab-org/gitlab-runner/executors/virtualbox"
   _ "gitlab.com/gitlab-org/gitlab-runner/helpers/secrets/resolvers/vault"
   _ "gitlab.com/gitlab-org/gitlab-runner/shells"
)

func main() {
   ...
   app := cli.NewApp()
   app.Name = path.Base(os.Args[0])
   ...
   app.Commands = common.GetCommands() // Commands的注册是在前面那一堆import中，是利用包的初始化机制完成的，逻辑在各个包的init()方法中
   app.CommandNotFound = func(context *cli.Context, command string) {
      logrus.Fatalln("Command", command, "not found.")
   }
   ...

   // 控制权交给了cli框架，之后执行的逻辑取决于Command.Action方法
   if err := app.Run(os.Args); err != nil {
      logrus.Fatal(err)
   }
}
```

commands包的init()方法在 commands/multi.go 文件中，我们可以看到，实际执行的逻辑是 RunCommand.Execute() 方法

其中利用golang-kardianos-service项目，把逻辑封装成了Service，下一步执行逻辑在 RunCommand.Start()方法中

```go
// ./commands/multi.go
package commands

...


// run 命令实际的运行逻辑
func (mr *RunCommand) Execute(_ *cli.Context) {
   svcConfig := &service.Config{
      Name:        mr.ServiceName,
      DisplayName: mr.ServiceName,
      Description: defaultDescription,
      Arguments:   []string{"run"},
      Option: service.KeyValue{
         "RunWait": mr.runWait,
      },
   }

   svc, err := service_helpers.New(mr, svcConfig) // 新建了一个 Service
   if err != nil {
      logrus.WithError(err).
         Fatalln("Service creation failed")
   }

   if mr.Syslog {
      log.SetSystemLogger(logrus.StandardLogger(), svc)
   }

   logrus.AddHook(&mr.sentryLogHook)
   logrus.AddHook(&mr.prometheusLogHook)

   err = svc.Run() // Service.Run()的调用会阻塞，直到Service.Stop()执行完成，
   if err != nil {
      logrus.WithError(err).
         Fatal("Service run failed")
   }
}


...

func init() {
   requestStatusesCollector := network.NewAPIRequestStatusesMap()

   common.RegisterCommand2( // run 命令对应的是 RunCommand
      "run",
      "run multi runner service",
      &RunCommand{
         ServiceName:                     defaultServiceName,
         network:                         network.NewGitLabClientWithRequestStatusesMap(requestStatusesCollector),
         networkRequestStatusesCollector: requestStatusesCollector,
         prometheusLogHook:               prometheus_helper.NewLogHook(),
         failuresCollector:               prometheus_helper.NewFailuresCollector(),
         buildsHelper:                    newBuildsHelper(),
      },
   )
}
```

```go
// ./common/command.go
package common

import (
   "github.com/sirupsen/logrus"
   "github.com/urfave/cli"
   clihelpers "gitlab.com/ayufan/golang-cli-helpers"
)

var commands []cli.Command

type Commander interface {
   Execute(c *cli.Context)
}

func RegisterCommand(command cli.Command) {
   logrus.Debugln("Registering", command.Name, "command...")
   commands = append(commands, command)
}

func RegisterCommand2(name, usage string, data Commander, flags ...cli.Flag) {
   RegisterCommand(cli.Command{
      Name:   name,
      Usage:  usage,
      Action: data.Execute, // Action 对应的是 data.Execute
      Flags:  append(flags, clihelpers.GetFlagsFromStruct(data)...),
   })
}

func GetCommands() []cli.Command {
   return commands
}
```

RunCommand.Start() 另启动了一个 goroutine 来运行实际逻辑，并把控制权转给了 RunCommand.runWait() 方法，实际运行的逻辑在 RunCommand.run() 方法中，至此，启动流程完成

```go
// ./commands/multi.go
// Start is the method implementing `github.com/ayufan/golang-kardianos-service`.`Interface`
// interface. It's responsible for a non-blocking initialization of the process. When it exits,
// the main control flow is passed to runWait() configured as service's RunWait method. Take a look
// into Execute() for details.
func (mr *RunCommand) Start(_ service.Service) error {
    mr.abortBuilds = make(chan os.Signal)
    mr.runSignal = make(chan os.Signal, 1)
    mr.reloadSignal = make(chan os.Signal, 1)
    mr.runFinished = make(chan bool, 1)
    mr.stopSignals = make(chan os.Signal)

    mr.log().Info("Starting multi-runner from ", mr.ConfigFile, "...")
     
    userModeWarning(false)
     
    if len(mr.WorkingDirectory) &gt; 0 {
        err := os.Chdir(mr.WorkingDirectory)
        if err != nil {
            return err
        }
    }
     
    err := mr.loadConfig() // 加载配置
    if err != nil {
        return err
    }
     
    // Start should not block. Do the actual work async.
    go mr.run() // 实际业务逻辑
     
    return nil
}

// run is the main method of RunCommand. It's started asynchronously by services support
// through `Start` method and is responsible for initializing all goroutines handling
// concurrent, multi-runner execution of jobs.
// When mr.stopSignal is broadcasted (after `Stop` is called by services support)
// this method waits for all workers to be terminated and closes the mr.runFinished
// channel, which is the signal that the command was properly terminated (this is the only
// valid, properly terminated exit flow for `gitlab-runner run`).
func (mr *RunCommand) run() {
    mr.setupMetricsAndDebugServer()
    mr.setupSessionServer()

    runners := make(chan *common.RunnerConfig)
    go mr.feedRunners(runners)
     
    signal.Notify(mr.stopSignals, syscall.SIGQUIT, syscall.SIGTERM, os.Interrupt)
    signal.Notify(mr.reloadSignal, syscall.SIGHUP)
     
    startWorker := make(chan int)
    stopWorker := make(chan bool)
    go mr.startWorkers(startWorker, stopWorker, runners)
     
    workerIndex := 0
     
    // Update number of workers and reload configuration.
    // Exits when mr.runSignal receives a signal.
    for mr.stopSignal == nil {
        signaled := mr.updateWorkers(&workerIndex, startWorker, stopWorker)
        if signaled != nil {
            break
        }
     
        signaled = mr.updateConfig()
        if signaled != nil {
            break
        }
    }
     
    // Wait for workers to shutdown
    for mr.currentWorkers &gt; 0 {
        stopWorker &lt;- true
        mr.currentWorkers--
    }
     
    mr.log().Info("All workers stopped. Can exit now")
     
    close(mr.runFinished)
}
```