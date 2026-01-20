---
title: 由 Waypoint 数据包异常引起的网络协议错误问题
published: 2026-01-20
description: 记录一次由于服务端发送异常 Waypoint 数据包，导致客户端协议错误掉线的问题排查、分析与解决过程。
tags:
  - Waypoint
  - 网络协议错误
category: Minecraft
draft: false
---

## 问题现象

在游玩服务器过程中，我注意到一个的现象：

- **部分玩家会因为网络协议错误而掉线**
- 而在**相同服务器、相同游戏版本**的情况下，我本人并没有遇到该问题

从表象上看，这似乎是一个只影响“部分玩家”的问题。

---

## 初步判断与思路

由于我曾阅读过 Minecraft 客户端中与网络协议错误相关的代码，因此在第一时间我就意识到：

> **该问题是出现在客户端处理服务端数据包的过程中**

基于这一认知，我最初的判断是：

- 可能有某个 **客户端模组**
- 在处理相关数据包时引入了异常逻辑
- 从而导致客户端抛出协议错误并断开连接

但随着进一步排查，这一判断很快被推翻。

---

## 获取并分析玩家客户端日志

在向发生掉线问题的玩家索要客户端日志后，我在日志中注意到一个关键的异常信息：

```text
java.lang.NullPointerException: Cannot invoke "net.minecraft.class_11200.method_70766(net.minecraft.class_11200)" because the return value of "java.util.Map.get(Object)" is null
	at knot/net.minecraft.class_11264.method_70956(class_11264.java:24) ~[client-intermediary.jar:?]
````

该异常清楚地表明：

* 客户端在处理某个来自服务端的数据包时
* 从 `Map` 中取到了一个 **不存在的值**

---

## 使用 Linkie 查询映射

为了进一步弄清楚异常发生的位置，我通过 **Linkie** 查询了对应的映射：

* [class_11264.method_70956](https://linkie.shedaniel.dev/mappings?namespace=yarn&version=1.21.8&search=class_11264.method_70956)
* [class_11200.method_70766](https://linkie.shedaniel.dev/mappings?namespace=yarn&version=1.21.8&search=class_11200.method_70766)


随后，我在 **IntelliJ IDEA** 中查看了反编译后的对应代码。

---

## 定位空指针异常的具体代码

在源码中，可以看到导致异常的核心逻辑如下：

```java
((TrackedWaypoint) this.waypoints.get(trackedWaypoint.getSource())).handleUpdate(trackedWaypoint);
```

异常的根本原因非常明确：

* `this.waypoints.get(trackedWaypoint.getSource())` 返回了 `null`
* 随后代码直接调用了其方法
* 最终触发了 `NullPointerException`

也就是说：

> **客户端收到了一个引用了“不存在 Waypoint”的更新数据包**

---

## 推翻最初判断：并非客户端模组导致

在确认异常发生位置后，我开始重新审视之前“客户端模组作祟”的判断。

通常情况下：

* 客户端模组 **极少会直接修改 `waypoints` 这样的核心数据结构**
* 更不太可能导致其中的数据出现缺失

为了进一步验证这一点，我查看了**自己客户端的日志**，结果发现：

* **我同样遇到了完全一致的报错**
* 但我并没有因此掉线

---

## Network Protocol Disconnect 的影响

进一步检查后，我发现自己安装了一个客户端模组：

> **Network Protocol Disconnect**

该模组的作用是：

* 在发生网络协议相关异常时
* **阻止客户端因异常而直接断开连接**

这也解释了为何：

* 其他玩家在触发该异常时会直接掉线
* 而我只是日志中出现报错，游戏仍可继续进行

至此可以确认：

> **掉线的原因并非客户端导致的，而是服务端发送了错误的数据包，使得客户端检测到处理数据包发送错误，主动断开连接**

---

## 第一种解决方案：安装Network Protocol Disconnect模组

因此，第一个可行的解决方案已经非常明确：

* 在客户端安装 **Network Protocol Disconnect**
* 即可避免因该异常导致的强制掉线

不过，该方案仍然存在不足：

* 客户端日志中依然会产生异常信息

---

## 客户端侧的彻底规避方案

为了从逻辑层面完全阻止该异常的发生，我编写了一个 **Fabric 客户端模组（1.21.8）**。

核心思路非常简单：

> **在执行 Waypoint 更新逻辑之前，先判断对应 Waypoint 是否存在**

---

## Mixin 实现逻辑

> ⚠️ 补充说明
> 为避免潜在的违反 Minecraft EULA 的风险：
>
> * 上文引用的原版代码使用的是 **Yarn mappings**
> * 下文我编写的代码使用的是 **Mojang mappings**

```java
@Mixin(ClientWaypointManager.class)
public class MixinClientWaypointManager {

    @Shadow @Final
    private Map<Either<UUID, String>, TrackedWaypoint> waypoints;

    @Inject(
        method = "updateWaypoint(Lnet/minecraft/world/waypoints/TrackedWaypoint;)V",
        at = @At("HEAD"),
        cancellable = true
    )
    private void guardNullWaypoint(TrackedWaypoint trackedWaypoint, CallbackInfo ci) {
        if (this.waypoints.get(trackedWaypoint.id()) == null) {
            ci.cancel();
        }
    }
}
```

---

## 实际测试与调试验证

在正常游玩的客户端中安装该模组后：

* 客户端日志中 **不再出现相关异常**
* 游戏过程中也未再发生掉线情况

同时，我使用 **IntelliJ IDEA 调试游戏**：

* 在 `ci.cancel()` 处设置断点
* 确认该注入逻辑确实成功拦截了异常执行路径

---

## 问题的根源仍在服务端

需要强调的是：

* 该模组只是客户端侧的防御性处理
* 问题的**根源仍然来自服务端**

也就是说：

> **服务端在某些情况下发送了引用不存在 Waypoint 的错误数据包**

---

## 服务端侧的解决方案

在服务端侧，也存在一个相对直接的解决方式：

```text
/gamerule locatorBar false
```

该指令会：

* 禁用玩家定位栏功能
* 从根源上避免该类 Waypoint 数据包的发送

代价是：

* 玩家将无法使用定位栏相关功能

---

## 模组下载与源码

* GitHub 仓库：
  [https://github.com/AsOne2020/WaypointPacketError](https://github.com/AsOne2020/WaypointPacketError)

* 模组下载（Fabric 1.21.8）：
  [https://github.com/AsOne2020/WaypointPacketError/releases](https://github.com/AsOne2020/WaypointPacketError/releases)

---

如果对该问题的服务端成因感兴趣，还有不少值得进一步深入研究的空间。

---

## 关于 AI 辅助写作的说明

本文中的问题排查、测试过程、源码定位、模组编写及结论均来自我个人的实际实践。  
AI 仅用于整理和优化文字表达，以提升可读性，未参与实际问题排查与解决。
