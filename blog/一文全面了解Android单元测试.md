---

		title:   一文全面了解Android单元测试
		date: 2018/07/09 23:06:00   
		tags: 
		- Android测试
		categories: 安卓进阶
		thumbnail: http://img06.tooopen.com/images/20170106/tooopen_sy_195812293421.jpg
---

---
# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

[==》完整项目单元测试学习案例](https://github.com/JsonChao/Awesome-WanAndroid/tree/master/app/src/test/java/json/chao/com/wanandroid)

众所周知，一个好的项目需要不断地打造，而一些有效的测试则是加速这一过程的利器。本篇博文将带你了解并逐步深入Android单元测试。

## 什么是单元测试？
---

单元测试就是针对类中的某一个方法进行验证是否正确的过程，单元就是指**独立的粒子**，在Android和Java中大都是指方法。

## 为什么要进行单元测试？
---

使用单元测试可以提高开发效率，当项目随着迭代越来越大时，每一次编译、运行、打包、调试需要耗费的时间会随之上升，因此，使用单元测试可以不需这一步骤就可以对单个方法进行功能或逻辑测试。
同时，为了能测试每一个细分功能模块，需要将其相关代码抽成相应的方法封装起来，这也在一定程度上改善了代码的设计。因为是单个方法的测试，所以能更快地定位到bug。

单元测试case需要对这段业务逻辑进行验证。在验证的过程中，开发人员可以**深度了解业务流程**，同时新人来了看一下项目单元测试就知道**哪个逻辑跑了多少函数，需要注意哪些边界**——是的，单元测试做的好和文档一样**具备业务指导能力。**

## Android测试的分类
---

Android测试主要分为三个方面：

- 1）、单元测试（Junit4、Mockito、PowerMockito、Robolectric）
- 2）、UI测试（Espresso、UI Automator）
- 3）、压力测试（Monkey）


# 一、单元测试之基础Junit4
---

## 什么是Junit4？
---

Junit4是事实上的Java标准测试库，并且它是JUnit框架有史以来的最大改进，其主要目标便是利用Java5的Annotation特性简化测试用例的编写。

## 开始使用Junit4进行单元测试
---

### 1.Android Studio已经自动集成了Junit4测试框架，如下

```groovy
    dependencies {
        ...
        testImplementation 'junit:junit:4.12'
    }
```
    
### 2.Junit4框架使用时涉及到的重要注解如下

```java
    @Test 指明这是一个测试方法 (@Test注解可以接受2个参数，一个是预期错误
    expected，一个是超时时间timeout，
    格式如 @Test(expected = IndexOutOfBoundsException.class), 
    @Test（timeout = 1000)
    @Before 在所有测试方法之前执行
    @After 在所有测试方法之后执行
    @BeforeClass 在该类的所有测试方法和@Before方法之前执
    行 （修饰的方法必须是静态的）@AfterClass 在该类的所有测试方法和@After
    方法之后执行（修饰的方法必须是静态的）
    @Ignore 忽略此单元测试
```


此外，很多时候，因为某些原因（比如正式代码还没有实现等），我们可能想让JUnit忽略某些方法，让它在跑所有测试方法的时候不要跑这个测试方法。要达到这个目的也很简单，只需要在要被忽略的**测试方法前面加上@Ignore**就可以了
    
### 3.主要的测试方法——断言

```java
    assertEquals(expected, actual) 判断2个值是否相等，相等则测试通过。
    assertEquals(expected, actual, tolerance) tolerance 偏差值
```
    
注意：上面的每一个方法，都有一个重载的方法，可以加一个String类型的参数，表示如果验证失败的话，将**用这个字符串作为失败的结果报告**。

### 4.自定义Junit Rule——实现TestRule接口并重写apply方法

```java
    public class JsonChaoRule implements TestRule {
    
        @Override
        public Statement apply(final Statement base, final Description description) {
            Statement repeatStatement =  new Statement() {
                @Override
                public void evaluate() throws Throwable {
                        //测试前的初始化工作
                        //执行测试方法
                        base.evaluate();
                        //测试后的释放资源等工作
                }
            };
            return repeatStatement;
        }
    }
```
    
然后在想要的测试类中使用@Rule注解声明使用JsonChaoRule即可（注意**被@Rule注解的变量必须是final的**）：

```java
    @Rule
    public final JsonChaoRule repeatRule = new JsonChaoRule();
```

### 5.开始上手，使用Junit4进行单元测试

- 1.编写测试类。
- 2.鼠标右键点击测试类，选择选择Go To->Test
    （或者使用快捷键Ctrl+Shift+T，此快捷键可
    以在方法和测试方法之间来回切换）在Test/java/项目
    测试文件夹/下自动生成测试模板。
- 3.使用断言（assertEqual、assertEqualArrayEquals等等)进行单元测试。
- 4.右键点击测试类，Run编写好的测试类。
    
### 6.使用Android Studio自带的Gradle脚本自动化单元测试

点击Android Studio中的**Gradle projects**下的**app/Tasks/verification/test**即可同时测试module下所有的测试类（案例），并在**module下的build/reports/tests/下**生成对应的**index.html测试报告**。

### 7.对Junit4的总结：

- 优点：速度快，支持代码覆盖率等代码质量的检测工具，
- 缺点：无法单独对Android UI，一些类进行操作，与原生JAVA有一些差异。
    
可能涉及到的额外的概念：

打桩方法：使方法简单快速地返回一个有效的结果。

测试驱动开发：编写测试，实现功能使测试通过，然后不断地使用这种方式实现功能的快速迭代开发。

# 二、单元测试之基础Mockito


## 什么是Mockito？

Mockito 是美味的 Java 单元测试 Mock 框架，mock可以模拟各种各样的对象，从而代替真正的对象做出希望的响应。

## 开始使用Mockito进行单元测试

### 1.在build.gradle里面添加Mcokito的依赖

```java
    testImplementation 'org.mockito:mockito-core:2.7.1'
```

### 2.使用mock()方法模拟对象

```java
    Person mPerson = mock(Person.class); 
```

#### 能量补充站（-vov-）    

在JUnit框架下，case（即每一个测试点，带@Test注解的那个函数）也是个函数，直接调用这个函数就不是case，和case是无关的，两者并不会相互影响，可以直接调用以减少重复代码。**单元测试不应该对某一个条件过度耦合**，因此，需要用mock解除耦合，**直接mock出网络请求得到的数据，单独验证页面对数据的响应。**


### 3.验证方法的调用，指定方法的返回值，或者执行特定的动作

```java
    when(iMathUtils.sum(1, 1)).thenReturn(2); 
    doReturn(3).when(iMathUtils).sum(1,1);   
    //给方法设置桩可以设置多次，只会返回最后一次设置的值
    doReturn(2).when(iMathUtils).sum(1,1);
    
    //验证方法调用次数
    //方法调用1次
    Mockito.when(mockValidator.verifyPassword("xiaochuang_is_handsome")).thenReturn(true);
    //方法调用3次
    Mockito.when(mockValidator.verifyPassword("xiaochuang_is_handsome")
    ， Mockito.times(3).thenReturn(true);
    
    //verify方法用于验证“模仿对象”的互动或验证发生的某些行为
    verify(mPerson, atLeast(2)).getAge();
    
    //参数匹配器,用于匹配特定的参数
    any()
    contains()
    argThat()
    when(mPerson.eat(any(String.class))).thenReturn("米饭");
    
    //除了mock()外，spy()也可以模拟对象，spy与mock的
    //唯一区别就是默认行为不一样：spy对象的方法默认调用
    //真实的逻辑，mock对象的方法默认什么都不做，或直接
    //返回默认值
    //如果要保留原来对象的功能，而仅仅修改一个或几个
    //方法的返回值，可以采用spy方法,无参构造的类初始
    //化也使用spy方法
    Person mPerson = spy(Person.class); 
    
    //检查入参的mocks是否有任何未经验证的交互
    verifyNoMoreInteractions(iMathUtils);
```

### 4.使用Mockito后的思考

简单的测试会使整体的代码更简单，更可读、更可维护。如果你**不能把测试写的很简单，那么请在测试时重构你的代码**。	

- 优点：丰富强大的方式验证“模仿对象”的互动或验证发生的某些行为
- 缺点：Mockito框架不支持mock匿名类、final类、static方法、private方法。
    
虽然，static方法可以使用wrapper静态类的方式实现mockito的单元测试，但是，毕竟过于繁琐，因此，PowerMockito由此而来。


# 三、拯救Mockito于水深火热的PowerMockito

## 什么是PowerMockito？

PowerMockito是一个扩展了Mockito的具有更强大功能的单元测试框架，它支持mock匿名类、final类、static方法、private方法

## 开始PowerMockito之旅


### 1.在build.gradle里面添加Mcokito的依赖

```java
    testImplementation 'org.powermock:powermock-module-junit4:1.6.5'
    testImplementation 'org.powermock:powermock-api-mockito:1.6.5'
```

### 2.用PowerMockito来模拟对象

```java
    //使用PowerMock须加注解@PrepareForTest和@RunWith(PowerMockRunner.class)（@PrepareForTest()里写的
    // 是对应方法所在的类	，mockito支持的方法使用PowerMock的形式实现时，可以不加这两个注解）
    @PrepareForTest(T.class)
    @RunWith(PowerMockRunner.class)

    //mock含静态方法或字段的类	
    PowerMockito.mockStatic(Banana.class);
    
    //Powermock提供了一个Whitebox的class，可以方便的绕开权限限制，可以get/set private属性，实现注入。
    //也可以调用private方法。也可以处理static的属性/方法，根据不同需求选择不同参数的方法即可。
    修改类里面静态字段的值
    Whitebox.setInternalState(Banana.class, "COLOR", "蓝色");
    
    //调用类中的真实方法
    PowerMockito.when(banana.getBananaInfo()).thenCallRealMethod();
    
    //验证私有方法是否被调用
    PowerMockito.verifyPrivate(banana, times(1)).invoke("flavor");
    
    //忽略调用私有方法
    PowerMockito.suppress(PowerMockito.method(Banana.class, "flavor"));
    
    //修改私有变量
    MemberModifier.field(Banana.class, "fruit").set(banana, "西瓜");
    
    //使用PowerMockito mock出来的对象可以直接调用final方法
    Banana banana = PowerMockito.mock(Banana.class);
    
    //whenNew 方法的意思是之后 new 这个对象时，返回某个被 Mock 的对象而不是让真的 new
    //新的对象。如果构造方法有参数，可以在withNoArguments方法中传入。
    PowerMockito.whenNew(Banana.class).withNoArguments().thenReturn(banana);
```
    
### 3.使用PowerMockRule来代替@RunWith(PowerMockRunner.class)的方式，需要多添加以下依赖：

```java
    testImplementation "org.powermock:powermock-module-junit4-rule:1.7.4"
    testImplementation "org.powermock:powermock-classloading-xstream:1.7.4"
```

使用示例如下：

```java
    @Rule
    public PowerMockRule mPowerMockRule = new PowerMockRule();
```

### 4.使用Parameterized来进行参数化测试：

**通过注解@Parameterized.parameters提供一系列数据给构造器中的构造参数**或给**被注解@Parameterized.parameter注解的public全局变量**

```java
    RunWith(Parameterized.class)
    public class ParameterizedTest {

        private int num;
        private boolean truth;
    
        public ParameterizedTest(int num, boolean truth) {
            this.num = num;
            this.truth = truth;
        }
    
        //被此注解注解的方法将把返回的列表数据中的元素对应注入到测试类
        //的构造函数ParameterizedTest(int num, boolean truth)中
        @Parameterized.Parameters
        public static Collection providerTruth() {
            return Arrays.asList(new Object[][]{
                    {0, true},
                    {1, false},
                    {2, true},
                    {3, false},
                    {4, true},
                    {5, false}
            });
        }

    //    //也可不使用构造函数注入的方式，使用注解注入public变量的方式
    //    @Parameterized.Parameter
    //    public int num;
    //    //value = 1指定括号里的第二个Boolean值
    //    @Parameterized.Parameter(value = 1)
    //    public boolean truth;
    
        @Test
        public void printTest() {
            Assert.assertEquals(truth, print(num));
            System.out.println(num);
        }
    
        private boolean print(int num) {
            return num % 2 == 0;
        }
    
    }
```

# 四、能在Java单元测试里面执行Android代码的Robolectric

## 什么是Robolectric？

Robolectric通过**一套能运行在JVM上的Android代码**，解决了在Java单元测试中很难进行Android单元测试的痛点。

## 进入Roboletric的领地
---

### 1.在build.gradle里面添加Robolectric的依赖
   
```java
        //Robolectric核心
        testImplementation "org.robolectric:robolectric:3.8"
        //支持support-v4
        testImplementation 'org.robolectric:shadows-support-v4:3.4-rc2'
        //支持Multidex功能
        testImplementation "org.robolectric:shadows-multidex:3.+" 
```

### 2.Robolectric常用用法

首先给指定的测试类上面进行配置

```java
    @RunWith(RobolectricTestRunner.class)
    //目前Robolectric最高支持sdk版本为23。
    @Config(constants = BuildConfig.class, sdk = 23)
```

下面是一些常用用法：

```java
    //当Robolectric.setupActivity()方法返回的时候，
    //默认会调用Activity的onCreate()、onStart()、onResume()
    mTestActivity = Robolectric.setupActivity(TestActivity.class);
    
    //获取TestActivity对应的影子类，从而能获取其相应的动作或行为
    ShadowActivity shadowActivity = Shadows.shadowOf(mTestActivity);
    Intent intent = shadowActivity.getNextStartedActivity();
    
    //使用ShadowToast类获取展示toast时相应的动作或行为
    Toast latestToast = ShadowToast.getLatestToast();
    Assert.assertNull(latestToast);
    //直接通过ShadowToast简单工厂类获取Toast中的文本
    Assert.assertEquals("hahaha", ShadowToast.getTextOfLatestToast());
    
    //使用ShadowAlertDialog类获取展示AlertDialog时相应的
    //动作或行为（暂时只支持app包下的，不支持v7。。。）
    latestAlertDialog = ShadowAlertDialog.getLatestAlertDialog();
    AlertDialog latestAlertDialog = ShadowAlertDialog.getLatestAlertDialog();
    Assert.assertNull(latestAlertDialog);
        
    //使用RuntimeEnvironment.application可以获取到
    //Application，方便我们使用。比如访问资源文件。
    Application application = RuntimeEnvironment.application;
    String appName = application.getString(R.string.app_name);
    Assert.assertEquals("WanAndroid", appName);
    
    //也可以直接通过ShadowApplication获取application
    ShadowApplication application = ShadowApplication.getInstance();
    Assert.assertNotNull(application.hasReceiverForIntent(intent));
```

自定义Shadow类：

```java
    @Implements(Person.class)
    public class ShadowPerson {
    
        @Implementation
        public String getName() {
            return "AndroidUT";
        }
    
    }
    
    @RunWith(RobolectricTestRunner.class)
    @Config(constants = BuildConfig.class,
            sdk = 23,
            shadows = {ShadowPerson.class})
    
        Person person = new Person();
        //实际上调用的是ShadowPerson的方法，输出JsonChao
        Log.d("test", person.getName());
         
        ShadowPerson shadowPerson = Shadow.extract(person);
        //测试通过
        Assert.assertEquals("JsonChao", shadowPerson.getName());
        
    }
```

注意：异步测试出现一些问题（比如改变一些编码习惯，比如回调函数不能写成匿名内部类对象，需要定义一个全局变量，并破坏其封装性，即提供一个get方法，供UT调用），解决方案**使用Mockito来结合进行测试，将异步转为同步**。

### 3.Robolectric的优缺点

- 优点：支持大部分Android平台依赖类底层的引用与模拟。
- 缺点：异步测试有些问题，需要结合一些框架来配合完成更多功能。


# 五、单元测试覆盖率报告生成之jacoco

## 什么是Jacoco

Jacoco的全称为Java Code Coverage（Java代码覆盖率），可以**生成java的单元测试代码覆盖率报告**。

## 加入Jacoco到你的单元测试大家族

在应用Module下加入jacoco.gradle自定义脚本，app.gradle apply from它，同步，即可看到在app的Task下生成了Report目录，Report目录
下生成了JacocoTestReport任务。

```java
    apply plugin: 'jacoco'
    
    jacoco {
        toolVersion = "0.7.7.201606060606" //指定jacoco的版本
        reportsDir = file("$buildDir/JacocoReport") //指定jacoco生成报告的文件夹
    }
    
    //依赖于testDebugUnitTest任务
    task jacocoTestReport(type: JacocoReport, dependsOn: 'testDebugUnitTest') {
        group = "reporting" //指定task的分组
        reports {
            xml.enabled = true //开启xml报告
            html.enabled = true //开启html报告
        }
    
        def debugTree = fileTree(dir: "${buildDir}/intermediates/classes/debug",
                includes: ["**/*Presenter.*"],
                excludes: ["*.*"])//指定类文件夹、包含类的规则及排除类的规则，
                //这里我们生成所有Presenter类的测试报告
        def mainSrc = "${project.projectDir}/src/main/java" //指定源码目录
    
        sourceDirectories = files([mainSrc])
        classDirectories = files([debugTree])
        executionData = files("${buildDir}/jacoco/testDebugUnitTest.exec")//指定报告数据的路径
    }
```

在Gradle构建板块**Gradle.projects**下的**app/Task/verification**下，其中**testDebugUnitTest**构建任务会生成单元测试结果报告，包**含xml及html**格式，分别对应**test-results和reports**文件夹；jacocoTestReport任务会生成单元测试覆盖率报告，结果存放在jacoco和JacocoReport文件夹。
    
![image](https://user-gold-cdn.xitu.io/2020/1/8/16f82c8e037af99f?w=774&h=241&f=png&s=58736)

生成的JacocoReport文件夹下的index.html即对应的单元测试覆盖率报告，用浏览器打开后，可以看到覆盖情况被不同的颜色标识出来，其中**绿色表示代码被单元测试覆盖到，黄色表示部分覆盖，红色则表示完全没有覆盖到**。

# 六、单元测试的流程

要验证程序正确性，必然要给出所有可能的条件（极限编程），并验证其行为或结果，才算是100%覆盖条件。实际项目中，验证**一般条件**和**边界条件**就OK了。

在实际项目中，**单元测试对象与页面是一对一的**，并不建议跨页面，这样的单元测试耦合太大，维护困难。
需要写完后，看覆盖率，找出单元测试中没有覆盖到的函数分支条件等，然后继续补充单元测试case列表，并在单元测试工程代码中补上case。
直到规划的**页面中所有逻辑的重要分支、边界条件都被覆盖**，该项目的单元测试结束。

### 建议（-ovo-）~

可以从公司项目**小规模使用**，形成**自己的单元测试风格**后，就可更大范围地推广了。

## 参考链接：
---
1、[必知必会 | Android 测试相关的方方面面都在这儿](http://www.wanandroid.com/blog/show/2085)

2、[在Android Studio中进行单元测试和UI测试](https://www.jianshu.com/p/03118c11c199)

3、[Android单元测试（一）](https://www.jianshu.com/p/0a8bbfe6cba2)

4、[Android单元测试（二）](https://www.jianshu.com/subscriptions)


# Contanct Me

##  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

##  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="https://user-gold-cdn.xitu.io/2020/1/7/16f7dc352011e1fe?w=1013&h=1920&f=jpeg&s=86819" width=35%>
</div>
        

##  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


## About me

- ### Email: [chao.qu521@gmail.com]()
- ### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- ### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    


### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。


