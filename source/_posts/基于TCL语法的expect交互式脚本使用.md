## 基于TCL语法的expect交互式脚本使用



​	有些事情是是头疼且重复的，譬如登陆线上服务器存在相同密码验证的跳板机、没有持续交付自动化流程的线上代码编译、构建、部署操作等。今天来说说TCL语法的expect脚本（极其简单），用可用的脚本来注释说明即可：



### 免密码登陆服务器

​	***密码的当然是要写在脚本里的，或者只输入一次，这不是一个hack脚本…***

```shell
#!/usr/bin/expect -f

proc terminal:password:get {promptString} {		# TCL的子程序定义方式

     # Turn off echoing, but leave newlines on.  That looks better.
     # Note that the terminal is left in cooked mode, so people can still use backspace
     exec stty -echo echonl <@stdin

     # Print the prompt
     puts -nonewline stdout $promptString
     flush stdout

     # Read that password!  :^)
     gets stdin password

     # Reset the terminal
     exec stty echo -echonl <@stdin

     return $password
}

proc chooseMachine { } {
    puts "请选择机器\n1) 10.0.0.73 (codecenter-pre default);\n2) 10.0.2.153 (codecenter-prod);\n3) 10.0.2.154 (codecenter-prod);\na) 10.0.0.72 (codecenter-node-pre);\nb) 10.0.2.151 (codecenter-node-prod);\nc) 10.0.2.152 (codecenter-node-prod);";
    gets stdin choice;
    switch $choice {
        1 {
            return "10.0.0.73";
        }
        1 {
            return "10.0.2.153";
        }
        2 {
            return "10.0.2.154";
        }
        a {
            return "10.0.0.72";
        }
        b {
            return "10.0.2.151";
        }
        c {
            return "10.0.2.152";
        }
        default {
            return "10.0.0.73";
        }
    }
}

proc chooseJumper { IP } {
    switch $IP {
        "10.0.0.73" { 
            return "120.27.220.50";
        }
        "10.0.0.72" {
            return "120.27.220.50";
        }
        default {
            return "121.43.182.162";
        }
    }
}

proc chooseJumperAccount { JUMPER } {
    switch $JUMPER {
        "120.27.220.50" {
            return "root";
        }
        default {
            return "admin";
        }
    }
}

set IP [ chooseMachine ];		# TCL 语法的创建变量 + 子程序调用方式
set JUMPER [ chooseJumper $IP ];
set JUMPER_ACCOUNT [ chooseJumperAccount $JUMPER ];
set PASSWORD [ terminal:password:get "输入密码" ];
spawn ssh van.yzt@100.69.206.131;			# expect 语法，生成一个子程序用于接着后续的“expect”语句交互
expect "*password:";
send "$PASSWORD\r";
expect "*$*";
send "sudo su admin\r";
expect "*password*";
send "$PASSWORD\r";
expect "*$*";
send "ssh $JUMPER_ACCOUNT@$JUMPER\r";
expect "*]*";
send "ssh root@$IP\r";
set timeout 1;
expect {
    "*yes/no" { send "yes\r";exp_continue; }
    "*password:" { send "Codecenter123\r"; }
}
interact;			# 告知子程序，停留在机器上不结束自己，提供给用户后续的操作


```





​	同样的，利用expect，还可以做其他的操作，例如打包源码->上传代码到服务器->在服务器上编译、构建、部署等。