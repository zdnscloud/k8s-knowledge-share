# What is Lambda

AWS Lambda (简称为 Lambda) 是由亚马逊云计算（AWS）提供的`serverless` 服务。
当我们使用Lambda来构建serverless 应用时，虽然我们不需要知道Lambda内部是如何工作的，但是我们了解`functions`是如何被执行的还是非常重要的。


## Lambda Specs

我们从Lambda的技术规范开始。Lambda支持如下运行时。

- Node.js: v10.15 and v8.10
- Java 8
- Python: 3.7, 3.6, and 2.7
- .NET Core: 1.0.1 and 2.1
- Go 1.x
- Ruby 2.5
- Rust

每个函数都在一个64位Amazon Linux AMI中的容器内运行。并且执行环境具有：

- Memory: 128MB - 3008MB, in 64 MB increments
- Ephemeral disk space: 512MB
- Max execution duration: 900 seconds
- Compressed package size: 50MB
- Uncompressed package size: 250MB

我们看到CPU未被提及作为容器规范的一部分。
这是因为我们无法直接控制CPU。 
随着内存的增加，CPU也会增加。

临时磁盘空间以`/tmp`目录的形式提供。
我们只能将此空间用于临时存储，因为后续调用将无法访问此空间。
我们将在下面讨论Lambda函数的无状态特性。

执行持续时间意味着您的Lambda函数最多可以运行900秒或15分钟。
这意味着Lambda不适用于长时间运行的进程。

包大小是指运行函数所需的所有代码。
这包括我们的函数可能导入的任何依赖项（Node.js 中的 `node_modules` 目录）。 
未压缩的上限制为250MB，压缩后限制为50MB。
我们将看看下面的包装过程。

## Lambda Function

最后，这是 Lambda Function (Node.js 版本) 的样子。

```javascript

exports.myHandler = function(event, context, callback) {

  // Do Stuff
  
  callback(Error error, Object result)

}

```

这里，`myHandler` 是我们的Lambda function 的名字。
`event` 对象包含了触发这个Lambda的时间的所有信息。
如果event为一个HTTP请求触发，信息为HTTP请求的所有特征。
`context`对象包含了Lambda function执行的运行时的信息
当函数执行结束后，我们使用`callback`来回结束执行，并返回结果，AWS会把结果返回给HTTP请求

## Packaging Functions

Lambda function需要打包并发送到AWS。
这通常是压缩函数及其所有依赖项并将其上载到S3存储桶的过程。并且让AWS知道您希望在特定event发生时使用此包。
为了简化完成此过程，我们可以使用`Serverless Framework`。

## Execution Model

运行我们的`function`的容器（以及它使用的资源）完全由AWS管理。
它在`event`发生时启动，如果没有使用则关闭。
如果在执行`event`时发出了其他相同`event`，则会启动一个新容器来响应请求。
这意味着如果我们遇到使用量激增，云提供商只需使用我们的函数创建容器的多个实例来为这些请求提供服务。

这有一些有趣的含义。
首先，我们的`function`实际上是`stateless`的。
其次，每个请求（或event）由独立的实例提供`Lambda function`服务。
这意味着我们不需要在代码中处理并发请求。
只要有新请求，AWS就会调出一个容器。
它在这里进行了一些优化。
它会让容器留存几分钟（5到15分钟，具体取决于负载），因此它可以在没有`Cold Start`的情况下响应后续请求。

## Stateless Functions

上述执行模型使 `Lambda function` 实际的无状态。
这意味着每次`Lambda function`由`event`触发时，都会在全新的环境中调用它。
您无权访问上一个事件的执行上下文。

但是，由于上面提到的优化，每个容器实例化仅调用一次实际的`Lambda function`。
回想一下，我们的函数是在容器内运行的。
因此，当首次调用函数时，我们的处理函数中的所有代码都会被执行，并且调用处理函数。
如果容器仍可用于后续请求，则将调用您的函数，而不是其周围的代码。

例如，下面的`createNewDbConnection`方法在每个容器实例化时调用一次，而不是每次调用`Lambda function`时调用。
另一方面，`myHandler`函数在每次调用时都会被调用。

```javascript

var dbConnection = createNewDbConnection();

exports.myHandler = function(event, context, callback) {
  var result = dbConnection.makeQuery();
  callback(null, result);
};

```

容器的缓存效果也适用于我们上面讨论过的`/tmp`目录。
只要容器被缓存，它就可用。

现在我们可以猜测这不是一种使我们的`Lambda function`有状态的非常可靠的方法。
这是因为我们只是不控制调用`Lambda`或缓存其容器的基础进程。

## Pricing

最后，Lambda函数仅对执行函数所花费的时间进行计费。
它从它开始执行到返回或终止的时间计算。
它向上舍入到最接近的100毫秒。

请注意，虽然AWS可能会在容器完成后保留容器和Lambda函数; 但我们不用为这个付费。

Lambda提供了非常慷慨的免费套餐。

Lambda免费套餐包括每月1,000,000次免费请求和每月400,000 GB-seconds的计算时间。
过去，每100万个请求需要0.20美元，每GB-seconds需要0.00001667美元。
GB-seconds基于Lambda函数的内存消耗。
有关详细信息，请查看Lambda定价页面。

Lambda通常是我们基础设施成本中最便宜的部分。

# Why Create Serverless Apps?

```
 _____ _   _       _____ _                     
|_   _| | ( )     /  __ \ |                    
  | | | |_|/ ___  | /  \/ |__   ___  ___ _ __  
  | | | __| / __| | |   | '_ \ / _ \/ _ \ '_ \ 
 _| |_| |_  \__ \ | \__/\ | | |  __/  __/ |_) |
 \___/ \__| |___/  \____/_| |_|\___|\___| .__/ 
                                        | |    
                                        |_|    
```

重要的是要明白为什么值得学习如何创建`serverless`应用程序。
`serverless`应用程序优于传统服务器托管应用程序的原因有几个：

- 低维护
- 低成本
- 易于扩展

到目前为止，最大的好处是我们只需要关心代码而不需要担心其他问题。
低维护是因为没有任何服务器需要管理。
我们无需确保服务器正常运行并保证我们的服务器应用正确的安全更新。
我们只需要处理自己的应用程序代码，没有别的。

运行`serverless`应用程序更便宜的主要原因是您实际上只按每个请求付费。
因此，当我们的服务未被使用时，我们不会被收取任何费用。
让我们快速分析一下我们运行笔记应用的成本。 
我们假设每天有1000个活跃用户每天向我们的API发出20个请求，并在S3上存储大约10MB的文件。
这是对我们成本的非常粗略的计算。

| Service | Rate | Cost |
| :--- | :--- | ---: |
| Cognito | Free[1] | $0.00
| API Gateway | $3.5/M reqs + $0.09/GB transfer | $2.20
| Lambda | Free[2] | $0.00
| DynamoDB | $0.0065/hr 10 write units, $0.0065/hr 50 read units[3] | $2.80
| S3 | $0.023/GB storage, $0.005/K PUT, $0.004/10K GET, $0.0025/M objects[4] | $0.24
| CloudFront | $0.085/GB transfer + $0.01/10K reqs | $0.86
| Route53 | $0.50 per hosted zone + $0.40/M queries | $0.50
| Certificate Manager | Free | $0.00 |
| Total	|   | $6.10 |

* [1] Cognito is free for < 50K MAUs and $0.00550/MAU onwards.
* [2] Lambda is free for < 1M requests and 400000GB-secs of compute.
* [3] DynamoDB gives 25GB of free storage.
* [4] S3 gives 1GB of free transfer.

因此成本大约是每月**$6.1**。
这些都是非常粗略的估计。 
现实世界的使用模式可能会有一些不同。 
但是，这让我们了解如何计算运行`serverless`应用程序的成本。
