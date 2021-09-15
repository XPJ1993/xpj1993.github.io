---
layout: post
title: 使用 github 作为远程 maven 仓库
subtitle: github 怎样用之 做仓库
image: /img/lifedoc/jijiji.jpg
tags: [Maven]
---

### 操作步骤

1. 打包 aar 到本地

```groovy
uploadArchives {
    repositories {
        mavenDeployer {
            // 引用路径为： com.xpj:XPJFirstPlugin:1.0.0
            pom.groupId = 'com.xpj'
            pom.artifactId = "XPJFirstPlugin"
            pom.version = '1.0.2'
            //上传到本地
            repository(url: uri('../studyPlugin'))
        }
    }
}
```

2. 在 github 创建一个接收 aar 的仓库，我这里是 mavenAAR

3. 上传打包成功的 aar 到 github ，注意这里需要加上包的完整路径如我的就是 com/xpj/XPJFirstPlugin

4. 在使用 aar 的地方添加 maven 依赖

```groovy
maven {
    url "https://raw.githubusercontent.com/XPJ1993/mavenAAR/master"
}

// 使用
classpath "com.xpj:XPJFirstPlugin:1.0.2"
```

5. 解决因为众所周知的原因带来的连不上 raw.githubusercontent.com 

{: .box-note}
**Note:** 查询真实IP , 在https://www.ipaddress.com/查询raw.githubusercontent.com的真实IP。

通过修改hosts解决此问题:

{: .box-note}
**Note:** 199.232.28.133 raw.githubusercontent.com

