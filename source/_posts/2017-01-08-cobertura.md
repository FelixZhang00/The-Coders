---
title: 【2017-01-08 cobertura】
date: 2017-01-08 15:14:31
tags: Java
---

## cobertura

### [cobertura website](http://cobertura.sourceforge.net/)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;cobertura是一个用于分析Java单测覆盖率的工具，我按照自己maven工程配置，来说下吧。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**也不知道为什么，反正我官网没有打开过，资料也不多**

---

### 参考文档

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; [apache maven plugin](http://www.mojohaus.org/cobertura-maven-plugin/)，这个可以打开😂。


### 引入

```
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>cobertura-maven-plugin</artifactId>
        <version>${cobertura.version}</version>
        <configuration>
          <instrumentation>
            <includes>			// 规定计算哪一部分的单测覆盖率，如DAO的覆盖率没必要计算
              <include>com/van/service/**/*.class</include>
            </includes>
          </instrumentation>
        </configuration>
      </plugin>
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 按照上面的pom配置，cobertura只会计算com.van.service.\*\*.\*.class的单测覆盖率。同理，还可以配置<excludes>标签来排除不需要计算的类。  

### 计算

`
mvn -B cobertura:cobertura -Dmaven.test.failure.ignore=true
`  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  执行上述命令，会在对应的targe/sit/cobertura/下生成需要的内容，打开对应的index.html即可以看到单测覆盖率。    
<center>
![cobertura命令执行](/images/2017/01/08/cobertura执行.png)

![cobertura统计结果](/images/2017/01/08/cobertura统计结果.png)
</center>

### 值得注意

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 值得注意的是，在我自己的测试(mavne:3, cobertura-maven-plugin:5.0.2)和stack overflow上的提问中，都有类似的情况：cobertura的<ignore>标签在maven中似乎没有什么作用，不过有<include>、<exclude>应该也能满足要求。
[stack overflow - cobertura-exclude-vs-ignore](http://stackoverflow.com/questions/25900958/cobertura-exclude-vs-ignore)

<center>
![cobertura stack overflow ignore vs exclude](/images/2017/01/08/stackoverflow-cobertura-ignore-vs-exclude.png)
</center>
