---
layout: post
title: await async 模式造成的死锁
subtitle: 使用环境winform
categories: .net
tags: .net c#
---

##  场景

在使用await async关键字时，程序会莫名其妙的卡住，就算声明为async的方法中的Task任务执行完成后，外部方法的await调用还是阻塞着。await and async模式使用不当很容易造成程序的死锁。

## 理解

### await之前和之后的线程

await关键字之前和之后的代码有可能是在不同的线程上执行的：

```c#
// await之前的代码可能在线程A上执行
string result = await GetStringAsync();
// 待GetStringAsync()执行完毕后，await之后的代码有可能是在线程A上执行，也有可能是在一个新的线程B上执行
```

至于await关键字之前和之后的代码是否是在相同的线程上执行，这是由.NET根据当前状态（默认情况下，是线程池当前的状态）决定的。

### 示例

这些代码会造成死锁

#### UI Example

``` csharp
// My "library" method.
public static async Task<JObject> GetJsonAsync(Uri uri)
{
  using (var client = new HttpClient())
  {
    var jsonString = await client.GetStringAsync(uri);
    return JObject.Parse(jsonString);
  }
}

// My "top-level" method.
public void Button1_Click(...)
{
  var jsonTask = GetJsonAsync(...);
  textBox1.Text = jsonTask.Result;
}
```

#### ASP.NET Example

```c#
// My "library" method.
public static async Task<JObject> GetJsonAsync(Uri uri)
{
  using (var client = new HttpClient())
  {
    var jsonString = await client.GetStringAsync(uri);
    return JObject.Parse(jsonString);
  }
}

// My "top-level" method.
public class MyController : ApiController
{
  public string Get()
  {
    var jsonTask = GetJsonAsync(...);
    return jsonTask.Result.ToString();
  }
}
```



### 什么导致了死锁

**在你await一个Task对象后，当方法要继续执行await关键字之后的代码时，线程需要在一个context中方能继续执行。**

**另外一个很重要的知识点：一个UI context并不会和一个特有的线程绑定（ASP.NET request context也是），但是一个context在任意时刻都只允许一个线程进入。这个有趣的知识点并没有在AFAIK上的任何文档中被记录，但是它在MSDN上[介绍SynchronizationContext的文章中被提到了](https://docs.microsoft.com/en-us/archive/msdn-magazine/2011/february/msdn-magazine-parallel-computing-it-s-all-about-the-synchronizationcontext)。**

所以这就是发生的事情，从top-level method开始（对于上面UI的例子是Button1_Click方法，对于ASP.NET的例子是MyController的Get方法）

1. The top-level method calls GetJsonAsync (within the UI/ASP.NET context). 

2. GetJsonAsync starts the REST request by calling HttpClient.GetStringAsync (still within the context). 

3. GetStringAsync returns an uncompleted Task, indicating the REST request is not complete.

4. GetJsonAsync awaits the Task returned by GetStringAsync. The context is captured and will be used to continue running the GetJsonAsync method later. GetJsonAsync returns an uncompleted Task, indicating that the GetJsonAsync method is not complete. 

   （**注意现在这个context是被top-level method的线程占有的，当GetStringAsync方法执行完毕返回后，继续执行GetJsonAsync方法的线程，需要获得该context来继续执行await关键字之后的代码，这也是造成本例中代码会死锁的原因**

5. The top-level method synchronously blocks on the Task returned by GetJsonAsync. This blocks the context thread.（因为代码访问了jsonTask.Result属性，而访问这个属性现在会被阻塞，因为GetJsonAsync方法还未执行完），这会导致context线程（也就是执行top-level method的线程）被阻塞。

6. … Eventually, the REST request will complete. This completes the Task that was returned by GetStringAsync. 

7. The continuation for GetJsonAsync is now ready to run, and it waits for the context to be available so it can execute in the context. 

8. Deadlock. The top-level method is blocking the context thread, waiting for GetJsonAsync to complete, and GetJsonAsync is waiting for the context to be free so it can complete. **死锁发生了。**top-level method中现在占有context的线程正在被阻塞来等待GetJsonAsync方法执行完成，这时top-level method的线程会一直占有context，然而GetJsonAsync方法也在等待context被释放才能执行完成。也就是说top-level method中的线程在等待GetJsonAsync方法执行完成，所以被阻塞，GetJsonAsync方法又在等待top-level method中的线程释放context也被阻塞，两个阻塞相互等待，相互死锁。

For the UI example, the “context” is the UI context; for the ASP.NET example, the “context” is the ASP.NET request context. This type of deadlock can be caused for either “context”.



## 死锁怎么解决

这里有三个最佳实践来避免死锁，其中前面两个在**[文章](https://blog.stephencleary.com/2012/02/async-and-await.html)**也介绍过：

第一种方法，设置`ConfigureAwait(false)`后，会导致await关键字之后的代码在一个新的线程上运行，如果是在Winform程序中，await关键字之后的代码设置了控件的属性，会产生Winform程序的线程安全异常，所以方法一不适用于.NET中的UI项目（诸如Winform、WPF等项目），同理第三种方法也不适合.NET中的UI项目。

其实最好的方法应该是第二种，将await and async模式在调用方法中贯彻到底，由.NET自己来管理持有"context"的线程，就不会出现本文所述的死锁情况，此外一直保持await and async模式还有个好处，所有await and async模式中的线程都是由.NET来自动创建和销毁的，这样可以保证线程池中的线程得到最大的重用，避免了由于人为阻塞线程（注意当用await关键字等待一个未完成的Task对象时，执行await代码的线程实际上会立即返回，正如本文开始时所述，待await的Task对象完成后可能由新的线程来继续执行await关键字之后的代码，但是这个过程中并没有线程被**阻塞**，所以所有的线程都可以去做它们该做的事情，并不会进行无谓的等待），导致线程池需要创建新的线程，从而产生额外的性能开销。

那么我们来想想为什么第二种方法可以适用于.NET中的UI项目（诸如Winform、WPF等项目），.NET之所以让本文所述的"context"只允许被一个线程占有，是为了保证await关键字后面的代码是被**正确的线程**执行的，就拿Winform项目来举例，设想下如果await关键字后面的代码设置了控件的属性，而该代码是在一个非UI线程（非主线程）来执行的，就会产生线程安全异常（正如上面第一种方法所述）。现在有了"context"这个机制后，由于"context"被UI线程（主线程）一直持有，所以只有UI线程（主线程）才能执行await关键字后面的代码，这样就避免了.NET中UI项目的线程安全异常问题，使得只有UI线程（主线程）才能执行await关键字后面的代码，但是由于本例中的UI线程（主线程）被阻塞了，所以才导致await关键字后面的代码无法被执行进入了死锁，**所以我们在任何时刻都不应该去阻塞UI线程（主线程）**，这也就是await and async模式的作用，await关键字并不会去真正阻塞UI线程（主线程），它会让UI线程（主线程）返回去做其它事情，待await关键字等待的Task对象执行完毕后，在合适的时候（这里所说的**"合适的时候"**是.NET自己判断的，我们不用去管）.NET会让UI线程（主线程）回来继续执行await关键字后面的代码，这样即便await关键字后面的代码设置了控件的属性，也是由UI线程（主线程）来设置的，所以就避免了UI项目的线程安全异常问题。

- 第一种方法

  ```c#
  public static async Task<JObject> GetJsonAsync(Uri uri)
  {
    using (var client = new HttpClient())
    {
      var jsonString = await client.GetStringAsync(uri).ConfigureAwait(false);
      return JObject.Parse(jsonString);
    }
  }
  ```

  This changes the continuation behavior of GetJsonAsync so that it does *not* resume on the context. Instead, GetJsonAsync will resume on a thread pool thread. This enables GetJsonAsync to complete the Task it returned without having to re-enter the context. The top-level methods, meanwhile, do require the context, so they cannot use ConfigureAwait(false).

  > **Using ConfigureAwait(false) to avoid deadlocks is a dangerous practice.** You would have to use ConfigureAwait(false) for every await in the transitive closure of all methods called by the blocking code, including all third- and second-party code. Using ConfigureAwait(false) to avoid deadlock is at best just a hack).
  > As the title of this post points out, the better solution is “Don’t block on async code”.

- 第二种方法

  ```c#
  public async void Button1_Click(...)
  {
    var json = await GetJsonAsync(...);
    textBox1.Text = json;
  }
  
  public class MyController : ApiController
  {
    public async Task<string> Get()
    {
      var json = await GetJsonAsync(...);
      return json.ToString();
    }
  }
  ```

  

  This changes the blocking behavior of the top-level methods so that the context is never actually blocked; all “waits” are “asynchronous waits”.

  **Note:** It is best to apply both best practices. Either one will prevent the deadlock, but *both* must be applied to achieve maximum performance and responsiveness.

- 第三种方法

  ```c#
  // My "library" method.
  public static async Task<JObject> GetJsonAsync(Uri uri)
  {
      using (var client = new HttpClient())
      {
          var jsonString = await client.GetStringAsync(uri);
          return JObject.Parse(jsonString);
      }
  }
  
  // My "top-level" method.
  public string Get()
  {
      JObject jObject = null;
  
      Task.Run(async () =>
      {
          jObject = await GetJsonAsync(...);
          //await之后的代码
      }).Wait();//此处启动线程是为了防止Async & Await模式造成死锁
  
      return jObject.ToString();
  }
  ```

  这样因为GetJsonAsync方法是由Task.Run新启动的线程来调用的，而Task.Run新启动的线程是线程池线程，该线程没有SynchronizationContext，所以在await GetJsonAsync(...)执行完毕之后，一个线程（有可能就是执行await关键字之前的线程，也有可能是一个新的线程，如本文开始时所述）不需要获得"context"就可以继续执行await之后的代码，不会和top-level method的线程阻塞，造成死锁。



**最后再补充说一点，本文提到的await and async死锁问题，在.NET控制台项目和ASP.NET Core项目（因为微软在ASP.NET Core中移除了SynchronizationContext，[详情可以查看这里](https://www.cnblogs.com/heyuquan/p/async-deadlock.html)）中并不存在。因为经过实验发现在.NET控制台项目和ASP.NET Core项目中，await关键字这一行后面的代码不需要线程重新进入"context"就可以执行，也就是说在.NET控制台项目和ASP.NET Core项目中就算不调用Task.ConfigureAwait(false)，await关键字这一行后面的代码也会由一个线程池线程来成功执行，不会和主线程发生死锁。但是在Winform和老的ASP.NET（指.NET Framework中的ASP.NET）中就会发生死锁。**



