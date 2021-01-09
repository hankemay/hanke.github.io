---
layout: post
title: "Perf Test Tools -- Gatling"
subtitle: "Test Tools"
date: 2021-01-09
author: "Hanke"
header-style: "text"
tags: [Test, Test Tools, Gatling]
---
> 今天做作业的时候需要做一个网站的压测，搜到了这个好用工具，Track一下，后续找时间仔细研究一下。

### Gatling
[Gating](https://gatling.io/open-source/start-testing/)

### IDEA
* IDEA安装Scala插件 `File -> Settings -> Plugins`, 搜索Scala，点击`install`
* Create New Project
    * Select Maven
    * Create from archetype -> Add Archetype and fill in
>  * groupid: io.gatling.highcharts
>  * artifactid: gatling-highcharts-maven-archetype
>  * version: 2.3.7 ([maven version](https://mvnrepository.com/))
* 填写项目对应的groupid, artifactid 

### Scala
编写Scala code来模拟做压测  
```scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
/**
  * Created at 2021-01-09 18:27
  *
  * Description:
  * PerfTestSimulation
  *
  * @author hmxiao
  * @version 1.0
  */
class PerfTestSimulation extends Simulation{
  val httpConf = http
    .baseURL("http://www.baidu.com")
    .acceptHeader("text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8")
    .acceptLanguageHeader("en-US;q=0.5")
    .acceptEncodingHeader("gzip, deflate")
    .userAgentHeader("Mozilla/5.0 (Windows NT 5.1; rv:31.0) Gecko/20100101 Firefox/31.0")

  val data = csv("baidu_search.csv").random

  val scenarioA = scenario("BaiduPerfTest")
    .feed(data)
    .exec(http("requestsearch")
      .get("/s?${params}")
      .check(status.is(200)))

  setUp(scenarioA.inject(constantUsersPerSec(10).during(10)).protocols(httpConf))

}
```

### 运行Engine
右键Scala路径test下的Engine文件，run engine 按提示信息输入对应的要运行的simulation。 在运行结果的最后会输出index html的地址打开看即可。   
上面的[运行结果](/ref/perftestsimulation-1610190130151/index.html)



### Reference
* [参考环境博客](https://blog.csdn.net/mosicol/article/details/89060040)
* [参考博客](https://blog.csdn.net/lb245557472/article/details/80967889)






<b><font color="red">本网站的文章除非特别声明，全部都是原创。
原创文章版权归数据元素</font>(</b>[DataElement](https://www.dataelement.top)<b><font color="red">)所有，未经许可不得转载!</font></b>  
**了解更多大数据相关分享，可关注微信公众号"`数据元素`"**
![数据元素微信公众号](/img/dataelement.gif)
