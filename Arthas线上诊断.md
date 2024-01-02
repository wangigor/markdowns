# Arthas线上诊断

> [Arthas官方文档](https://arthas.aliyun.com/doc/)介绍的很详细，这里只做使用案例分享。
>
> `Arthas` 支持 JDK 6+，支持 Linux/Mac/Windows，采用命令行交互模式。

## 安装

### 外网环境安装、启动

```bash
# 下载
curl -O https://arthas.aliyun.com/arthas-boot.jar
# 进入交互
java -jar arthas-boot.jar
```

- 如果下载速度慢，可以使用aliyun镜像

  ```bash
  java -jar arthas-boot.jar --repo-mirror aliyun --use-http
  ```

- 也可以使用`as.sh`方式

  ```bash
  curl -L https://arthas.aliyun.com/install.sh | sh
  ```

  上述命令会下载启动脚本文件 `as.sh` 到当前目录，你可以放在任何地方或将其加入到 `$PATH` 中。

  直接在 shell 下面执行`./as.sh`，就会进入交互界面。

**注意：外网环境需要主机具有外网环境「启动时需要下载相关依赖」。否则启动失败。**

### 离线环境安装、启动

> 离线环境只需要下载全量包。
>
> 其他操作一样。

[Arthas最新全量包下载地址](https://arthas.aliyun.com/download/latest_version?mirror=aliyun) 或者 [github releases页面下载bin.zip](https://github.com/alibaba/arthas/releases)

解压后使用`./as.sh`或者`java -jar arthas-boot.jar`启动。



### 总部云环境

> 需要使用离线方式。68的nginx里放了一个全量包。

```bash
# 进入pod控制台，下载全量包
wget --no-check-certificate https://10.174.104.68:10443/static/file/arthas-bin.zip
```



## 使用案例

### jad查看运行代码

> 假如你在怀疑自己的代码上没上生产「我咋发了，但是好像没生效」。
>
> 代码没合并？tag打错了？pod启动失败？
>
> 这些问题似乎排查起来有些麻烦，那就**直接进jvm看源码。**

默认情况下，反编译结果里会带有`ClassLoader`信息，通过`--source-only`选项，可以只打印源代码。

```bash
jad --source-only 类的全路径
```

![image-20240102172257035](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240102172257035.png)

### trace代码慢在哪里

```bash
trace 类的全路径 方法名 
# 增加耗时过滤、调用一次退出
trace 类的全路径 方法名 '#cost > 10' -n 1
```

- 默认情况下，trace 不会包含 jdk 里的函数调用，如果希望 trace jdk 里的函数，需要显式设置`--skipJDKMethod false`。
- 如果方法调用的次数很多，那么可以用`-n`参数指定捕捉结果的次数。比如下面的例子里，捕捉到一次调用就退出命令。
- `'#cost > 10'`只会展示耗时大于 10ms 的调用路径，有助于在排查问题的时候，只关注异常情况

![image-20240102172658864](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240102172658864.png)

### redefine更新代码

> 假如你有一个线上的非生产小bug，比如灰度环境的bug。
>
> 而你自信修改，提了合并请求，打了tag，cicd，复测还在报错。
>
> 你面临到了新的崩溃：我要是不那么自信就好了。我或许应该加上try-catch捕获住这个异常看看他到底什么样子。但是发布流程太漫长了。
>
> 此时，可以采用这样的方式，因为这一套操作流程只要几秒钟。

原文件如下。

```java
package com.cmos.retail.web;

import com.cmos.retail.beans.common.dataformat.Result;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/echo")
public class EchoController {

    @GetMapping("/{params}")
    public Result fakeUrlHandler(@PathVariable String params) {
        return Result.of("hello " + params);
    }
}
```

「需求」可以简单抽象为`修改返回值hello为hello11111`。

现在使用`jad`查看jvm中的的class反编译结果与java文件差不多一致。

![image-20240102170246941](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240102170246941.png)

- 先把jad的反编译文件输出到临时文件中

  ```bash
  jad --source-only com.cmos.retail.web.EchoController > /tmp/redefineFiles/EchoComtroller.java
  ```

- 打开一个新的终端，修改临时文件。

  ![image-20240102170745123](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240102170745123.png)

- 查询原类的类加载器

  ```bash
  sc -d com.cmos.retail.web.EchoController
  ```

  ```log
  class-info        com.cmos.retail.web.EchoController
   code-source       /Users/wangke/IdeaProjects/AsiaInfo/onlinehandle-new-retail/onlinehandle-new-retail-web/target/class
                     es/
   name              com.cmos.retail.web.EchoController
   isInterface       false
   isAnnotation      false
   isEnum            false
   isAnonymousClass  false
   isArray           false
   isLocalClass      false
   isMemberClass     false
   isPrimitive       false
   isSynthetic       false
   simple-name       EchoController
   modifier          public
   annotation        org.springframework.web.bind.annotation.RestController,org.springframework.web.bind.annotation.Reque
                     stMapping
   interfaces
   super-class       +-java.lang.Object
   class-loader      +-org.springframework.boot.devtools.restart.classloader.RestartClassLoader@25968570
                       +-sun.misc.Launcher$AppClassLoader@18b4aac2
                         +-sun.misc.Launcher$ExtClassLoader@2a742aa2
   classLoaderHash   25968570
  ```

- 编译`.java`成`.class`

  ```bash
  mc -c 25968570 /tmp/redefineFiles/EchoController.java -d /tmp
  ```

  ![image-20240102171058254](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240102171058254.png)

- redefine热更新

  ```bash
  redefine /tmp/com/cmos/retail/web/EchoController.class
  ```

  ![image-20240102171306792](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240102171306792.png)

> 这样的方式可以绕过发布直接修改jvm中已加载类，可能导致线上运行代码与发布代码版本不一致。
>
> **慎用。**

