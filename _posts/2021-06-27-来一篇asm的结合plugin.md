---
layout: post
title: 来一篇ASM结合Plugin的
subtitle: 这个是初级的ASM，未完待续。。。
image: /img/lifedoc/jijiji.jpg
tags: [gradle学习]
---

### asm插桩

通过一个小的插桩功能完成对asm插桩配合gradle的完整流程。统计方法的执行时间，这个需要保存变量，去记录时间。为了实现这么一个看起来很小的功能，我几乎搞了两天。只能说，纸上得来终觉浅，觉知此事需躬行。

### 首先是在gradle里面增加asm相关

![](https://raw.githubusercontent.com/Pjex/images/master/20210627141048.png)

然后在原来的plugin里面增加对应transform的注册，下图表示了再对应的点位插桩，我们一般插桩的时候都是在生成了对应的class文件之后，然后我们就对我们感兴趣的点位或者特定的class进行插桩。

![](https://raw.githubusercontent.com/Pjex/images/master/20210627141450.png)

注册对应transform:
```kotlin
(android as AppExtension).registerTransform(XPJTransform())
```

### transform相关

transform文件：
```kotlin
package com.xpj.firstgradlelibrary.asm

import com.android.build.api.transform.*
import com.android.build.gradle.internal.pipeline.TransformManager
import com.xpj.firstgradlelibrary.XPJ_TRANSFORM
import org.apache.commons.io.FileUtils
import org.objectweb.asm.ClassReader
import org.objectweb.asm.ClassWriter
import java.io.File
import java.io.FileOutputStream
import java.io.IOException
import java.lang.Exception
import java.nio.file.*
import java.nio.file.attribute.BasicFileAttributes

/**
 * author : xpj
 * date : 6/26/21 8:06 PM
 * description :
 */
class XPJTransform : Transform() {
    override fun getName(): String {
        return XPJ_TRANSFORM
    }

    override fun getInputTypes(): MutableSet<QualifiedContent.ContentType> {
        return TransformManager.CONTENT_CLASS
    }

    override fun getScopes(): MutableSet<in QualifiedContent.Scope> {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    override fun isIncremental(): Boolean {
        return false
    }

    override fun transform(transformInvocation: TransformInvocation?) {
        super.transform(transformInvocation)
        transformInvocation?.let {
            try {
                println("XUEXI 进入transform里面了，准备处理了啊")
                innerTransform(transformInvocation.inputs, transformInvocation.outputProvider)
            } catch (e: Exception) {
                println("XUEXI 异常的是 ${e.message} ")
                e.printStackTrace()
            }
        }
    }

    private fun innerTransform(
        inputs: Collection<TransformInput>,
        outProvider: TransformOutputProvider
    ) {
        if (!isIncremental) {
            println("XUEXI 增量删除，删掉ß")
            outProvider.deleteAll()
        }

        inputs.forEach {
            it.directoryInputs.forEach { dic ->
                handleDic(dic, outProvider)
            }

            it.jarInputs.forEach { jar ->
                handleJar(jar, outProvider)
            }
        }
    }

    // fixme 这里没有任何处理是可以正常运行的
    private fun handleDic(dicInput: DirectoryInput, outProvider: TransformOutputProvider) {
        println("XUEXI 处理dic ${dicInput.name}")
        Files.walkFileTree(Paths.get(dicInput.file.toURI()), object : FileVisitor<Path> {
            override fun preVisitDirectory(
                dir: Path?,
                attrs: BasicFileAttributes?
            ): FileVisitResult {
//                println("XUEXI 11111111111111 pre visit directory $dir")
                return FileVisitResult.CONTINUE
            }

            override fun visitFile(filePath: Path?, attrs: BasicFileAttributes?): FileVisitResult {
                println("XUEXI 22222222222222 visit File visit directory $filePath")
                filePath?.apply {
                    val file = toFile()
                    val fName = file.name
                    if (filterClass(fName)) {
                        println("XUEXI 这里要修改了 aaaaa 红红火火恍恍惚惚")
                        // 这里怀疑readBytes和要求的byte[]是否相同，实际上是相同的主要原因是自己的蠢笨。
                        val classReader = ClassReader(file.readBytes())
                        //传入COMPUTE_MAXS  ASM会自动计算本地变量表和操作数栈
                        val classWriter = ClassWriter(classReader, ClassWriter.COMPUTE_MAXS)
                        //创建类访问器   并交给它去处理
                        val classVisitor = XPJClassVisitor(classWriter)
                        classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES)
                        val code = classWriter.toByteArray()
                        val fos =
                            FileOutputStream(file.parentFile.absolutePath + File.separator + fName)
                        fos.write(code)
                        fos.close()
                        println(
                            "XUEXI zheli这里写入了吗  草丛嗷嗷哦啊哦啊哦 " +
                                    " ${file.parentFile.absolutePath + File.separator + fName}  " +
                                    "name is ->>> $name   fname --->>>> $fName"
                        )
                    }
                }
                return FileVisitResult.CONTINUE
            }

            override fun visitFileFailed(file: Path?, exc: IOException?): FileVisitResult {
                return FileVisitResult.CONTINUE
            }

            override fun postVisitDirectory(dir: Path?, exc: IOException?): FileVisitResult {
                return FileVisitResult.CONTINUE
            }

        })
        val dest = outProvider.getContentLocation(
            dicInput.name, dicInput.contentTypes,
            dicInput.scopes, Format.DIRECTORY
        )
        println(
            "XUEXI ---->>>> 写入文件 原来的 ${dicInput.name} 目标的 : ${dest.name}"
                    + " types ： ${dicInput.contentTypes}   scope ： ${dicInput.scopes}"
        )
        FileUtils.copyDirectory(dicInput.file, dest)
    }

    // fixme 这里仅仅处理文件有问题，需要把jar原封不动的也复制过去
    private fun handleJar(jarInput: JarInput, outProvider: TransformOutputProvider) {
//        println("XUEXI 处理jar ${jarInput.name}")
    // 首先是怀疑这里dest有误，实际上，没有问题，是自己的问题，jar包这里仅仅是复制过去没有做任何操作
        val dest = outProvider.getContentLocation(
            jarInput.name, jarInput.contentTypes,
            jarInput.scopes, Format.JAR
        )
//        println("XUEXI ---->>>> JAR JAR 写入文件 原来的 ${jarInput.name} 目标的 : ${dest.name}"
//                +  " types ： ${jarInput.contentTypes}   scope ： ${jarInput.scopes}")
        FileUtils.copyFile(jarInput.file, dest)
    }

    private fun filterClass(className: String): Boolean {
        return (className.endsWith(".class") && !className.startsWith("R\$")
                && "R.class" != className && "BuildConfig.class" != className)
    }

    private fun filterMainActivity(className: String): Boolean {
        return "MainActivity.class" == className
    }
}
```

这里的核心代码是，一个是对不同的情况下的文件处理，一个是处理对应的文件夹，一个是处理对应的jar，因为我们把output provider都删除了是用的非增量的方式，因此每次都要把文件复制到dest否则会提示没有对应的类。另外这里用到了java.nio.Files 的 walkFileTree 去遍历文件，仅仅对那些非R 非 BuildConfig文件去操作，如果是我们需要操作的文件那么我们就去调用我的method visitor去访问并修改对应的入口点和出口点，进而调用method的visitor去修改。

### class visitor用法

method visitor核心代码：
```kotlin
    private var className: String? = null

    override fun visit(
        version: Int,
        access: Int,
        name: String?,
        signature: String?,
        superName: String?,
        interfaces: Array<out String>?
    ) {
        className = name
        super.visit(version, access, name, signature, superName, interfaces)
    }

    override fun visitMethod(
        access: Int,
        name: String?,
        descriptor: String?,
        signature: String?,
        exceptions: Array<out String>?
    ): MethodVisitor {
        val mv = super.visitMethod(access, name, descriptor, signature, exceptions)
        println("XUEXI visitMethod class is $className ")
        name ?: return mv
        descriptor ?: return mv
        return if (true) {
            XPJMethodVisitorAdapter(mv, access, "$className/$name", descriptor)
        } else {
            mv
        }
    }
```

在visit核心位置去设置对应的className，然后在visitMethod去替换为我们自己的对应的method visitor。

### method visitor 用法

method visitor代码如下：
```kotlin
package com.xpj.firstgradlelibrary.asm


import org.objectweb.asm.Label
import org.objectweb.asm.MethodVisitor
import org.objectweb.asm.Opcodes
import org.objectweb.asm.commons.AdviceAdapter


/**
 * author : xpj
 * date : 6/27/21 9:25 AM
 * description :
 */
class XPJMethodVisitorAdapter(visitor: MethodVisitor, access: Int, name: String, desc: String) :
    AdviceAdapter(
        Opcodes.ASM6,
        visitor, access, name, desc
    ) {
    override fun visitCode() {
        println("XUEXI 限制了，刚刚仅仅打印log可以   visitCode !!!!!!!!!!!!!!!!!!!!!!!!!++++++ $mv")
        // fixme 这个log是为了验证这里加东西是没有问题的，成功了，他也就功成名就了，最根本原因是插入的第一个位置不对
//        mv.visitLdcInsn("XUEXI")
//        mv.visitLdcInsn("im in on resume. 擦哦OA从嗷嗷嗷哦肯定是附近可拉伸法打卡机克里斯多夫-------$name-----")
//        mv.visitMethodInsn(
//            Opcodes.INVOKESTATIC,
//            "android/util/Log",
//            "i",
//            "(Ljava/lang/String;Ljava/lang/String;)I",
//            false
//        )
//        mv.visitInsn(Opcodes.POP)

        mv.visitMethodInsn(
            Opcodes.INVOKESTATIC,
            "java/lang/System",
            "currentTimeMillis",
            "()J",
            false
        );
        mv.visitVarInsn(Opcodes.LSTORE, 3)
        val label1 =  Label()
        mv.visitLabel(label1)

        super.visitCode()
    }

    override fun onMethodExit(opcode: Int) {
        super.onMethodExit(opcode)
        println("XUEXI onMethodExit !!!!!!!!!!!!!!!! visit insn p0 is $opcode v2 +++++++++____$mv _____---------")
        if (false) {
            return super.onMethodExit(opcode)
        }
        getMessageEndCostTime(mv, name)
    }

    private fun getMessageEndCostTime(methodVisitor: MethodVisitor, name: String) {
        methodVisitor.visitMethodInsn(
            Opcodes.INVOKESTATIC,
            "java/lang/System",
            "currentTimeMillis",
            "()J",
            false
        );
        methodVisitor.visitVarInsn(Opcodes.LLOAD, 3)
        methodVisitor.visitInsn(Opcodes.LSUB)
        methodVisitor.visitVarInsn(Opcodes.LSTORE, 4)
        val label2 = Label();
        methodVisitor.visitLabel(label2)
        println("XUEXI -----    gggggggggggggg ->>>>>>>>>>>>>>>>    lavel2 is ${label2.info}")
        methodVisitor.visitLdcInsn("XUEXI")
        methodVisitor.visitTypeInsn(Opcodes.NEW, "java/lang/StringBuilder");
        methodVisitor.visitInsn(Opcodes.DUP);
        methodVisitor.visitMethodInsn(
            Opcodes.INVOKESPECIAL,
            "java/lang/StringBuilder",
            "<init>",
            "()V",
            false
        )
        methodVisitor.visitLdcInsn(name + "消耗的ORRRRRRRRRRRROOOOOO时间:")
        methodVisitor.visitMethodInsn(
            Opcodes.INVOKEVIRTUAL,
            "java/lang/StringBuilder",
            "append",
            "(Ljava/lang/String;)Ljava/lang/StringBuilder;",
            false
        );
        methodVisitor.visitVarInsn(Opcodes.LLOAD, 4)
        methodVisitor.visitMethodInsn(
            Opcodes.INVOKEVIRTUAL,
            "java/lang/StringBuilder",
            "append",
            "(J)Ljava/lang/StringBuilder;",
            false
        );
        methodVisitor.visitMethodInsn(
            Opcodes.INVOKEVIRTUAL,
            "java/lang/StringBuilder",
            "toString",
            "()Ljava/lang/String;",
            false
        )
        methodVisitor.visitMethodInsn(
            Opcodes.INVOKESTATIC,
            "android/util/Log",
            "e",
            "(Ljava/lang/String;Ljava/lang/String;)I",
            false
        )
        methodVisitor.visitInsn(Opcodes.POP)
        val label3 = Label()
        methodVisitor.visitLabel(label3)
        println("XUEXI ------>>>>>>>>>>>>>>>>lavel3 is ${label3.toString()}")
    }
}
```

method visitor虽然最终可以使用了，但是还有遗留问题，这个问题也是这次一直迟迟弄不出来的核心原因，当然经过这次之后也学会了看一些log了，第一个没有对应的入口类，这个是因为没有正确写回导致，下面是因为写入到不正确位置导致，这里排除这个问题可以使用asm plugin查看对应的代码，会发现text1 占用了 store 2的位置，而我们在method visitor里面首先将第一个时间戳放在2导致用的时候类型出错。

```java
> Task :app:dexBuilderDebug
/Users/xpj/AndroidStudioProjects/MyGrowthPath/app/build/intermediates/transforms/XPJTransform/debug/39/com/xpj/mygrowthpath/SecondActivity.class: D8: Cannot constrain type: @Nullable android.widget.TextView {} for value: v9(text1) by constraint: LONG
org.gradle.workers.WorkerExecutionException: There was a failure while executing work items
        at org.gradle.workers.internal.DefaultWorkerExecutor.workerExecutionException(DefaultWorkerExecutor.java:264)
```

关于查看asm plugin则是右击对应文件，点击asm那个相关的，具体可如图，主要查看ASMified里面的内容。通过这里可以看到已经被占用。

![](https://raw.githubusercontent.com/Pjex/images/master/20210627144250.png)

这时候就会提示上面的编译问题。

#### 问题写入类错误

```kotlin
06-27 12:46:38.847 20754 20754 E AndroidRuntime: FATAL EXCEPTION: main
06-27 12:46:38.847 20754 20754 E AndroidRuntime: Process: com.xpj.mygrowthpath, PID: 20754
06-27 12:46:38.847 20754 20754 E AndroidRuntime: java.lang.VerifyError: Verifier rejected class com.xpj.mygrowthpath.MainActivity: void com.xpj.mygrowthpath.MainActivity.onCreate(android.os.Bundle) failed to verify: void com.xpj.mygrowthpath.MainActivity.onCreate(android.os.Bundle): [0xB] register v0 has type Long (Low Half) but expected Precise Reference: android.os.Bundle (declaration of 'com.xpj.mygrowthpath.MainActivity' appears in /data/app/com.xpj.mygrowthpath-2/base.apk)
06-27 12:46:38.847 20754 20754 E AndroidRuntime: 	at java.lang.Class.newInstance(Native Method)
06-27 12:46:38.847 20754 20754 E AndroidRuntime: 	at android.app.Instrumentation.newActivity(Instrumentation.java:1078)
06-27 12:46:38.847 20754 20754 E AndroidRuntime: 	at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2557)
06-27 12:46:38.847 20754 20754 E AndroidRuntime: 	at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2726)
06-27 12:46:38.847 20754 20754 E AndroidRuntime: 	at android.app.ActivityThread.-wrap12(ActivityThread.java)
```

当我们写入的东西不符合标准的时候，就会被Java虚拟机的校验拒之门外，这时候就要考虑是我们对应的插桩方法哪里出问题了，就像我们之前的对应store和load位置有问题了。

[使用ASM完成编译时插桩](https://chsmy.github.io/2019/09/28/architecture/%E4%BD%BF%E7%94%A8ASM%E5%AE%8C%E6%88%90%E7%BC%96%E8%AF%91%E6%97%B6%E6%8F%92%E6%A1%A9/)

[強大！ASM 插樁實現 Android 端無埋點性能監控](https://www.gushiciku.cn/pl/gz8d/zh-hk)

[Android函数插桩](https://www.jianshu.com/p/16ed4d233fd1)

### 总结

经过本次简单用法，总结了几种需要注意的地方。

1. 这次我全程使用的kotlin去写的对应gradle；
2. 使用println方法输出对应的关键log，例如对应的transform或者对应的method class visitor里面具体插桩的时候的执行是否达到预期，例如打印对应的类信息，对应的方法信息等；
3. 使用asm plugin查看对应的目标文件的asm形式，有时候我们的store或者啥的冲突就是这里导致的；
4. 查看log文件谷歌或者百度核心原因；
5. 最重要的是：**经过这次的经历，提炼了一个关键方法论，就是凡事从最简单的开始，在不知道的时候贸然复制或者抄代码没有意义，这次就是经历了，第一添加空的transform文件并不做修改去运行没有问题；第二在transform的关键方法里过滤类和某个方法去插桩；第三在插入地里面添加一个log打印；第四则是由有实际应用的统计时间或者打点上报等**
6. 后期可以使用transform或者对应的plugin去扩展更多的应用场景和方法，例如检查是否有非法的操作或者调用，另外asm也大有可为可以结合注解去做一些自定义的操作，例如用注解去标记或者提取信息，避免全量去操作某些东西。

前途是光明的，随时准备，勇往直前，哈哈哈！

