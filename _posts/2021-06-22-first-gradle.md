---
layout: post
title: android自定义gradle 1
tags: [gradle学习]
image: /img/hello_world.jpeg
---

重新学习一下自定义gradle插件，搞Android开发的要想搞些骚操作免不了需要用到自定义gradle这个武器，例如资源整理，asm插桩，代码检测等等。

直入主题，开始说明步骤。

1. 创建一个测试Android 项目
2. 在改项目中创建一个Android Module，然后更改对应library的build.gradle文件，修改后内容如下
![](https://raw.githubusercontent.com/Pjex/images/master/20210622164741.png)
3. 然后就是集成最基本的在对应的src文件夹下创建一个kotlin文件写对应的实现，这里集成最基本的plugin类，具体实现如下

```java
package com.xpj.firstgradlelibrary

import com.android.build.gradle.AppExtension
import com.android.build.gradle.BaseExtension
import com.android.build.gradle.internal.tasks.factory.dependsOn
import org.gradle.api.Plugin
import org.gradle.api.Project
import java.io.File
import java.text.SimpleDateFormat
import java.util.*

/**
 * author : xpj
 * date : 6/18/21 11:13 AM
 * description :
 */
class XPJFirstPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        // todo 这里的方法是吧自己加到assemble之前的task，构建依赖树
        //将MyFirstPlugin添加到构建树
        val android = project.extensions.findByType(BaseExtension::class.java)
        (android as AppExtension).applicationVariants.all {
            //将MyFirstPlugin task添加到assemble task前
            //assemble依赖MyFirstPlugin的意思是说assemble运行前先运行MyFirstPlugin
            it.assembleProvider.dependsOn(XPJ_PLUGIN_NAME)
        }

        project.tasks.create(XPJ_PLUGIN_NAME) { task ->
            task.group = XPJ_PLUGIN_GROUP
            println("我在自定义plugin里面，改变之后的。")
            File("${project.projectDir.path}/IMOUT.txt").apply {
                writeText("Hello World! AAAA BBB CCC\nPrinted at: ${SimpleDateFormat("HH:mm:ss").format(Date())}")
            }
            // 这里因为dolast在执行的时候并没有生成文件
            task.doLast {
                File("${project.projectDir.path}/myFirstGeneratedFile.txt").apply {
                    writeText("Hello World! AAAA BBB CCC\nPrinted at: ${SimpleDateFormat("HH:mm:ss").format(Date())}")
                }
            }
        }
    }
}
```

4. 这里写好之后在对应的build.gradle下面执行我们的上传task这时候如果成功就会在本地或者maven生成对应的包，包含maven的形式，例如pom文件xml文件啥的
5. 在app工程里面引用，首先在project下面增加对应的引用和依赖
![](https://raw.githubusercontent.com/Pjex/images/master/20210622164819.png)
6. 在app 项目下的build引用
![](https://raw.githubusercontent.com/Pjex/images/master/20210622164857.png)
