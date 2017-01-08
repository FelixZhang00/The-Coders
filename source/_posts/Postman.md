layout: 工具
title: Postman
date: 2016-12-28 13:08:17

---

## Postman工具

**[Docs](https://www.getpostman.com/docs/)**



### 变量设置

<center>
![](/images/2016/12/28/Postman设置变量1.png)


![](/images/2016/12/28/Postman设置变量2.png)

![](/images/2016/12/28/Postman设置变量3.png)
</center>


### 预调用脚本

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;遇到随着时间变化的变量，譬如：认证key，在之前简单的变量设置就显得有些不足了。最好就是在调用脚本的时候，能自动按照规则计算出变量的值，并且把变量这只到环境变量中。Postman的Pre-request Script就是很好的解决途径。

<center>
![](/images/2016/12/28/Postman-pre-request-script-01.png)
</center>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;此时，在用`{ { token } }`这样的环境变量形式(和上面提到的一样)，就可以了。
