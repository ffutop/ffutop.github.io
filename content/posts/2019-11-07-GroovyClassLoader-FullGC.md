---
title: GroovyClassLoader 引发的 FullGC
author: fangfeng
date: 2019-11-06
categories:
  - 技术
tags:
  - Groovy
  - Java
  - FullGC
  - GC
  - Metaspace
---

## 背景

近日，一个线上应用的存活探针频繁报警。几分钟内 Full GC 次数暴涨上百次，**stop the world** :< 从 `gc.log` 中，看到的原因是触及了 `Metaspace` 的阈值，进而引发了 FGC。

```plain
2019-11-05T14:31:38.880+0800: 504600.716: [Full GC (Metadata GC Threshold) 2019-11-05T14:31:38.880+0800: 504600.716: [CMS2019-11-05T14:31:39.364+0800: 504601.201: [CMS-concurrent-mark: 0.485/0.488 secs] [Times: user=0.48 sys=0.00, real=0.49 secs]
 (concurrent mode failure): 266828K->266865K(873856K), 1.3752377 secs] 267430K->266865K(1267072K), [Metaspace: 204152K->204152K(1267712K)], 1.3757441 secs] [Times: user=1.36 sys=0.00, real=1.38 secs]
2019-11-05T14:31:40.256+0800: 504602.092: [Full GC (Last ditch collection) 2019-11-05T14:31:40.256+0800: 504602.092: [CMS: 266865K->266841K(873856K), 0.8590901 secs] 266865K->266841K(1267072K), [Metaspace: 204152K->204152K(1267712K)], 0.8595047 secs] [Times: user=0.86 sys=0.00, real=0.86 secs]
2019-11-05T14:31:41.117+0800: 504602.953: [Full GC (Metadata GC Threshold) 2019-11-05T14:31:41.117+0800: 504602.953: [CMS: 266841K->266816K(873856K), 0.9109948 secs] 267218K->266816K(1267072K), [Metaspace: 204153K->204153K(1267712K)], 0.9114975 secs] [Times: user=0.91 sys=0.00, real=0.91 secs]
2019-11-05T14:31:42.028+0800: 504603.865: [Full GC (Last ditch collection) 2019-11-05T14:31:42.028+0800: 504603.865: [CMS: 266816K->266769K(873856K), 1.0351283 secs] 266816K->266769K(1267072K), [Metaspace: 204153K->204153K(1267712K)], 1.0355630 secs] [Times: user=0.92 sys=0.00, real=1.04 secs]
```

FGC 前后，`Metaspace` 占用的内存没有得到任何释放。*[Metaspace: 204153K->204153K(1267712K)]* 。这也就是反复 FGC 的原因。

<!--more-->

## 问题排查

由于这个问题是近期第二次出现，项目代码已经跑了好几年。从最近的 Git 提交记录入手，找到了很重要的变更记录——开始启动 4 年不曾真正动用的 `GroovyClassLoader` 。用以支持在 Java 项目中的动态脚本。没曾想这就是噩梦的开始...

通过 MAT 分析 Java 堆，泄露分析报告给到的三个可能的问题都很简单地排除了。由于经验的缺失，排除了占用大块内存的类实例之后，就直接放弃了这条路（后来发现分析零散的类，就可以直接定位问题了:<）

抽取了一些核心的代码，大体上就是下面这三个类啦。

**Loader.java**

```java
package com.ffutop.loader;

import com.ffutop.api.Action;
import groovy.lang.GroovyClassLoader;

import java.util.HashMap;
import java.util.Map;

/**
 * @author fangfeng
 * @since 2019/11/6
 */
public class Loader {

    private static final String printScript = "import com.ffutop.api.Action; public class HelloAction implements Action { public void print() { System.out.println(\"Hello, World.\"); }}";

    public Map<String, Action> loadScript() throws IllegalAccessException, InstantiationException {
        Map<String, Action> map = new HashMap<String, Action>();
        GroovyClassLoader groovyClassLoader = new GroovyClassLoader();
        Class clazz = groovyClassLoader.parseClass(printScript);
        Action action = (Action) clazz.newInstance();
        map.put("HelloAction", action);
        return map;
    }
}
```

**Action.java**

```java
package com.ffutop.api;

/**
 * @author fangfeng
 * @since 2019/11/6
 */
public interface Action {
    void print();
}
```

**Main.java**

*在项目中，这个 Main 其实是一个定时任务，定期去数据库抓取的 Groovy 脚本，完成编译并做缓存（请先忽略这个做法的合理性）*

```java
package com.ffutop;

import com.ffutop.api.Action;
import com.ffutop.loader.Loader;

import java.io.IOException;
import java.util.Map;

/**
 * @author fangfeng
 * @since 2019/11/6
 */
public class Main {
    public static void main(String[] args) throws InstantiationException, IllegalAccessException, IOException {
        Loader loader = new Loader();
        for (int i=0;i<1000000;i++) {
            Map<String, Action> map = loader.loadScript();
            Action action = map.get("HelloAction");
            action.print();
        }
        System.in.read();
    }
}
```

很遗憾，问题的根源在于 Maven 引入的 groovy 库的版本，用 2.5.7 版本反复调试了一天都没有复现，最后才从版本入手成功复现 （Version <= 2.4.7）。

## 追根溯源

回过头来再审视了一遍 dump 下来的 Java 堆。从 *duplicate_classes* 也进一步证实了上面猜测的原因。

![](https://img.ffutop.com/11FB209B-8635-4ACC-8812-A0BA6133376F.png)

每次加载的脚本都被认为是各不相同的类，并在加载的同时被 Groovy 全局的 ClassInfo 做了缓存。

![](https://img.ffutop.com/FB6CA0BF-992E-4199-BF57-35C31D6D3C51.png)

至于为何 >= 2.4.8 版本的 groovy 没有问题，代码出现了重要变更。`klass` 由 `StrongReference` 变成了 `WeakReference` 。因此，也就能够保证在 GC 时回收这个引用。

![](https://img.ffutop.com/9F91AA9F-BAB4-47C3-B8CE-DC8AFEE5480A.png)

进一步的，Metaspace 存储的类的元数据信息，从收集到的信息来说，需要满足下列三个条件，才会被回收（未经考证）：

1. 该类所有的实例都已经被回收

2. 加载该类的 ClassLoader 已经被回收

3. 该类对应的 java.lang.Class 对象没有任何地方被引用

而移除 `ClassInfo.klass` 下的引用，按照分析也就是阻止元数据被回收的最后一道关卡。

## 处理方案

最后的处理方案也相当简单，由于不想直接对代码逻辑动刀，升级 `groovy-all.jar` 的版本是最简单有效的。

附上升级版本后的 Metaspace 使用情况。

![optimized](https://img.ffutop.com/944505AE-582A-4D25-80CC-F008E5DD1ECF.png)

## 参考

\[1\]. [JVM源码分析之Metaspace解密](https://lovestblog.cn/blog/2016/10/29/metaspace/)
\[2\]. [GROOVY-7683](https://issues.apache.org/jira/browse/GROOVY-7683?focusedCommentId=15066871&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-15066871)

