---
layout: post
title: android自定义gradle 2
tags: [gradle学习]
image: /img/hello_world.jpeg
---

### 外部传入参数

今天继续学习gradle的其他用法，其中之一是增加参数，增加参数方便我们根据不同的情况去做个性化工作。

plugin文件中的改法:
```kotlin
      //定义参数 在build.gradle设置参数 XPJTestParams{ name = "" } XPJParams 这个类需要时open的类
        project.extensions.create("XPJTestParams", XPJParams::class.java)
        //获取参数
        // fixme 这里取值不对是因为先把自己还没有assemble的依赖里，这里的取值在赋值之前
        val name = project.extensions.findByType(XPJParams::class.java)?.name

       task.doLast {
                // fixme 这里可以是因为已经在上一个赋值之后了
                println("在dolast的时候再取值，这时候应该ok了")
                File("${project.projectDir.path}/XPJPluginStudy.txt").apply {
                    writeText(
                        "Hello ${project.extensions.findByType(XPJParams::class.java)?.name}! AAAA BBB CCC\nPrinted at: ${
                            SimpleDateFormat("HH:mm:ss").format(
                                Date()
                            )
                        }"
                    )
                }
```

然后在调用的app build文件中增加对应的调用
```groovy
// fixme 这里就是按照顺序去执行的，在android的task之后
// 注意这里的名字是和添加到extensions里面的第一个参数是一致的，也就是key
XPJTestParams {
    println("这里创建的extensions实际上是一个groovy的闭包。 什么时候执行这个设置方法，是不是之后才高的呀")
    name = "啦啦啦，德玛西亚"
}
```

### 增量编译

首先是创建一个对应的task，如下
```kotlin
open class XPJFirstTask : DefaultTask(){
    @Input
    var inputName:String? = null
    @OutputFile
    var file: File? = null
    @TaskAction
    fun generateFile(){
        println("我在自定义task里面执行")
        file?.apply {
            writeText("Hello $inputName!\nPrinted at: ${SimpleDateFormat("HH:mm:ss").format(Date())}")
        }
    }
}
```

然后在我们的plugin文件中增加对应代码，编译几次，发现up to date。
```kotlin
//注册Task
        project.tasks.register(XPJ_PLUGIN_NAME_V2, XPJFirstTask::class.java) {
            it.group = XPJ_PLUGIN_TASKS
            //设置输入
            it.inputName = project.extensions.findByType(XPJParams::class.java)?.name
            //设置输出
            // fixme 这里的输出文件需要与之前的文件不同，不然通知执行两个task的时候由于第一个修改了这个文件，
            // fixme 那么第二个task探测到变了就会再次执行task了
            it.file = File("${project.projectDir.path}/$XPJ_PLUGIN_GENERATE_FILE_V2")
        }

        //将MyFirstPlugin添加到构建树
        val android = project.extensions.findByType(BaseExtension::class.java)
        (android as AppExtension).applicationVariants.all {
            //将MyFirstPlugin task添加到assemble task前
            //assemble依赖MyFirstPlugin的意思是说assemble运行前先运行MyFirstPlugin
            it.assembleProvider.dependsOn(XPJ_PLUGIN_NAME_V2)
            // fixme 连续增加两个不同的task
            it.assembleProvider.dependsOn(XPJ_PLUGIN_NAME)
        }
```

这里主要用了增量编译的几个注解api更多的需要查阅gradle官方。

