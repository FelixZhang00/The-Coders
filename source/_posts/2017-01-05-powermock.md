---
title: 【2017-01-05-powermock】
date: 2017-01-05 01:33:50
tags: Java
---

## PowerMock

### [PowerMock github project](https://github.com/powermock/powermock)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;PowerMock是一个强大的测试框架，它集成了Mockito、EasyMock等测试框架，在基于这些框架的基础上，利用修改字节码的手段mock静态方法、私有方法、初始化方法，是一种对过去熟悉的测试框架的增强扩张。

---

### 使用

1. 引入PowerMock的jar包，需要注意不同版本的jar包依赖的JUnit版本；
2. 建立测试类；
3. 利用PowerMock写单测；

---

### 实例

#### 被测试接口：UserService

```java
package com.van.service;

/**
 * 被测试类
 * Created by Wuha on 2017/1/8.
 */
public interface UserService {
    String thePublicMethod();
}
```

##### 被测试类实现：UserServiceImpl

```java
package com.van.service.impl;

import com.van.UserDAO;
import com.van.service.UserService;
import com.van.util.UserUtil;
import org.springframework.beans.factory.annotation.Autowired;

import java.io.File;

/**
 * Created by Wuha on 2017/1/8.
 */
public class UserServiceImpl implements UserService {
    @Autowired
    private UserDAO userDAO;

    private Long XXXX_XXX_LONG;

    private String XXXX_XXXX_STRING;

    public UserServiceImpl() {
        this.XXXX_XXX_LONG = 0L;
        this.XXXX_XXXX_STRING = "";
    }

    public UserServiceImpl(Long XXXX_XXX_LONG, String XXXX_XXXX_STRING) {
        this.XXXX_XXX_LONG = XXXX_XXX_LONG;
        this.XXXX_XXXX_STRING = XXXX_XXXX_STRING;
    }

    public String thePublicMethod() {
        // some logic

        this.thePrivateMethod();

        userDAO.getUserById(XXXX_XXX_LONG);

        UserUtil.theStaticMethod();

        return XXXX_XXXX_STRING;
    }

    private void thePrivateMethod() {
        // some logic
        File file = new File(XXXX_XXXX_STRING);
        if (!file.exists()) {
            // do some thing
            throw new RuntimeException("file.exist() method mock failed.");
        }
    }
}
```

#### 被测试类调用的静态方法

```java
package com.van.util;

/**
 * Created by Wuha on 2017/1/8.
 */
public class UserUtil {
    public static void theStaticMethod() {
        // some logic
        throw new RuntimeException("static method mock failed.");
    }
}
```

#### 测试类

```java
package com.van.test;

import com.van.UserDAO;
import com.van.service.UserService;
import com.van.service.impl.UserServiceImpl;
import com.van.util.UserUtil;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.*;
import org.powermock.api.mockito.PowerMockito;
import org.powermock.core.classloader.annotations.PrepareOnlyThisForTest;
import org.powermock.modules.junit4.PowerMockRunner;

import java.io.File;

/**
 * Created by Wuha on 2017/1/8.
 */
@RunWith(PowerMockRunner.class)     // 测试运行于PowerMock测试框架下,加载特定对象、清除MockRepository等
@PrepareOnlyThisForTest({UserUtil.class, UserServiceImpl.class})    // 该在注解标明了需要用PowerMock字节码增强的类,如:mock静态方法、final域、私有方法等
public class UserServiceTest {
    private static final String EXPECTED_STRING = "Hello world.";

    private static final Long EXPECTED_LONG = -1L;

    @Mock                           // mock 一个UserDAO类
    private UserDAO uerDAO;

    @Mock
    private File mockFile;

    @Spy  // spy 一个类,这个注解产生的类,如果不去mock方法,那么在运行它的时候,会调用实际的方法,这点和mockObject.thenCallRealMethod()效果一样
    @InjectMocks    // 由于UserService利用了@Autowired注解,而没有public的setter方法,该注解可以帮助吧userDAO给set到service中
    private UserService userService = new UserServiceImpl(EXPECTED_LONG, EXPECTED_STRING);

    @Before
    public void setUp() throws Exception {
        /**
         * 对注解的对象进行语义逻辑处理
         */
        MockitoAnnotations.initMocks(this);
        /**
         * mock处理静态类
         */
        PowerMockito.mockStatic(UserUtil.class);

        /**
         * mock some method
         */
        PowerMockito.when(mockFile.exists()).thenReturn(true);
        /**
         * mock一个construct方法
         */
        PowerMockito.whenNew(File.class).withArguments(Mockito.anyString()).thenReturn(mockFile);
        /**
         * mock static method, the private method mock is the same
         */
        PowerMockito.doNothing().when(UserUtil.class, "theStaticMethod");
    }

    @Test
    public void testThePublicMethod() {
        Assert.assertEquals(EXPECTED_STRING, userService.thePublicMethod());
    }
}
```
---

### 测试结果

<center>
![PowerMock运行结果](/images/2017/01/08/powermock运行结果.png)
</center>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;按照代码中的逻辑，如果没被mock，则会抛出异常。


