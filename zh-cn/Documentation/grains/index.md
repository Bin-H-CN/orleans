---
layout: page
title: Developing a Grain
---

# 准备

在编写代码以实现Grains类之前，请创建一个针对.NET Standard（首选）或.NET Framework 4.6.1或更高版本的新类库项目（如果由于依赖性而无法使用.NET Standard）。可以在同一个“类库”项目中或在两个不同的项目中定义grains接口和grains类，以更好地将接口与实现分开。无论哪种情况，项目都需要参考`Microsoft.Orleans.Core.Abstractions`和`Microsoft.Orleans.CodeGenerator.MSBuild`NuGet软件包。

有关更详尽的说明，请参见[项目设置](../tutorials_and_samples/tutorial_1.md#project-setup)的部分[教程一–Orleans基础](../tutorials_and_samples/tutorial_1.md)。

# Grains接口和类

Grains通过从外部调用各个Grains接口申明的方法进行相互交互。Grains类实现一个或多个先前声明的Grains接口。Grain接口的所有方法都必须返回`Task`（对于`virtual`方法），一个`Task<T>`或一个`ValueTask <T>`（对于返回类型为值的方法`T`）。

以下是Orleans 1.5 Presence Service示例的摘录：

```csharp
//an example of a Grain Interface
public interface IPlayerGrain : IGrainWithGuidKey
{
  Task<IGameGrain> GetCurrentGame();
  Task JoinGame(IGameGrain game);
  Task LeaveGame(IGameGrain game);
}

//an example of a Grain class implementing a Grain Interface
public class PlayerGrain : Grain, IPlayerGrain
{
    private IGameGrain currentGame;

    // Game the player is currently in. May be null.
    public Task<IGameGrain> GetCurrentGame()
    {
       return Task.FromResult(currentGame);
    }

    // Game grain calls this method to notify that the player has joined the game.
    public Task JoinGame(IGameGrain game)
    {
       currentGame = game;
       Console.WriteLine(
           "Player {0} joined game {1}",
           this.GetPrimaryKey(),
           game.GetPrimaryKey());

       return Task.CompletedTask;
    }

   // Game grain calls this method to notify that the player has left the game.
   public Task LeaveGame(IGameGrain game)
   {
       currentGame = null;
       Console.WriteLine(
           "Player {0} left game {1}",
           this.GetPrimaryKey(),
           game.GetPrimaryKey());

       return Task.CompletedTask;
   }
}
```

# Grains方法的返回值

返回`T`类型值的grain方法在grain接口中定义为返回`Task<T>`。对于未标有async关键字，当返回值可用时，通常通过以下语句返回：

```csharp
public Task<SomeType> GrainMethod1()
{
    ...
    return Task.FromResult(<variable or constant with result>);
}
```

没有返回值的grain方法（实际上是void方法）在grain接口中定义为返回`Task`。返回的`Task`指示方法的异步执行和完成。对于未标有async关键字，当`void`方法完成执行时，需要返回的特殊值`Task.CompletedTask`：

```csharp
public Task GrainMethod2()
{
    ...
    return Task.CompletedTask;
}
```

标记为async直接返回值：

```csharp
public async Task<SomeType> GrainMethod3()
{
    ...
    return <variable or constant with result>;
}
```

一种`void`的Grains方法标记为async不返回值的代码只是在执行结束时返回：

```csharp
public async Task GrainMethod4()
{
    ...
    return;
}
```

如果grain方法从另一个异步方法调用接收到的返回值（是否返回grain），并且不需要对该调用执行错误处理，则只需返回`Task`它从该异步调用接收作为其返回值：

```csharp
public Task<SomeType> GrainMethod5()
{
    ...
    Task<SomeType> task = CallToAnotherGrain();
    return task;
}
```

同样，`void`Grain方法可以返回`Task`通过另一个调用返回给它，而不是等待它。

```csharp
public Task GrainMethod6()
{
    ...
    Task task = CallToAsyncAPI();
    return task;
}
```

`ValueTask <T>`可以代替`Task<T>`

### Grains引用

Grains引用是实现与相应Grains类相同的Grains接口的代理对象。它封装了目标Grain的逻辑标识（类型和唯一键）。Grains引用是用于调用目标Grains的工具。每个Grains引用都针对单个Grains（Grains类的单个实例），但是可以为同一个Grains创建多个独立的引用。

由于Grains引用代表目标Grains的逻辑标识，因此它与Grains的物理位置无关，即使在系统完全重启后也仍然有效。开发人员可以像其他任何.NET对象一样使用Grains引用。它可以传递给方法，用作方法的返回值等，甚至可以保存到持久化存储中。

可以通过将Grains的身份传递给`GrainFactory.GetGrain <T>(key)`方法，在哪里`T`是Grains接口，`key`是该类型中纹理的唯一键。

以下是如何获取`IPlayerGrain`上面定义的接口。

从Grains类内部：

```csharp
    //construct the grain reference of a specific player
    IPlayerGrain player = GrainFactory.GetGrain<IPlayerGrain>(playerId);
```

来自Orleans客户代码。

```csharp
    IPlayerGrain player = client.GetGrain<IPlayerGrain>(playerId);
```

### Grain方法调用

Orleans编程模型基于[异步编程](https://docs.microsoft.com/en-us/dotnet/csharp/async)。

使用上一个示例中的grain引用，这里是执行grain方法调用的方法：

```csharp
//Invoking a grain method asynchronously
Task joinGameTask = player.JoinGame(this);
//The await keyword effectively makes the remainder of the method execute asynchronously at a later point (upon completion of the Task being awaited) without blocking the thread.
await joinGameTask;
//The next line will execute later, after joinGameTask is completed.
players.Add(playerId);
```

可以加入两个或多个`Task`;联接操作创建一个新的`Task`当所有组成部分都解决时`Task`s完成。当Grains需要启动多个计算并等待所有计算完成后再继续操作时，这是一种有用的模式。例如，生成由许多部分组成的网页的前端纹理可能会进行多个后端调用，每个部分一个，并接收一个`Task`对于每个结果。然后Grains将等待所有这些的加入`Task`;当加入`Task`解决了，个人`Task`已完成，并且已收到格式化网页所需的所有数据。

例：

```csharp
List<Task> tasks = new List<Task>();
Message notification = CreateNewMessage(text);

foreach (ISubscriber subscriber in subscribers)
{
   tasks.Add(subscriber.Notify(notification));
}

// WhenAll joins a collection of tasks, and returns a joined Task that will be resolved when all of the individual notification Tasks are resolved.
Task joinedTask = Task.WhenAll(tasks);
await joinedTask;

// Execution of the rest of the method will continue asynchronously after joinedTask is resolve.
```

### 虚方法

Grains类可以选择覆盖`OnActivateAsync`和`OnDeactivateAsync`在激活和停用类的每个Grain时，由Orleans运行时调用的虚拟方法。这使Grains代码有机会执行其他初始化和清理操作。引发的异常`OnActivateAsync`无法激活过程。而`OnActivateAsync`，如果被覆盖，则始终被称为Grains激活过程的一部分，`OnDeactivateAsync`不能保证在所有情况下（例如在服务器故障或其他异常事件的情况下）都会被调用。因此，应用程序不应依赖`OnDeactivateAsync`用于执行关键操作，例如状态变化的持久化。他们应仅将其用于尽力而为的操作。