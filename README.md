
## 简介


在ASP.NET Core中，速率限制中间件是用来控制客户端对Web API或MVC应用程序发出请求的速率，以防止服务器过载和提高安全性。


下面是 `AddRateLimiter` 的一些基本用法：


### 1\. 注册服务


在 `Startup.cs` 或 `Program.cs` 中，需要注册 `AddRateLimiter` 服务。这可以通过以下代码完成：



```
builder.Services.AddRateLimiter(options =>
{
    // 配置速率限制选项
});

```

### 2\. 添加速率限制策略


可以添加不同类型的速率限制策略， 包括固定窗口、滑动窗口、令牌桶和并发限制。


#### 固定窗口限制器（Fixed Window Limiter）


固定窗口限制器使用固定的时间窗口来限制请求。当时间窗口到期后，会开始一个新的时间窗口，并重置请求限制。例如，可以设置一个策略，允许每个12秒的时间窗口内最多4个请求。



```
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1); // 时间窗口
        opt.PermitLimit = 3; // 在时间窗口内允许的最大请求数
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst; // 请求处理顺序
        opt.QueueLimit = 2; // 队列中允许的最大请求数
    });
});
app.UseRateLimiter();

```

即固定时间请求的次数，超过次数就会限流，下一个窗口时间将次数重置


经过测试，多余的请求还是会等待



> [https://github.com/guoxiaotian/p/17834892\.html](https://github.com)


#### 滑动窗口限制器（Sliding Window Limiter）


滑动窗口算法：


* 与固定窗口限制器类似，但为每个窗口添加了段。 窗口在每个段间隔滑动一段。 段间隔的计算方式是：(窗口时间)/(每个窗口的段数)。
* 将窗口的请求数限制为 permitLimit 个请求。
* 每个时间窗口划分为一个窗口 n 个段。
* 从倒退一个窗口的过期时间段（当前段之前的 n 个段）获取的请求会添加到当前的段。 我们将倒退一个窗口最近过期时间段称为“过期的段”。


请考虑下表，其中显示了一个滑动窗口限制器，该限制器的窗口为 30 秒、每个窗口有三个段，且请求数限制为 100 个：


* 第一行和第一列显示时间段。
* 第二行显示剩余的可用请求数。 其余请求数的计算方式为可用请求数减去处理的请求数和回收的请求数。
* 每次的请求数沿着蓝色对角线移动。
* 从时间 30 开始，从过期时间段获得的请求会再次添加到请求数限制中，如红色线条所示。
![](https://i-blog.csdnimg.cn/img_convert/0be59048ade0beba218832688219659b.png)


下表换了一种格式来显示上图中的数据。 “可用”列显示上一个段中可用的请求数（来自上一个行中的“结转”）。 第一行显示有 100 个可用请求，因为没有上一个段。




| 时间 | 可用 | 获取的请求数 | 从过期段回收的请求数 | 结存请求数 |
| --- | --- | --- | --- | --- |
| 0 | 100 | 20 | 0 | 80 |
| 10 | 80 | 30 | 0 | 50 |
| 20 | 50 | 40 | 0 | 10 |
| 30 | 10 | 30 | 20 | 0 |
| 40 | 0 | 10 | 30 | 20 |
| 50 | 20 | 10 | 40 | 50 |
| 60 | 50 | 35 | 30 | 45 |



```
 services.AddRateLimiter(options =>
    {
        options.AddSlidingWindowLimiter("sliding", opt =>
        {
            opt.Window = TimeSpan.FromMinutes(1); // 总窗口时间为1分钟
            opt.SegmentsPerWindow = 6; // 将1分钟的窗口分为6个段，即每10秒一个段
            opt.PermitLimit = 10; // 整个窗口时间内允许的最大请求数
        });
    });

```

#### 令牌桶限制器（Token Bucket Limiter）


令牌桶限制器维护一个滚动累积的使用预算，作为一个令牌的余额。它以一定的速率添加令牌，当服务请求发生时，服务尝试提取一个令牌（减少令牌计数）来满足请求。如果没有令牌，服务就达到了限制，响应被阻塞。



```
    services.AddRateLimiter(configureOptions =>
    {
        configureOptions.AddTokenBucketLimiter("token-bucket", options =>
        {
            options.TokenLimit = 100; // 桶的容量
            options.ReplenishmentPeriod = TimeSpan.FromSeconds(10); // 补充周期，即每10秒补充一次令牌
            options.TokensPerPeriod = 10; // 每个周期补充的令牌数
            options.AutoReplenishment = true; // 是否自动补充令牌
            options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst; // 队列处理顺序
            options.QueueLimit = 10; // 请求队列长度限制
        });
    });


```

#### 并发限制器（Concurrency Limiter）


并发限制器是最简单的速率限制形式。它不关注时间，只关注并发请求的数量。



```
    services.AddRateLimiter(options =>
    {
        options.AddConcurrencyLimiter("concurrency", options =>
        {
            options.PermitLimit = 1; // 最大并发请求数
            options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst; // 队列处理顺序
            options.QueueLimit = 10; // 请求队列长度限制
        });
    });

```

### 3\. 使用中间件


在 `Configure` 方法或 `Program.cs` 中，需要使用 `UseRateLimiter` 中间件：



```
app.UseRateLimiter();

```

### 4\. 应用速率限制策略


可以全局应用速率限制策略，或者将其应用于特定的控制器或动作：


#### 全局配置



```
app.MapControllers().RequireRateLimiting("fixed");

```

#### 应用于特定的控制器



```
[EnableRateLimiting("fixed")]
public class RateLimitTestController : ControllerBase
{
    // 控制器动作
}

```

#### 应用于特定的动作



```
[EnableRateLimiting("fixed")]
public async Task Get()
{
    // 动作逻辑
}

```

### 5\. 禁用速率限制


也可以选择禁用速率限制，无论是在控制器级别还是特定动作级别：


#### 禁用控制器级别的速率限制



```
[DisableRateLimiting]
public class RateLimitTestController : ControllerBase
{
    // 控制器动作
}

```

#### 禁用特定动作的速率限制



```
[DisableRateLimiting]
public async Task Get()
{
    // 动作逻辑
}

```

### 自定义响应


当客户端超出速率限制时，可以自定义响应。例如，可以设置`OnRejected`回调来自定义响应：



```
options.OnRejected = (context, token) =>
{
    context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
    context.HttpContext.Response.Headers["Retry-After"] = "60"; // 建议60秒后重试
     context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
 context.HttpContext.Response.WriteAsync("Too many requests. Please try again later.", cancellationToken: token);
                   
    return Task.CompletedTask;
};

```

![](https://i-blog.csdnimg.cn/direct/a01bf00182ef40e0b09b859a4eec9b52.png)


## 总结


在ASP.NET Core应用程序中实现有效的速率限制策略，以保护的API免受滥用和过载。欢迎关注我的公众号：Net分享


 本博客参考[蓝猫加速器官网](https://lanmaovqn.com/)。转载请注明出处！
