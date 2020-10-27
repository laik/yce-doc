# 主题: 企业云原生实践

摘要:
在开发运营人员的角度如何上线一个可靠的高可用往往需要配合运维支撑人员的基础资源的设计,半自动化运维?,更新? ...

## 1. Tekton 流水线的持续交付

### 背景
我们通常的开发流程是在本地开发完成应用之后，使用git或svn作为版本管理工具，将本地代码提交到类似版本管理仓库中做源代码持久化存储; 然而来自多个仓库涉及到多个中间件作为底层依赖一起部署到生产环境中时，大部份公司内部通常会有持继集成流程; 早期使用Jenkins工具链完成当下持续集成的工作流程,解决上层devops流程中的必要环节,但使用jenkins由master-slave的架构加之使用jvm运行及其内部使用DSL(groovy)拓展,对于服务器资源使用率,拓展性都不太理想;

### 提出问题
那么如何解决可以快速拓展及拥有强大调度能力框架及架构可以支持千量级并发持续集成的方案? 答案是肯定的，而且不乏竞争者，业内知名的有knavtieBuild/Jenkins/JenkinsX/Spinnaker/ArgoCD/Tekton，其中tekon凭借其众多优良特性在一众竞争者中胜出，成为领域内的事实标准, 我们公司在实践云原生的道路上的第一站,基于Tekton框架的高阶实现;

### 如何解决
...

### 其他(延伸点)
...

## 2. 自定义CRD增强部署管理

### 背景

当前我们在持续集成后的应用采用2种部署在物理机或者vm上(后统称node),docker化部署及传统裸机/vm(supervisor管理)部署在这些node上; 在运维人员的角度考虑如何做到高可用,负载均衡等基础资源架构; 需要借助大量的第三方工具链实现拓展; 其次这种组织在node架构上运行应用程序。对于运行node架构中的应用对于运维人员角度来看,比较难定义资源边界，这会导致资源分配问题。例如在node原本的基础上运行多个应用程序，则可能会出现一个应用程序占用大部分资源的情况，结果可能导致其他应用程序的性能下降。一种解决方案是在不同的node上运行每个应用程序，但是由于资源利用不足而无法扩展，管理的成本很高。

### 提出问题

我们如何在


### 如何解决

...

### 其他(延伸点)

...

## 3. 云原生下的运维管理(管理工具)

### 背景

通常我们一般在开发环境或者测试环境，遇到应用的网络不通，内存泄露，cpu飙升等，需要调试k8s里面的pod的业务容器，并且容器技术的一个最佳实践是构建尽可能精简的容器镜像。但这一实践却会给排查问题带来麻烦：精简后的容器中普遍缺失常用的排障工具，每次进去都需要（apt-get）去下载一些工具，给我们带来了极大的不便利性，甚至把基础镜像打的很大,导致在镜像不好传输等，为了让用户更方便的调试又精简容器的大小，我们需要在容器云推出一个很好的调试工具。

### 提出问题

那么针对上面的背景,我们需要这精简的容器并且又不需要安装其他命令的情况下，怎么去把这块的调试功能做好呢？

### 如何解决

在我们容器云平台里面，我们支持web-shell的功能,点击pod的时候，通过web-shell的方式进入容器里面,让用户针对它自己的业务容器排错，本身原理是非常简单的，如果理解了原理那么就能知道它是如何实现和如何使用的。

在讲这个运维工具之前，我们需要了解一下docker的基本原理，k8s是站在docker之上的，本质运行的还是docker，docker的本质是基于namespace隔离和cgroup资源限制，倘若，我们启动一个进程加入目标容器的namespace中，这样子就和目标容器共享namespace了，那么共享了namespace会怎么样呢？就能看到了目标容器的进程，网络，挂在的目录等。

如果进入了目标容器又能和它共享namespace的同时我们带入一些调试工具，比如网络的排查的image（netshuoot),当目标容器出现了网络的问题，那么就可以针对这个image进去调试网络故障的问题，因为这个工具几乎包含了所有的网络故障排查工具。那么在k8s里面我们是怎么做的呢？

详细的实现流程如下：
- 提前在所有的node安装一个Daemonset，其职责负责attach到目标容器的namespace，并且和后台简历SPDY连接
- 查询目标Pod在哪个节点上
- 找到所在的节点的Pod的节点，并且找到改pod的容器，发送指令给Daemonset的程序，并且attach到目标容器里面
- 用户排查问题

知道了原理和对应的流程之后，就很很容易的使用我们compass容器云来进行排查问题了，在我们的容器云里面找到你要调试的pod，点击当前的Pod，然后进行选择对应的容器进行debug操作,容器云的前端采用[socketjs](https://github.com/sockjs/sockjs-client)和微软的[xterm](https://github.com/xtermjs/xterm.js/)来进行整体的交互，`socketjs`用来
管理底层的websocket连接，而`xterm`用在web上的终端,在容器云上很方便用户的操作。

![image](./img/debug-show.png)
> 右上角有个齿轮的东西点击就可以进入调试

![image](img/terminal.png)
> 可以看到，当我们attach到容器里面的，执行了netstat命令，由于一些极简的镜像什么东西都没有的

![image](img/performance.png)

[netshoot](https://github.com/nicolaka/netshoot)包含了如上的工具，这样子我们想调试目标容器都是这么轻而易举的。

为了制造一些方便开发使用的运维工具的时候，我们又要考虑到业务容器的镜像的极简性(足够小)，于此同时萌发了我们的思考，所以才造就我们开发了这个调试工具,后面，我们需要接入更多的业务团队，由于我们公司的多语言技术栈，比如java,php,golang，python,不同的语言栈，会出现不同的工具，后续，我们针对不同的语言做不同的工具集，方便业务团队使用，比如java应用的内存泄露，cpu飙升，我们会在基础的image装载一些类似[alibab-arthas](https://github.com/alibaba/arthas)等，每个image只做一件事情的原则。


### 其他(延伸点)

用Daemonset的这种方式有点不太优雅，还有优雅的方式当然是 kubernetes的[临时容器](https://kubernetes.io/zh/docs/concepts/workloads/pods/ephemeral-containers/)的方案，但是此方案还是alpha版本，处于比较多bug阶段。这里我讲讲它是怎么比较优雅呢？

详细的原理如下：

- 临时容器其实在原生的Pod扩展了临时容器
- 临时容器是一份是一份CRD(CustomResourceDefinition),用户自定定于

  ```text
  {
    "apiVersion": "v1",
    "kind": "EphemeralContainers",
    "metadata": {
            "name": "nginx-6db489d4b7-tvgsm"
    },
    "ephemeralContainers": [{
        "command": [
            "sh"
        ],
        "image": "busybox:latest",
        "imagePullPolicy": "IfNotPresent",
        "name": "debugger",
        "stdin": true,
        "tty": true,
        "terminationMessagePolicy": "File"
    }]
  }
  ```
- 在原声的Pod字段加入这份JSON，那么原生的Pod就具备了一个调试功能，因为相同的Pod之间的容器本质也是类似我们前面说到共享namespace的，所以我们可以直接attch到这个容器里面做做我们想做的调试功能，只是把相关的image替换成我们各种工具的image。

最后但同样重要的是这是google的人员再推，也是kubernetes后面发展的趋势，所以我们应该跟随者kubernetes的发展。


## 4. 服务网格的实践

### 背景

...

### 提出问题

...

### 如何解决

...

### 其他(延伸点)

...
