# C# 中如何优雅地处理异步任务与异常

## 异步任务很麻烦

老鸟请轻喷跳过这部分。

如果你没有看懂这一部分，那么这篇文章可能不适合你。但如果你还是很想知道我在说什么，欢迎留言。

我们知道 C# 语言提供了 `async` 和 `await` 两个关键字可以帮助程序员逃离回调地狱。

如果你不认为回调式的代码是地狱的话，那么这篇文章也不适合你。

我确实见过有些程序员认为回调式代码更好理解，所以他们认为一个编程语言有没有提供 `async` / `await`，不是一个重要的特性，而仅仅是一个口味问题。本文稍后会试图劝说他们不要这么想。

现在 `async` 和 `await` 已不再是 C# 的专利了， typescript，javascript 甚至连 rust 这种面向系统编程的语言都已经支持了类似的特性——说你呢 C++，什么时候高傲的 C++ 会才肯支持？

简单解释一下这两个关键字的作用，用需求说话，用需求变动来讲道理，懂的人也可以跳过。

> 现在我们要从网络上下载一段文字，例如是《最终用户使用协议》（简称 `EULA`）之类的文字吧，**并且要在用户界面上显示出来**，然后让用户选择接受或者不接受。

我们的用户很可能不会愿意见到这样的软件：下载完成之前，整个用户界面都处于卡住不响应的状态。所以，我们需要把下载的工作，放到和用户界面不是同一个步调的后台线程里面执行。这就是土法解释“异步”。

如果没有 `async` / `await` 这样语言内置的异步机制，也不使用类似 `Future` 和 `Task` 之类的东西（实际上他们都是同一个东西的不同名称而已），那么为了保证下载过程不会阻塞用户界面的线程，我们必须把这个下载的工作安排在后台线程来执行。我们很可能需要这样来设计代码，来描述这个过程（代码是 C#，但 Java 也差不多）：

```C#
class Sample
{
    public void DisplayEulaText(Func<string> downloadEulaText)
    {
        // 执行一些准备显示界面的工作
        // 开始启动后台下载
        this.StartDownload(
            downloadEulaText, // 调用这个方法来下载
            OnDownloadEulaTextCompleted // 下载完成后调用这个回调函数
        );
    }

    private void StartDownload(Func<string> dowanload)
    {
        // 将工作安排到后台线程
    }

    private void OnDownloadEulaTextCompleted(string eulaText)
    {
        // 省略了显示文字代码

        if (accepted)
            this.OnAccept(); // 如果接受了协议
        else
            this.OnDecline(); // 如果不接受协议
    }
}

```

这样我们才可以避免，在 `DisplayEulaText` 方法中阻塞了用户界面线程。

我们可以看出，在这样的回调风格的代码里面，为了等待需要很长时间才能完成的任务，我们必须把“发起耗时的任务” 和 “取得耗时任务的结果” 这两个事情，分开在两个不同的方法里完成，形成了不必要的耦合。

所谓不必要的耦合，就是当业务逻辑有变化的时候，会影响到过多的不同处理逻辑的代码。

在这个具体的例子里来说就是，如果我下载的《最终用户协议》不仅仅是纯文字，而是有可能有多媒体的内容，那么很可能仅仅使用 `string` 就不太适合用来表示多媒体的内容了，例如需要改成或者增加支持 `byte[]` 字节流，那么我们需要如何改动代码来适应这个变化呢？

首先，`DisplayEulaText(Func<string> downloadEulaText)` 就需要改成（或者增加一个）`DisplayEulaMultimedia(Func<byte[]> downloadEulaData)`，

然后 `StartDownload` 和 `OnDownloadEulaTextCompleted` 显然也是要跟随改动的。

这显然不仅仅是口味问题，而是工作量的问题了。而且这些工作量看起来很无趣，不光是我们程序员会想，就连普通用户也会这么想的：

显示文字也是显示，显示多媒体也是显示，下载文字和下载多媒体不都是下载吗，**反正最后都是要接受协议的**，为什么就要写那么多功能相似但完全不同的代码？是享受过程的意思吗？

真相了兄弟们，为何你的头发日渐稀少，为何妹纸觉得你机械无趣，原来这就是原因啊！

那么我们该如何~~挽救发量~~解决问题呢？我们要使用 C# 里面的 `Task` 类。因为 `async` 和 `await` 这样的语法糖隐藏了太多 **C# 编译器帮我们实现的细节**，这里要先展示一下，如果不借助编译器生成的代码，我们会如何写代码：


```C#
public void DisplayEulaText()
{
    var isAcceptedTask = this.DownloadTextAsync();
        .ContinueWith(ShowContentAsync)
        .ContinueWith(HandleAcceptOrDeclineAsync);

    // 注意这里，用户选择了接受或者不接受协议，可以直接在一个方法里处理了。
    if (isAcceptedTask.Result)
        this.OnAccept();
    else
        this.OnDecline();
}

private Task<string> DownloadTextAsync() { /*省略内容*/ }

private Task ShowContentAsync(Task<string> eulaText) { /*省略内容*/ }

private Task<bool> HandleAcceptOrDeclineAsync() { /*省略内容*/ }

```

我们可以注意到，现在的写法，可以在一个地方完整观察到中间的下载过程代码了。这意味着什么呢？我们的业务逻辑在一个方法里面完全定义好了，其他的异步过程调用，都只是其他辅助功能，是可以被替换掉的。例如我们可以把获取 EULA 变成不是从网络下载，而是在硬盘上读取，大家可以试想一下如果真的改成这样，不使用 `Task` 会有多麻烦。 

总之，为了分离发起异步任务，和获取异步任务结果这两件事情，这些不同的异步过程被 `Task` 类中的 `ContinueWith` 方法串联了起来，而异步任务的结果，可以用 `Task<T>.Result` 来获取，避免了像之前 `OnDownloadEulaTextCompleted` 的尴尬。

尴尬之处在于，一个让用户选择接受或者拒绝 EULA 的代码，为什么要管下载的内容是什么以及怎么下载呢？而且更要命的是，虽然我们知道在逻辑上，下载不同的内容之后都是执行相同的后续操作，但我们还是需要把这些相同的后续操作的代码 **在不同的地方粘贴相同的东西**。

如果你认为这都可以接受，为什么头发稀少不可以接受？


## 有异常的异步任务更麻烦

现在我们来考虑更加复杂的现实情况，假设在下载 EULA 文字或者内容的过程中，超时了应该怎么办？

或者，更加广泛的问题，异步任务耗费的时间超出设计范畴，但我们 **不想让软件崩溃**，我们想处理这个情况，应该怎么写代码呢？

### 第一层：使用 TimeoutException

在第一层的朋友马上就有答案了，我们可以抛出超时异常啊。

于是， `DisplayEulaText` 的异步版本我们可以这样写（现在我们使用 `await` 关键字来简化代码）。


```C#
public async Task DisplayEulaTextAsync()
{
    var downloadTask = this.DownloadTextAsync(); // 启动下载任务
    var countDownTask = Task.Delay(TimeSpan.FromSeconds(10)); // 注意：这里擅自设置了10秒超时

    // 看看超时和下载完成哪一个先来
    await Task.WhenAny(downloadTask, countDownTask);

    // 如果超时了，就抛出异常
    if (countDownTask.IsCompleted)
        throw new TimeoutException("下载超时");

    var eulaText = await downloadTask;
    await this.ShowContentAsync(eulaText);

    var isAccepted = await this.HandleAcceptOrDeclineAsync();

    // 省略后续的代码
}
```

然后，在调用到 `DisplayEulaTextAsync` 的地方，用 `try` 和 `catch` 包一下就完事儿了。

```C#
try
{
    this.DisplayEulaTextAsync()
}
catch (TimeoutException timeoutException)
{
    // 处理超时异常
}
```


### 第二层：使用 CancellationToken 和 OperationCanceledException

第二层的朋友可能就说了，这10秒的超时谁定的？需求没写这么具体啊，万一我的用户就愿意等我一小时两小时呢？

所以决定超时与否的判断权利，肯定不在 `DisplayEulaText`，甚至具体的超时时长也是不确定的。这个判断权利只能由外部决定，所以我们可以使用一个 `CancellationToken` 类型的参数，来传达这种外部依赖（当然外部代码也要响应这里的更改）。

再者，更精确地说，`DisplayEulaText` 代码里要控制的是下载超时，而不是显示 EULA 超时啊，调用 `DisplayEulaText` 的地方可能根本就没想到会有超时异常，或者就算有也没打算在那里捕获，万一有很多地方都要调用 `DisplayEulaText` 呢？那岂不是到处都要包上？因此，不能指望这里发生的异常会被更外层的代码接住。

综上，代码应该这么写：

```C#
using System.Threading;
using System.Threading.Tasks;

public async Task DisplayEulaTextAsync(CancellationToken token)
{
    try
    {
        // 这里使用 Task.Run 并传入 CancellationToken 的方式来启动异步任务
        var eulaText = await Task.Run(this.DownloadTextAsync, token);

        await this.ShowEulaTextAsync(eulaText);

        var isAccepted = await this.HandleAcceptOrDeclineAsync();

        // 省略后续的代码
    }
    catch (OperationCanceledException operationCancelledException)
    {
        // 处理下载过程中被取消的异常
    }
}

```

### 第三层：返回错误而不是抛出异常

如果第二层的朋友确实如上面的例子里面那样给别人做示范，就是真的写着 `// 省略后续的代码` 以及 `// 处理下载过程中被取消的异常`，那么在第三层的朋友肯定就会吐槽：

“那你倒是写完整啊！”

“我就要看看你怎么把这 B 装完！”

第三层的朋友肯定都猜到了，异步任务被取消，这不是执行这个异步任务的代码所能处理的情况，所以实际上，那些 `// 省略后续的代码` 以及 `// 处理下载过程中被取消的异常` 是怎么都不可能写得出来的，或者说，怎么写都是不够合理的，因为实现这些处理措施的责任，不应该落在写这部分代码的程序员的头上。

讲道理的话，这种不能处理的锅，就应该用异常机制扔给调用者，让更高层次的代码来处理。这是各大编程语言中，设计异常机制的初心，把锅以一种优雅的方式甩出去。

但是实际上不同语言对于异常机制有不同的设计和使用方式，造成了不一样的问题。

#### 让人心惊胆战的 runtime exception

在 Java 里面有 `checked exception` 和 `runtime exception` 两种不同的异常。其中 `runtime exception` （运行时异常） 大致上和 C# 里面的异常机制设计相似。

这种机制允许，在调用有可能抛出运行时异常的方法的时候，调用者不需要明确地捕获异常。这种设计有可能导致的问题就是，万一存在一条能产生异常且又没有被捕获的调用链，那么你的这个应用就会崩溃。而且，看看上面的代码，如果你不看文档，你根本就不知道 `Task.Run(Func<Task<T>>, CancellationToken)` 这个方法，和 `OperationCanceledException` 之间到底有什么关系，这对于阅读和理解代码来说也是有点不够直观的。

目前从实践来看，解决运行时异常的有效方法之一就是多做测试，多写测试用例；之二，付费使用一些高级的代码分析工具，找出可能导致崩溃，但没有被捕获的产生异常的调用链。

看看你的发量吧，是否能够支撑你多写测试用例；或者牺牲掉你准备植发的钱，用来支付代码分析工具的费用。但这也只是降低了产生未被捕获的异常的概率。

#### 让人发量变少的 checked exception

`checked exception` 要求调用者明确地在 `catch` 子句中捕获这类异常，否则编译不会通过；而后者（运行时异常）则不强制要求调用者捕获这个异常。

具体来说，如果在Java 里面也有 `OperationCanceledException` 并且是一个 `checked exception`，那么在任何调用 `DisplayEulaText` 的地方，都至少要包含这样的写法，否则编译不通过：

```Java
try
{
    //...
    this.DisplayEulaText();
    //...
}
// 必须要处理这个异常
catch (OperationCanceledException oce)
{
    //...
}

```
或者，任何调用 `DisplayEulaText` 的方法也声明为

```Java

public void MethodMayCallDisplayEulaText(Sample s) throws OperationCanceledException
{
    s.DisplayEulaText();
    // ...
}

```
并且，由于 `DisplayEulaText` 的调用者，受到传染，变成也有可能抛出 `OperationCanceledException` 那么它也必须这样声明，或者在代码中包含捕获这种异常的 `catch` 语句，否则不会编译通过。

Java 语言的这个机制，确实保证了外层代码要么声明会上抛，要么会捕获所声明的异常，不会因为遗漏而导致整个应用崩溃。但是Java 语言的这个设计问题也很大。

如果 `DisplayEulaText` 在运行时的调用层次很深，距离能够处理的代码之间相隔了很多次调用，那么为了能让这个异常能正确地抛到处理的代码的调用处，中间的调用链肯定到处都充斥这样的毫无意义的代码：

```Java
catch (OperationCanceledException e)
{
    throw e;
}
```
又或者在调用链条上一堆其他很难看得出关联的函数定义中都声明 `throws OperationCanceledException`。

虽然我们知道 Java 的 IDE 工具很成熟，在需求变动而改代码的时候也可以很方便地改一大片相关代码。

但如果到处都是重复的代码都可以接受，为什么头发稀少不可以接受？


#### 借助多值返回同时传递错误和正确结果

以下是 Go 语言的代码

```go
func (s *Scanner) Scan() (token []byte, error)
```

> and then the example user code might be (depending on how the token is retrieved),

```go
scanner := bufio.NewScanner(input)
for {
    token, err := scanner.Scan()
    if err != nil {
        return err // or maybe break
    }
    // process token
}
```

这不是我写的，这是 golang 的发明者的代码（[原文](https://blog.golang.org/errors-are-values)）。

我在很多地方抨击过 golang 是一门设计得很差的语言，甚至比 Java 更差。Java 是好心但没有能力办好事情，太老了负担太多改不动。而 golang 则是存心要把事情办砸，还美其名说简洁才是我的风格。golang 处理其他语言中的异常的方式，使用的居然就是错误码，这简直就是大倒退。

由于 golang 太烂，我学不会，我拒绝写任何 golang 代码，这不是一个口味问题。

C# 7 中同样有多值返回特性，可以同时返回结果和错误的方式，但估计不会有人这么写，因为一看就很沙雕。

放在 `ShowEulaText` 的例子中，实际上相当于在 C# 中这样实现：

```c#
public async Task<(Result, Exception)> DisplayEulaTextAsync(CancellationToken token)
{
    // 为了方便讨论，这里假设 C# 像 golang 一样设计了 Task.Run 方法不是抛出异常而是返回异常
    (var downloadTextResult, var downloadTextErr) = await Task.Run(this.DownloadText, token);

    // 这里需要先查一遍 DownloadText 是否有错误，分别是什么意思，怎么处理
    if (downloadTextErr != null)
    {
        // 要想办法做合适的错误码转换
        if (downloadTextErr is OperatonCanceledException oce)
            return (default(Result), ...);

        if (dowanloadTextErr is SomeOtherException soe)
            return (default(Result), ...)

        // 然后这里还要想办法为对应不上的底层错误码，选一个合适的错误码返回
        return (default(Result), new Exception("Unkown Error"));
    }
    this.ShowEulaText(downloadText);
    // ...
}

```

第四层的同学已经忍不住要跳出来了，并迅速地指出以下的问题：

第一，返回 `Exception` 一定会被调用者正确处理吗？如果调用者就是不处理，或者忘了处理怎么办？如果靠人肉检查来解决这些问题，那为什么要有编译器？这就是为什么有人会发量越来越少的原因吧。

第二，返回错误依然没有解决分锅甩锅的问题，如果*处理错误的代码*和 *返回错误的代码* 之间，存在很长的调用链，那么这些调用链之间也会充斥着各种毫无作用的 `return err`。

甚至比异常机制更加不如的是，各层之间代码的作者，还要想办法在合适的时机把正确的错误码返回出去。最终形成的效果就是，调用链上的每一层，都要处理其调用链下游可能经过的任何方法、所产生的任何错误码，要么就是忽略掉。 

也许有人会争辩说，哪有那么多的错误，你这个想法过于极端了。

有些人把编程写代码，当成一个吃饭和排泄一样的新陈代谢的过程。这种人估计是打算一辈子编程的，我敬他是个狼灭。

但我的想法是，问题只需要解决一次就够了，其他都是机器来完成的重复的工作。

### 第四层：Rust 的启发

我们先总结我们到底要解决怎么样的问题。

我们实际上需要一种编程风格，只要遵守这种编程风格，可以确保底层调用的异常必定可以抛到上层的对应的处理代码，又希望在抛的过程中不相关的代码不用受到底层代码到底是哪种异常的干扰。

C# 本身的异常机制不能在编译器的层面确保，所有的异常必然被捕获。

Java 有 `chekced exception` 用编译器来强制所有人必须关注来自底层的异常，但是这个做法副作用太大了，我们的发量经不起这样的折腾，只有大厂才玩得起。

Golang 是自说自话地宣称我们的解决办法很简洁，然后把问题推给程序员。当然，大厂用 Golang 也是可以通过测试工作量来解决这个问题。但是 golang 的思路里面有一部分是对的，我们可以异常当作一种返回值来对待；只不过 golang 的具体办法让人觉得它这个结论是误打误撞的，这是 golang 的发明者坚持认为不需要泛型的恶果。

实际上，如果要一并返回正常结果和异常结果，那么正常和异常类型之间的逻辑关系应该是“或”，而不是“与”。Golang 返回的结果，相当于是说，

> 我给你返回了正常结果以及异常结果，你可以自（随）己（意）挑。

但实际上应该是

> 我给你返回了一个要么是正常，要么是异常的结果，你自己分辨吧。

在 C 系家族的语言里，也即 C、C++、C#、Java 也包括 golang 里面，实际上都有限度地支持“与”和“或”这两种类型并联结合关系，但都把风险留给程序员自己处理。

不讲道理的话，C 你总是可以返回一个无类型指针，让调用者自己猜；

C++ 是个变态，你可以用模板玩出任何魔法出来，但我们玩不起；

C# 和 Java 你确实也可以返回一个 object 让调用者自己分辨出类型，但编译器不会给你任何支持；

golang 实际上也可以返回空 interface；

在这一点上，我要特别介绍 Rust 语言的 `enum` 的设计：

```Rust
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

所以 Rust 版本的 `DisplayEulaText` 和 `DownloadText` 就可以大致上这么写

```Rust
// 返回的结果是 str 或 MyErr 
fn download_text() -> Result<str, MyErr> {
    //...
}

fn display_eula_text(&self) -> () {
    let downloadTextResult = download_text();

    // 编译器保证必须两种情况都处理
    match downloadTextResult {
        Ok(text_str) => self.show_eula_text(text_str),
        Err(my_err) => handle_err(err)
    }
}

```

在函数式编程的语言里，通常都有这种类型（实际上是 `monad`），叫做 `Either`。如果用 C# 来描述这两种类型大致上的功能，可以用以下代码描述：

```C#
// 表示可能是左边类型或者右边类型中的一种
public readonly struct Either<L, R>
{
    public X Match(
        Func<L, X> matchLeft,
        Func<R, X> matchRight,
    );
} 

// 为了方便可以添加这样的扩展
public static class EitherExtensions
{
    public static Either<X, R> MatchLeft<L, R, X>(in this Either<L, R> either, Func<L, X> match)
        => throw new NotImplementedException(); // 省略了不难想到的实现代码

    public static Either<L, X> MatchRight<L, R, X>(in this Either<L, R> either, Func<R, X> match)
        => throw new NotImplementedException(); // 省略了不难想到的实现代码
}

```

这个类怎么用呢，为了方便讨论，我们先在非异步的情况下，这样定义 `DownloadEulaText` 的签名：

```C#
Either<string, Exception> DownloadEulaText();
```

于是，在 `DisplayEulaText` 的实现里面，我们就可以这样写：

```C#
public Either<bool, Exception> DisplayEulaText()
{
    var strOrErr = this.DownloadEulaText();
    return strOrErr.MatchLeft(MatchTextStr);
    
    bool MatchTextStr(string eulaText)
    {
        // 直接处理 eulaText 正常结果
        this.ShowEulaText(eulaText);

        //
        return true;
    }
}
```
上面的代码肯定不是一个完整的解决方案，但在这个示例里面，我们看到了 C# 里面的一种可能性：在不先对结果进行分辨的情况下，如何把薛定谔的结果（正常还是异常的混合状态）层层传递出去。具体来说，就是依靠 `Either<L, R>` 这个类型。

写到这里，文章已经非常长了，只好用一份代码摘录来抛砖引玉，这个思路如何实现一个优雅的可取消等待的异步锁。

```C#
using System.Threading;
using System.Threading.Tasks;

public sealed class AsyncLock
{
    private readonly SemaphoreSlim semaphore_;

    public async Task<MayBeCancelled<X>> EnterAsync<X>(
            CancellationToken                token,
            Func<CancellationToken, Task<X>> critical)
    {
        try
        {
            var acquireJob = CancellableJob.Launch(
                token,
                this.semaphore_.WaitAsync
            );
            if (await acquireJob.HasBeenCancelledAsync())
                return token.CancelledResult();

            return await CancellableJob.Launch(token, critical);
        }
        finally
        {
            if (this.semaphore_.CurrentCount == 0)
                this.semaphore_.Release();
        }
    }
}

```

其中的 `MayBeCancelled<X>` 实际上就是一个类似 `Either` 的结构。