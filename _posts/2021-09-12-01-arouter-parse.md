---
layout: post
title: Arouter 解析
subtitle: 组件化基础之 Arouter 解析
image: /img/lifedoc/jijiji.jpg
tags: [三方库]
---

三方库总目录：
[三方库](https://xpj1993.github.io/2021-09-10-02-third-libary/)

### Arouter 三方库解析

| 文章目录 | 代码解析 |
|---|---|
| 使用方式解析 | 通过在 build.gradle 以及 文件中具体使用解析怎么使用 arouter 去打开对应 path 的 activity |
| router 原理解析 | 实现 router 功能，需要实现那些 |
| Arouter 怎么做 | Arouter 的实现以及其技术选型与实现原理 |

#### 使用方式

build.gradle 文件
```kotlin
plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'com.xpj.pbversion.versiongplugin'
    id 'kotlin-kapt'
    id 'dagger.hilt.android.plugin'
    id 'com.alibaba.arouter' // 导入对应的 plugin 
}

// default config 中添加对应的 compile 配置
javaCompileOptions {
            annotationProcessorOptions {
                includeCompileClasspath true
                // **特别注意这里是 += 而不是直接等于** ，因为我这个项目还有 hilt 以及 room 相关的 注解处理器 所以这里需要 arguments += 否则运行崩溃，别的注解解析失败
                arguments += [AROUTER_MODULE_NAME: project.getName()]
            }
        }

// 增加对应的库的引用和注解引用
    api OtherLibraryKt.arouterApi
    kapt OtherLibraryKt.arouterCompile
    annotationProcessor OtherLibraryKt.arouterCompile        
```

```kotlin
// 目标 Activity 里面增加 route 注解， 注意这里的路径至少两层，且以 / 分割
@Route(path = "/wulala/la")
class PBArouterTest : FragmentActivity() {


// 调用方使用 Arouter 调用
ARouter.getInstance().build("/wulala/la").navigation()
```

通过对上面的代码解析，可以看到最简单的用法就是在 build 文件中增加 arouter 的 api 引用，增加 apt 引用，以及在 default config 中添加对应的配置，具体这个配置好像 hilt 和 room 理论上也应该有，但是 谷歌 可能已经默默做了，因此不需要我们去给她指定 module name。

接着在 代码中使用 router 注解去声明该类对应的 path 是哪个，然后在调用的地方去通过获取 arouter 的 instance build 对应的 postcard 然后 navigation 就可以了。

#### router 原理解析

经过上一节简单使用可以发现 arouter 用起来并不是很复杂，那么它的实现原理是什么呢，如果是我们自己写要怎么写从哪里入手去做一个 router 框架。

首先一个 router 框架需要一个调度中心，在别人调用的时候可以找到对应的目标并且可以启动目标去完成功能， router 本意不就是寻址和导航吗。

要想实现这个框架，那么我们可以想一下都需要哪些步骤，


| 框架步骤 | 实现方式 |
|---|---|
| 目标对象收集 | 1. 使用反射获取实现某个特定接口的类，然后收集到对应的 map 中，这个接口可以提供一个接口就是返回对应的唯一 name，调用的时候根据这个名字去寻址； 2. 善用注解，使用生命周期为 RetentionPolicy.CLASS （这种注解信息会保存在类文件中，但是运行时会去掉这个注解）的注解去做，注解中定义他的 value 为唯一区别值，使用 apt 技术去解析对应的被注解的类 |
| 目标对象存储 | 可以使用 map 去存起来目标对象，这里 key 是 String 唯一值，value 为对应的 class |
| 目标对象使用 | 因为 router 需要不仅仅跳转到 activity 也需要 fragment 或者 service，可以根据他的类信息去判断具体的对象，然后在做跳转 |
| 使用方式 | 做一个单例去对这些已经收集好的目标做寻址，找到就导航过去 |
| 使用参数 | 这个可以定义一个序列化的数据作为参数，通过解析这个参数赋值给 intent 然后再去打开对应目标的时候将参数给到他们 |
| content1 | content2 |

#### arouter 是怎么做

##### 目标收集

arouter 是选择的注解加 apt 去收集信息的，具体地可以看他的 router processor 代码：

具体地，可以查看 arouter-compiler 代码:

parseRoutes ->

```java
private Map<String, Set<RouteMeta>> groupMap = new HashMap<>(); // ModuleName and routeMeta.
    private Map<String, String> rootMap = new TreeMap<>(); 

    private void parseRoutes(Set<? extends Element> routeElements) throws IOException {
        if (CollectionUtils.isNotEmpty(routeElements)) {
            // prepare the type an so on.

            logger.info(">>> Found routes, size is " + routeElements.size() + " <<<");

            rootMap.clear();
            // 1. 首先是收集对应的 type mirror 这个是 Java 的定义
            TypeMirror type_Activity = elementUtils.getTypeElement(ACTIVITY).asType();
            TypeMirror type_Service = elementUtils.getTypeElement(SERVICE).asType();
            TypeMirror fragmentTm = elementUtils.getTypeElement(FRAGMENT).asType();
            TypeMirror fragmentTmV4 = elementUtils.getTypeElement(Consts.FRAGMENT_V4).asType();

            // Interface of ARouter
            // 2. 收集对应的 element 这里是 arouter 定义的类
            TypeElement type_IRouteGroup = elementUtils.getTypeElement(IROUTE_GROUP);
            TypeElement type_IProviderGroup = elementUtils.getTypeElement(IPROVIDER_GROUP);
            ClassName routeMetaCn = ClassName.get(RouteMeta.class);
            ClassName routeTypeCn = ClassName.get(RouteType.class);

            /*
               3. 创建对应的 map 类型如下
               Build input type, format as :
               ```Map<String, Class<? extends IRouteGroup>>```
             */
            ParameterizedTypeName inputMapTypeOfRoot = ParameterizedTypeName.get(
                    ClassName.get(Map.class),
                    ClassName.get(String.class),
                    ParameterizedTypeName.get(
                            ClassName.get(Class.class),
                            WildcardTypeName.subtypeOf(ClassName.get(type_IRouteGroup))
                    )
            );

            /*
              4. 同上，创建对应的 map
              ```Map<String, RouteMeta>```
             */
            ParameterizedTypeName inputMapTypeOfGroup = ParameterizedTypeName.get(
                    ClassName.get(Map.class),
                    ClassName.get(String.class),
                    ClassName.get(RouteMeta.class)
            );

            /*
              5. 生成对应的 spec 分别对应 routes 别名 atlas providers 等，这里是使用 的 JavaOpt 代码生成对应的代码 ParameterSpec 这些是 JavaOpt 对应的类
              Build input param name.
             */
            ParameterSpec rootParamSpec = ParameterSpec.builder(inputMapTypeOfRoot, "routes").build();
            ParameterSpec groupParamSpec = ParameterSpec.builder(inputMapTypeOfGroup, "atlas").build();
            ParameterSpec providerParamSpec = ParameterSpec.builder(inputMapTypeOfGroup, "providers").build();  // Ps. its param type same as groupParamSpec!

            /*
              6. 创建 laodinto 方法的描述
              Build method : 'loadInto'
             */
            MethodSpec.Builder loadIntoMethodOfRootBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                    .addAnnotation(Override.class)
                    .addModifiers(PUBLIC)
                    .addParameter(rootParamSpec);

            //  Follow a sequence, find out metas of group first, generate java file, then statistics them as root.
            for (Element element : routeElements) {
                TypeMirror tm = element.asType();
                Route route = element.getAnnotation(Route.class);
                RouteMeta routeMeta;

                // 7. 收集 Activity or Fragment
                if (types.isSubtype(tm, type_Activity) || types.isSubtype(tm, fragmentTm) || types.isSubtype(tm, fragmentTmV4)) {
                    // Get all fields annotation by @Autowired
                    Map<String, Integer> paramsType = new HashMap<>();
                    Map<String, Autowired> injectConfig = new HashMap<>();
                    // 7.1 对使用 Autowired 的变量进行获取
                    injectParamCollector(element, paramsType, injectConfig);

                    if (types.isSubtype(tm, type_Activity)) {
                        // Activity
                        logger.info(">>> Found activity route: " + tm.toString() + " <<<");
                        routeMeta = new RouteMeta(route, element, RouteType.ACTIVITY, paramsType);
                    } else {
                        // Fragment
                        logger.info(">>> Found fragment route: " + tm.toString() + " <<<");
                        routeMeta = new RouteMeta(route, element, RouteType.parse(FRAGMENT), paramsType);
                    }

                    // 7.2 设置进去 7.1 收集的数据
                    routeMeta.setInjectConfig(injectConfig);
                } else if (types.isSubtype(tm, iProvider)) {         // IProvider 8. 收集 IProvider 相关的实现
                    logger.info(">>> Found provider route: " + tm.toString() + " <<<");
                    routeMeta = new RouteMeta(route, element, RouteType.PROVIDER, null);
                } else if (types.isSubtype(tm, type_Service)) {           // Service 9. 收集 Service 相关的实现
                    logger.info(">>> Found service route: " + tm.toString() + " <<<");
                    routeMeta = new RouteMeta(route, element, RouteType.parse(SERVICE), null);
                } else {
                    throw new RuntimeException("The @Route is marked on unsupported class, look at [" + tm.toString() + "].");
                }

                // 10. 对收集到的信息使用 group 进行归类，这个 group 对应的是定义的 path 其中 path 必须 / 开头且不能少于 / 一个的原因就在这里，因为这里默认第一个为 group 类别
                categories(routeMeta);
            }

            MethodSpec.Builder loadIntoMethodOfProviderBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                    .addAnnotation(Override.class)
                    .addModifiers(PUBLIC)
                    .addParameter(providerParamSpec);

            Map<String, List<RouteDoc>> docSource = new HashMap<>();

            // 11. 通过 JavaOpt 生成对应的类 Start generate java source, structure is divided into upper and lower levels, used for demand initialization.
            for (Map.Entry<String, Set<RouteMeta>> entry : groupMap.entrySet()) {
                String groupName = entry.getKey();

                MethodSpec.Builder loadIntoMethodOfGroupBuilder = MethodSpec.methodBuilder(METHOD_LOAD_INTO)
                        .addAnnotation(Override.class)
                        .addModifiers(PUBLIC)
                        .addParameter(groupParamSpec);

                List<RouteDoc> routeDocList = new ArrayList<>();

                // Build group method body
                Set<RouteMeta> groupData = entry.getValue();
                for (RouteMeta routeMeta : groupData) {
                    RouteDoc routeDoc = extractDocInfo(routeMeta);

                    ClassName className = ClassName.get((TypeElement) routeMeta.getRawType());

                    switch (routeMeta.getType()) {
                        // 11.1 生成 IProvider 类
                        case PROVIDER:  // Need cache provider's super class
                            List<? extends TypeMirror> interfaces = ((TypeElement) routeMeta.getRawType()).getInterfaces();
                            for (TypeMirror tm : interfaces) {
                                routeDoc.addPrototype(tm.toString());

                                if (types.isSameType(tm, iProvider)) {   // Its implements iProvider interface himself.
                                    // This interface extend the IProvider, so it can be used for mark provider
                                    loadIntoMethodOfProviderBuilder.addStatement(
                                            "providers.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, null, " + routeMeta.getPriority() + ", " + routeMeta.getExtra() + "))",
                                            (routeMeta.getRawType()).toString(),
                                            routeMetaCn,
                                            routeTypeCn,
                                            className,
                                            routeMeta.getPath(),
                                            routeMeta.getGroup());
                                } else if (types.isSubtype(tm, iProvider)) {
                                    // 11.2 生成 IProvider 的子类 This interface extend the IProvider, so it can be used for mark provider
                                    loadIntoMethodOfProviderBuilder.addStatement(
                                            "providers.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, null, " + routeMeta.getPriority() + ", " + routeMeta.getExtra() + "))",
                                            tm.toString(),    // So stupid, will duplicate only save class name.
                                            routeMetaCn,
                                            routeTypeCn,
                                            className,
                                            routeMeta.getPath(),
                                            routeMeta.getGroup());
                                }
                            }
                            break;
                        default:
                            break;
                    }

                    // Make map body for paramsType
                    StringBuilder mapBodyBuilder = new StringBuilder();
                    Map<String, Integer> paramsType = routeMeta.getParamsType();
                    Map<String, Autowired> injectConfigs = routeMeta.getInjectConfig();
                    if (MapUtils.isNotEmpty(paramsType)) {
                        List<RouteDoc.Param> paramList = new ArrayList<>();

                        for (Map.Entry<String, Integer> types : paramsType.entrySet()) {
                            mapBodyBuilder.append("put(\"").append(types.getKey()).append("\", ").append(types.getValue()).append("); ");

                            RouteDoc.Param param = new RouteDoc.Param();
                            Autowired injectConfig = injectConfigs.get(types.getKey());
                            param.setKey(types.getKey());
                            param.setType(TypeKind.values()[types.getValue()].name().toLowerCase());
                            param.setDescription(injectConfig.desc());
                            param.setRequired(injectConfig.required());

                            paramList.add(param);
                        }

                        routeDoc.setParams(paramList);
                    }
                    String mapBody = mapBodyBuilder.toString();

                    // 11.3 创建 atlas 相关的类
                    loadIntoMethodOfGroupBuilder.addStatement(
                            "atlas.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, " + (StringUtils.isEmpty(mapBody) ? null : ("new java.util.HashMap<String, Integer>(){{" + mapBodyBuilder.toString() + "}}")) + ", " + routeMeta.getPriority() + ", " + routeMeta.getExtra() + "))",
                            routeMeta.getPath(),
                            routeMetaCn,
                            routeTypeCn,
                            className,
                            routeMeta.getPath().toLowerCase(),
                            routeMeta.getGroup().toLowerCase());

                    routeDoc.setClassName(className.toString());
                    routeDocList.add(routeDoc);
                }

                // 11.4 使用 JavaOpt 生成 groups 相关的 Java 代码 Generate groups
                String groupFileName = NAME_OF_GROUP + groupName;
                JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                        TypeSpec.classBuilder(groupFileName)
                                .addJavadoc(WARNING_TIPS)
                                .addSuperinterface(ClassName.get(type_IRouteGroup))
                                .addModifiers(PUBLIC)
                                .addMethod(loadIntoMethodOfGroupBuilder.build())
                                .build()
                ).build().writeTo(mFiler);

                logger.info(">>> Generated group: " + groupName + "<<<");
                rootMap.put(groupName, groupFileName);
                docSource.put(groupName, routeDocList);
            }

            // 12. 输出 routes 信息到 routes map 中
            if (MapUtils.isNotEmpty(rootMap)) {
                // Generate root meta by group name, it must be generated before root, then I can find out the class of group.
                for (Map.Entry<String, String> entry : rootMap.entrySet()) {
                    loadIntoMethodOfRootBuilder.addStatement("routes.put($S, $T.class)", entry.getKey(), ClassName.get(PACKAGE_OF_GENERATE_FILE, entry.getValue()));
                }
            }

            // 13. 输出 JavaDoc Output route doc
            if (generateDoc) {
                docWriter.append(JSON.toJSONString(docSource, SerializerFeature.PrettyFormat));
                docWriter.flush();
                docWriter.close();
            }

            // 14. 输出 Provider 相关到文件中，也就是生成 Class 文件，会编译到 Jar 中 Write provider into disk
            String providerMapFileName = NAME_OF_PROVIDER + SEPARATOR + moduleName;
            JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                    TypeSpec.classBuilder(providerMapFileName)
                            .addJavadoc(WARNING_TIPS)
                            .addSuperinterface(ClassName.get(type_IProviderGroup))
                            .addModifiers(PUBLIC)
                            .addMethod(loadIntoMethodOfProviderBuilder.build())
                            .build()
            ).build().writeTo(mFiler);

            logger.info(">>> Generated provider map, name is " + providerMapFileName + " <<<");

            // 15. 输出 Routes 到相关文件中 Write root meta into disk.
            String rootFileName = NAME_OF_ROOT + SEPARATOR + moduleName;
            JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                    TypeSpec.classBuilder(rootFileName)
                            .addJavadoc(WARNING_TIPS)
                            .addSuperinterface(ClassName.get(elementUtils.getTypeElement(ITROUTE_ROOT)))
                            .addModifiers(PUBLIC)
                            .addMethod(loadIntoMethodOfRootBuilder.build())
                            .build()
            ).build().writeTo(mFiler);

            logger.info(">>> Generated root, name is " + rootFileName + " <<<");
        }
    }
```

至此， arouter 里面最核心的信息收集告一段落，其余的 AutowiredProcessor 、InterceptorProcessor 也是大同小异，都是<span style="color:#871F78;">使用 apt 对对应的 注解进行处理和收集，然后借助 JavaOpt 去生成对应的 class 文件</span>，这样使得咱们调用的时候直接有对应的类去初始化等等。

##### 目标跳转

```java
ARouter.getInstance().build("/wulala/la").navigation()
```

| 关键角色 | 角色作用 |
|---|---|
| Arouter | 提供各种入口的地方，就是外界的一个接口没啥可说的 |
| _Arouter | Arouter 幕后英雄，实际上是他在干活 |
| Postcard | A container that contains the roadmap. <span style="color:#871F78;">每一个具体的导航的信息存储，路线图存放地，这个是核心，最终具体的跳转啥的都是在这里面进行的</span> |
| PathReplaceService |  重写跳转URL // 实现PathReplaceService接口，并加上一个Path内容任意的注解即可 |
| LogisticsCenter |  LogisticsCenter contains all of the map.
1. Creates instance when it is first used.
2. Handler Multi-Module relationship map(*) 处理多 module 关系图
3. Complex logic to solve duplicate group definition
这个类也是个 大佬 啊，这里也提供了 晚上 Postcard 的方法 |
| AitoWired | 依赖注入 |

Postcard（明信片，特么的 Android 里面叫 Intent 那我就叫 明星片，反正目的都是为了传递信息。。） navigation 解析：
```java
// 通过 path 解析生成对应的 postcard，主要就是设置对应的 path 和 group
    protected Postcard build(String path, String group, Boolean afterReplace) {
        if (TextUtils.isEmpty(path) || TextUtils.isEmpty(group)) {
            throw new HandlerException(Consts.TAG + "Parameter is invalid!");
        } else {
            if (!afterReplace) {
                PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
                if (null != pService) {
                    path = pService.forString(path);
                }
            }
            return new Postcard(path, group);
        }
    }


// navigation 方法，Postcard 最终会调用到 Arouter 里
    public Object navigation(Context mContext, Postcard postcard, int requestCode, NavigationCallback callback) {
        return _ARouter.getInstance().navigation(mContext, postcard, requestCode, callback);
    }


// 最终调用地
 protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
     /*
     预处理服务
    ``` java
    // 实现 PretreatmentService 接口，并加上一个Path内容任意的注解即可
    @Route(path = "/xxx/xxx")
    public class PretreatmentServiceImpl implements PretreatmentService {
        @Override
        public boolean onPretreatment(Context context, Postcard postcard) {
            // 跳转前预处理，如果需要自行处理跳转，该方法返回 false 即可
        }
     */
     // 1. 获取预处理，如果自己已经处理了，那么就不会进入下面的流程
        PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
        if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
            // Pretreatment failed, navigation canceled.
            return null;
        }

        // Set context to postcard.
        postcard.setContext(null == context ? mContext : context);

        try {
            // 2. 对 Postcard 进行深加工，首先从 routes 的 map 中找到对应的 RouteMeta 然后将这里面的信息放到 Postcard 中，例如 target bundle type priority 等等
            LogisticsCenter.completion(postcard);
        } catch (NoRouteFoundException ex) {
            logger.warning(Consts.TAG, ex.getMessage());

            if (debuggable()) {
                // Show friendly tips for user.
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        Toast.makeText(mContext, "There's no route matched!\n" +
                                " Path = [" + postcard.getPath() + "]\n" +
                                " Group = [" + postcard.getGroup() + "]", Toast.LENGTH_LONG).show();
                    }
                });
            }

            if (null != callback) {
                callback.onLost(postcard);
            } else {
                // No callback for this invoke, then we use the global degrade service.
                DegradeService degradeService = ARouter.getInstance().navigation(DegradeService.class);
                if (null != degradeService) {
                    degradeService.onLost(context, postcard);
                }
            }

            return null;
        }

        if (null != callback) {
            callback.onFound(postcard);
        }

        // 3. 判断是否绿色通道，绿色就主线程跑，否则在异步线程 搞 intercept 先    
        if (!postcard.isGreenChannel()) {   // It must be run in async thread, maybe interceptor cost too mush time made ANR.
            interceptorService.doInterceptions(postcard, new InterceptorCallback() {
                /**
                 * Continue process
                 *
                 * @param postcard route meta
                 */
                @Override
                public void onContinue(Postcard postcard) {
                    _navigation(postcard, requestCode, callback);
                }

                /**
                 * Interrupt process, pipeline will be destory when this method called.
                 *
                 * @param exception Reson of interrupt.
                 */
                @Override
                public void onInterrupt(Throwable exception) {
                    if (null != callback) {
                        callback.onInterrupt(postcard);
                    }

                    logger.info(Consts.TAG, "Navigation failed, termination by interceptor : " + exception.getMessage());
                }
            });
        } else {
            return _navigation(postcard, requestCode, callback);
        }

        return null;
    }

// _Arouter 中的 _navigation 实现
    private Object _navigation(final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = postcard.getContext();

        switch (postcard.getType()) {
            case ACTIVITY:
                // 1.1 activity 首先封装 Intent
                // Build intent
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());

                // 1.2 Set flags.
                int flags = postcard.getFlags();
                if (0 != flags) {
                    intent.setFlags(flags);
                }

                // 1.3 Non activity, need FLAG_ACTIVITY_NEW_TASK
                if (!(currentContext instanceof Activity)) {
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                // 1.4 Set Actions
                String action = postcard.getAction();
                if (!TextUtils.isEmpty(action)) {
                    intent.setAction(action);
                }

                // 1.5 Navigation in main looper.
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        startActivity(requestCode, currentContext, intent, postcard, callback);
                    }
                });
                break;
            case PROVIDER:
                // 2. 如果是 Provider 直接返回，杂用，用户看着办
                return postcard.getProvider();
            case BOARDCAST:
            case CONTENT_PROVIDER:
            case FRAGMENT:
                // 3. 是 Fragment 返回，然后看用户怎么用
                Class<?> fragmentMeta = postcard.getDestination();
                try {
                    Object instance = fragmentMeta.getConstructor().newInstance();
                    if (instance instanceof Fragment) {
                        ((Fragment) instance).setArguments(postcard.getExtras());
                    } else if (instance instanceof android.support.v4.app.Fragment) {
                        ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                    }

                    return instance;
                } catch (Exception ex) {
                    logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
                }
            case METHOD:
            case SERVICE:
            default:
                return null;
        }

        return null;
    }

```

### 总结

经过上边的完整分析，可以看到完整的 Arouter 也不是特别复杂，但是中间也有一些精巧的设计，例如 interceptor provider 以及 更多 预处理或者 重写跳转 URL 等等类似 hook 的功能，人家的设计还是比较完整的。

整体思想梳理，首先就是通过 Router 等注解收集各种信息，收集完毕后会在之后的 Postcard 中承载，然后经过最终的 _Arouter 去 navigation 分发和调用。

还有依赖注入，升降级别等等。

