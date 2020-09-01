---
title: Things to check when high memory occurs
description: This article describes quick things to check when you experience high memory in ASP.NET.
ms.date: 07/28/2020
ms.prod-support-area-path: 
ms.reviewer: mgraham
---
# Quick things to check when you experience high memory levels in ASP.NET

This article describes the quick things to check when you experience high memory in Microsoft ASP.NET.

_Original product version:_ &nbsp; ASP.NET  
_Original KB number:_ &nbsp; 893660

## Introduction

This article will start with some common issues, actions you can take to remedy these issues, and a brief explanation of why these situations can cause problems.

## ASP.NET Support Voice column

In the April 2005 Support Voice column, we inadvertently provided a link to the wrong file. Instead of linking to a download for the Web service, we linked to the XML file that is returned by the Web service. That link has been corrected. If you'd like to review the article with the correct file attached, see [Dynamic page updates using XMLHTTP](https://support.microsoft.com/help/893659/dynamic-page-updates-using-xmlhttp).  

## What is considered high memory

Obviously, this is going to be dependent on volume and activity of specific applications. But, in general, high memory is when see that your Aspnet_wp.exe process (Internet Information Services (IIS)) or W3wp process (IIS) memory is consistently increasing and is not returning to a comfortable level. In general terms, a comfortable level would be under 600 MB in the default 2GB user memory address space. Once the memory level is higher than that, we are performing less than we should be, and this may affect other applications running on the system. The key here is to understand that some applications require more memory than others, and if you are exceeding these limits, you may want to add more memory or add another server to your Web farm (or consider a Web farm). Also, profiling is recommended in these cases, which can enable developers to create leaner applications. In this article, we are looking at a situation where you consistently see memory rise until the server stops performing.

## Application set up for debugging

One reason for high memory that we see here in Support a lot is when you have debugging, tracing, or both enabled for your application. While you are developing your application, this is a necessity. By default, when you create your application in Visual Studio .NET, you will see the following attribute set in your *Web.config* file:

```xml
<compilation
 ...
 debug="true"
 />
```

and/or

```xml
 <trace
 enabled="true"
 ...
 />
```

Also, when you do a final build of your application, make sure that you do this in Release mode, not Debug mode. Once you are in production, this should no longer be necessary. It can really slow down your performance and eat up your memory. Setting this attribute means you change a few things about how you handle your application. First, batch compile will be disabled, even if it's set in this `compilation` element. What this means is that you create an assembly for every page in your application so that you can break into it. These assemblies can be scattered randomly across your memory space, making it more and more difficult for you to find the contiguous space to allocate memory for when you need it. Second, the `executionTimeout` attribute ([\<httpRuntime> Element](/previous-versions/dotnet/netframework-1.1/e1f13641(v=vs.71))) is set to a high number, overriding the default of 90 seconds. This is fine when debugging, because you can't have the application time out while you patiently step through the code to find your blunders. However, it is a significant risk in production. This means that should you have a rogue request for whatever reason, it will hold on to a thread and continue any detrimental behavior for days rather than just a minute and a half. Finally, you will be creating more files in your *Temporary ASP.NET* files folder, and the `System.Diagnostics.DebuggableAttribute` ([System.Diagnostics Namespace](/dotnet/api/system.diagnostics&view=dotnet-plat-ext-3.1)) gets added to all generated code, which can cause performance degradation.

If you get nothing else from this article, I do hope you get this. Leaving debugging enabled is bad. We see this all too often, and it is so easy to change. Also, remember that this can be set at the page level, so make sure that all of your pages are not setting this.

## String concatenation

There are applications that build HTML output by using server-side code and by just building one large HTML string to send to the browser. This is fine, but if you are building the string by using `+` and `&` concatenation, you may not be aware of how many large strings you are building. For example:

```csharp
string mystring = "<html>";
mystring = mystring + "<table><tr><td>";
mystring = mystring + "First Cell";
mystring = mystring + "</td></tr></table>";
mystring = mystring + "</html>";
```

This code seems harmless enough, but here's what you are storing in memory:

```html
<html>
<html><table><tr><td>
<html><table><tr><td>First Cell
<html><table><tr><td>First Cell</td></tr></table>
<html><table><tr><td>First Cell</td></tr></table></html>
```

You may think that you are just storing the last line, but you are storing *all* of these lines. You can see how this could get out of hand, especially when you are building a large table, perhaps by looping through a large recordset. If this is what you are doing, use our `System.Text.StringBuilder` class, so that you just store the one large string. See [Use Visual C# to improve string concatenation performance](/troubleshoot/dotnet/csharp/string-concatenation)

## .NET Framework Service Pack 1 (SP1)

If you are not running the .NET Framework SP1 yet, install this if you are experiencing memory issues. I won't go into great detail, but basically, with SP1 we are now allocating memory in a much more efficient manner. Basically, we are allocating 16 MB at a time for large objects rather than 64 MB at a time. We've all moved, and we all know that we can pack a lot more into a car or truck if we are using many small boxes rather than a few large boxes. That is the idea here.

## Don't be afraid to recycle periodically

In IIS, by default, we recycle application pools every 29 hours. In IIS, the Aspnet_wp.exe process will keep going until you end the task, restart IIS, or restart the computer. This means that this process could be running for months. For some applications, it's a good idea to just restart the worker process every couple of days or so, at a convenient time.

## Questions to ask

The previous were all things that you can fix quickly. However, if you are experiencing memory issues, ask yourself these questions:

- Am I using many large objects? More than 85,000 KB are stored in a large object heap.

- Am I storing objects in Session state? These objects are going to stay in memory for much longer than if you use and dispose them.

- Am I using the `Cache` object? When it is used wisely, this is a great benefit to performance. But when it is used unwisely, you wind up with much memory used that is never released.

- Am I returning `recordsets` too large for a Web application? No one wants to look at 1,000 records on a Web page. You should be designing your application so that you never get more than 50 to 100 records in one trip.

## Debugging

I won't get into setting up WinDbg. But here are some commands you can use to see what exactly is in your memory, if you wish to troubleshoot more complicated issues.

```console
!eeheap -gc
```

This command will show you how much managed memory you have. If this value is high, there is something that your managed code is building.

```console
!dumpheap -stat
```

This command will take quite a while to run, even hours if your memory is large. But this command will give you a list of all of your objects, how many of each type, and how much memory each is using. (For example, for the `StringBuilder` class, you will see many `System.String` objects)

Once you have found an object taking much memory, you can dig further by using the following command:

```console
!do <addr>
```

You can get the address of the object you are looking for in the `dumpheap` command.

To learn more about how to set up and use these commands, see [Troubleshooting ASP.NET using WinDbg and the SOS extension](https://support.microsoft.com/help/892277/troubleshooting-asp-net-using-windbg-and-the-sos-extension).

We'll be trying to incorporate more ways to use this wonderful diagnostic tool for specific situations in these columns. Let us know if we are doing a good job with this!

## Memory and performance articles

[Garbage Collector Basics and Performance Hints](/previous-versions/dotnet/articles/ms973837(v=msdn.10))

[Developing High-Performance ASP.NET Applications](/previous-versions/dotnet/netframework-1.1/5dws599a(v=vs.71))

[ASP.NET Performance Monitoring, and When to Alert Administrators](/previous-versions/dotnet/articles/ms972959(v=msdn.10))

[Improving .NET Application Performance and Scalability](/previous-versions/msp-n-p/ff649152(v=pandp.10))