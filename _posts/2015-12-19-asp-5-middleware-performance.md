---
layout: post
title: MVC vs Middleware Performance in ASP.NET 5
---

As part of my day job I have been doing some performance testing using the really great [wrk](https://github.com/wg/wrk) load generator. Having benchmarked ASP.NET 4 WebAPI services running in IIS, I became curious about the different performance of the new ASP.NET 5 runtime when exposed to stress. I know about the [aspnet/benchmarks](https://github.com/aspnet/benchmarks) repository on GitHub and was interested what the performance differences would be when running as an API with controllers and the other standard ceremony that one gets from the stack compared to just injecting a handler as a raw middleware handler in Startup.cs.

I was also interested in the MacOS X development experience for the new version of the .NET framework. I use a MacBook Pro both at home and at work (separate computers), only at work I spend 90% of my time in a Windows 8.1 Parallels VM. I have a similar setup on my home computer, but [Visual Studio Code](https://code.visualstudio.com) has been showing a lot of promise as a native OS X app that I can write C# in.

First, I created a new ASP.NET API project using the Yeoman generator per the [instructions on the asp.net website](http://docs.asp.net/en/latest/client-side/yeoman.html). With that, I defined a very simple `HelloWorldController` that just returned the string "Hello World" and `DateTime.UtcNow`:

``` csharp
[Route("api/[controller]")]
public class HelloController : Controller
{
    [HttpGet]
    public async Task<string> Get()
    {
        return $"Hello world! {DateTime.UtcNow}";
    }
}
```

Hitting it with wrk shows that this setup can handle just under 5125 requests per second:

    Running 1m test @ http://localhost:5000/api/hello
      25 threads and 50 connections
      Thread Stats   Avg      Stdev     Max   +/- Stdev
        Latency    13.06ms   57.52ms   1.45s    97.37%
        Req/Sec   268.11    110.98     1.45k    69.80%
      307861 requests in 1.00m, 54.90MB read
      Socket errors: connect 0, read 0, write 0, timeout 32
    Requests/sec:   5124.58
    Transfer/sec:      0.91MB

I then created another project using Yeoman, but this time instead of using MVC and a controller I used the middleware interface to perform the request instead:

``` csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    app.Run(async context => {
        context.Response.ContentType = "text/plain";
        var helloWorldBytes = Encoding.UTF8.GetBytes($"Hello World! {DateTime.UtcNow}");
        await context.Response.Body.WriteAsync(helloWorldBytes, 0, helloWorldBytes.Length);
    });
}
```

Which yielded an impressive 26226 requests per second.

    Running 1m test @ http://localhost:5000
      25 threads and 50 connections
      Thread Stats   Avg      Stdev     Max   +/- Stdev
        Latency     9.74ms   75.82ms   1.38s    98.25%
        Req/Sec     1.35k   483.66     5.52k    53.52%
      1576268 requests in 1.00m, 258.56MB read
      Socket errors: connect 0, read 0, write 0, timeout 78
    Requests/sec:  26226.63
    Transfer/sec:      4.30MB

On it's face, it appears that using a middleware handler can yield around five times more requests per second than the MVC handler can. Admittedly, this is a very simplistic comparison and ignores much of the intricacy of what MVC actually provides: routing, request body serialization, and a whole mess of other functionality that is not present when using the raw middleware handlers. I seriously doubt anyone will be deploying an app into production as simple as my tiny benchmark.

All that aside, the new composable middleware interface in ASP.NET 5 is very cool and the idea that rather than having to use the entire MVC framework one can compose a tighter, purpose-driven request pipeline is powerful. While it isn't quite  [Netty](http://netty.io/), it's definitely compelling. I can imagine a lot of really great integrations with lighter-weight frameworks ([Nancy](http://nancyfx.org) and [ServiceStack](https://servicestack.net) come to mind) that seem to become first-class citizens with this approach.

Visual Studio Code on Mac is highly promising but isn't quite there yet IMHO. Maybe it's the years of Visual Studio training speaking but I found that the VSCode approach of having external tools to generate projects feels a little janky. The build experience is not my favorite either. I use iTerm2 on all my Macs and Visual Studio code still opens dnx commands in Terminal.app. There were a few times where I would close the terminal window and the dnx runtime would keep going, so when I rebuilt I had to use `ps` and `grep` to find where it was running and kill it. This felt risky because internally VSCode uses [OmniSharp](http://www.omnisharp.net), and I wasn't sure which dnx environments were stuff I had spawned to run the app and those powering OmniSharp.

Overall I am really excited about the new version of .NET. Having used C# for some time now I have seen Microsoft go from "gross-only-banks-and-boring-enterprise-apps-use-that" to capturing the top of Hacker News pretty regularly. With these new tools I think that developing for the stack can really only get better.

You can find the source code I used for these benchmarks [on GitHub](https://github.com/ametzger/basic-vnext-benchmark).
