## IoC

1. 初始化容器
2. 加载配置文件
3. 解析配置文件，初始化 BeanDefination
4. 根据 BeanDefination，调用动态代理实例化对象
5. 根据 BeanDefination 中的属性信息，调用实例的 setter 方法设置属性

## AOP

1. 初始化 Aop 容器。
2. 读取配置文件。
3. 将配置文件装换为 Aop 能够识别的数据结构 – `Advisor`。这里展开讲一讲这个advisor。Advisor对象中包又含了两个重要的数据结构，一个是 `Advice`，一个是 `Pointcut`。`Advice`的作用就是描述一个切面的行为，`pointcut`描述的是切面的位置。两个数据结的组合就是”在哪里，干什么“。这样 `Advisor` 就包含了”在哪里干什么“的信息，就能够全面的描述切面了。
4. Spring 将这个 Advisor 转换成自己能够识别的数据结构 – `AdvicedSupport`。Spirng 动态的将这些方法拦截器织入到对应的方法。
5. 生成动态代理代理。
6. 提供调用，在使用的时候，调用方调用的就是代理方法。也就是已经织入了增强方法的方法。