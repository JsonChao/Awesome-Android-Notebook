    注意：本笔记为设计模式核心学习笔记，为笔者快速复习和回顾设计模式时使用，更详细的教程请查看更专业的设计模式教程。


## 一、设计模式六大原则

设计模式有六大原则，如下所示：

- 单一职责原则
- 开放封闭原则
- 里氏替换原则
- 依赖倒置
- 迪米特原则
- 接口隔离原则

### 单一职责原则

一个类应该仅有一个引起它变化的原因，即不要让一个类承担过多的职责，以此降低耦合性。

### 开放封闭原则

类、函数、模块应该是可以扩展的，但是不可以修改，即对扩展开放，修改封闭。

### 里氏替换原则

所有引用基类的地方都能透明地替换为子类对象，即可以在定义时尽量使用基类对象，等到运行时再确定其子类类型，用子类对象来替换父类对象。

### 依赖倒置原则

高层、底层模块、模块间和细节都应该依赖于抽象，即通过接口或抽象类产生依赖关系。

### 迪米特原则

一个软件实体应该尽可能少地与其它实体发生相互作用，即最少知识原则。

如果一个对象需要调用其它对象的某个方法，可以通过第三者来调用，这个第三者的作用就如Android中的事件总线EventBus一样。

### 接口隔离原则

一个类对另一个类的依赖应该建立在最小的接口上。

## 二、设计模式分类

GoF提出的设计模式有23种，按照目的准则分类，有三大类：

- 创建性设计模式5种：单例、工厂方法、抽象工厂、建造者、原型。
- 结构型设计模式7种：适配器、装饰、代理、外观、桥接、组合、享元。
- 行为型设计模式11种：策略、模板方法、观察者、迭代器、责任链、命令、备忘录、状态、访问者、中介者、解释器。

## 三、Android开发常用设计模式

### 1、创建型设计模式

#### 单例模式

保证一个类仅有一个实例，提供一个访问它的全局访问点。

单例模式共有5种写法：

##### 1、饿汉模式

    public class Singleton {
        private static Singleton instance = new Singleton;
        private Singleton () {
            
        }
        public static Singleton getInstance() {
            return instance;
        }
    }

- 在类加载的时候就完成实例化，如果从始至终未使用这个实例，则会造成内存的浪费。

##### 2、懒汉模式（线程安全）

    public class Singletion {
        private static Singleton instance;
        private Singleton () {
        }
        public static synchronized Singleton getInstance() {
            if (instance == null) {
                instance = new Singleton();
            }
            return instance;
        }
    }

- 为了处理并发，每次调用getInstance方法时都需要进行同步，会有不必要的同步开销。

##### 3、双重检查模式（DCL）

    public class Singleton {
        private static volatile Singleton instance;
        private Singleton {
        }
        public static Singleton getInstance() {
            if (instance == null) {
                synchronized (Singleton.class) {
                    if (instance == null) {
                        instance = new Singleton();
                    }
                }
            }
            return instance;
        }
    }
    
- 第一次判空，省去了不必要的同步。第二次是在Singleton等于空时才创建实例。
- 使用volatile保证了实例的可见性。
- DCL在一定程度上解决了资源的消耗和多余的同步、线程安全等问题，但是在某些情况下会失效。

假设线程A执行到instance = new Singleton()语句，看起来只有一行代码，但实际上它并不是原子操作，这句代码最终会被编译成多条汇编指令，它大致做了3件事：

1）给instance的实例分配内存。

2）调用Singleton()构造函数，初始化成员字段。

3）将instance对象指向分配的内存空间（此时instance就不是null了）。

但是，由于Java编译器允许处理器乱序执行，以及JDK1.5之前JMM中的Cache、寄存器到主内存回写顺序的规定，上面的2和3的顺序是无法保证的，也就是说，执行顺序可能是1-2-3也可能是1-3-2。如果是后者，并且在3执行完毕、2未执行之前，被切换到线程B上，这时候instance因为已经在线程A内执行过了3，instance已经是非空了，所以，线程B直接取走instance，再使用时就会出错，这就是DCL失效问题，而且这种难以跟踪难以重现的错误可能会隐藏很久。

在JDK1.5之后，SUN官方已经注意到这种问题，调整了JVM，具体化了volatile关键字，因此，如果JDK1.5或之后的版本，只需要将instance的定义改成private volatile static Singleton instance = null就可以保证instance对象每次都是从主内存中读取，就可以使用DCL的写法来完成单例模式。当然，volatile或多或少也会影响到性能，但考虑到程序的正确性，这点牺牲也是值得的。

DCL优点：资源利用率高，第一次执行getInstance时单例对象才会被实例化，效率高。

缺点：第一次加载稍慢，也由于JMM的原因导致偶尔会失败。在高并发环境下也有一定的缺陷，虽然发生概率很小。DCL模式是使用最多的单例实现方式，它能够在需要时才实例化对象，并且能在绝大多数场景下保证对象的唯一性，除非你的代码在并发场景比较复杂或低于JDK1.6版本下使用，否则，这种方式一般能够满足要求。
    

##### 4、静态内部类单例模式

    public class Singleton() {
        private Singleton() {
        }
        public static Singleton getInstance() {
            return SingletonHolder.sInstance;
        }
        private static class SingletonHolder {
            private static final Singleton sInstance = new Singleton();
        }
    }
    
- 第一次调用getInstance方法时虚拟机才加载SingletonHolder并初始化sInstance，这样保证了线程安全和实例的唯一性。

##### 5、枚举单例

    public enum Singleton {
        INSTANCE;
        public void doSomeThing() {
        }
    }
    
- 默认枚举实例的创建是线程安全的，并且在任何情况下都是单例。
- 简单、可读性不高。

注意：上面的几种单例模式创建的单例对象被反序列化时会重新创建实例，可以重写readReslove方法返回当前的单例对象。

#### 简单工厂模式（补充）

也称为静态工厂方法模式，由一个工厂对象决定创建出哪一种产品类的实例。

简单工厂模式中有如下角色：

- 工厂类：核心，负责创建所有实例的内部逻辑，由外界直接调用。
- 抽象产品类：要创建所有对象的抽象父类，负责描述所有实例所共有的公共接口。
- 具体产品类：要创建的产品。

##### 简单示例

1、抽象产品类

    public abstract class Computer {
        public abstarct void start();
    }
    
2、具体产品类

    public class LenovaComputer extends Computer {
        @Override
        public void start() {
            ...
        }
    }
    
    public class HpComputer extends Computer {
        @Override
        public void start() {
            ...
        }
    }
    
    public class AsusComputer extends Computer {
        @Override
        public void start() {
            ...
        }
    }
    
3、工厂类

    public class ComputerFactory {
        public static Computer createComputer(String type) {
            Computer mComputer = null;
            switch (type) {
                case "lenovo":
                    mComputer = new LenovoComputer();
                    break;
                case "hp":
                    mComputer = new HpComputer();
                    break;
                case "asus":
                    mComputer = new AsusComputer();
                    break;
            }
            return mComputer;
        }
    }
    
- 它需要知道所有工厂类型，因此只适合工厂类负责创建的对象比较少的情况。
- 避免直接实例化类，降低耦合性。
- 增加新产品需要修改工厂，违背开放封闭原则。

#### 工厂方法模式

定义一个用于创建对象的接口，使类的实例化延迟到子类。

工厂方法有以下角色：

- 抽象产品类。
- 具体产品类。
- 抽象工厂类：返回一个泛型的产品对象。
- 具体工厂类：返回具体的产品对象。

##### 简单示例

抽象产品类和具体产品类同简单工厂一样。

3、抽象工厂类

    public abstract class ComputerFactory {
        public abstract <T extends Computer> T createComputer(Class<T> clz);
    }

4、具体工厂类

    public class GDComputerFactory extends ComputerFactory {
        @Override
        public <T extends Computer> T createComputer(Class<T> clz) {
            Computer computer = null;
            String classname = clz.getName();
            try {
                computer = (Computer) Class.forName(classname).newInstance();
            } catch (Exception e) {
                e.printStackTrace();
            }
            return (T) computer;
        }
    }
    
- 相比简单工厂，如果我们需要新增产品类，无需修改工厂类，直接创建产品即可。

#### 建造者模式

将一个复杂对象的构建和它的表示分离，使得同样的构建过程可以创建不同的表示。

建造者有以下角色：

- 导演类：负责安排已有模块的安装顺序，最后通知建造者开始建造。
- 建造者：抽象Builder类，用于规范产品的组建。
- 具体建造者：实现抽象Builder类的所有方法，并返回建造好的对象。
- 产品类。

##### 简单示例

1、产品类

    public class Computer {
        private String mCpu;
        private Stiring mMainboard;
        private String mRam;
        public void setmCpu(String mCpu) {
            this.mCpu = mCpu;
        }
        public void setmMainboard(String mMainboard) {
            this.mMainboard = mMainboard;
        }
        public void setmRam(String mRam) {
            this.mRam = mRam;
        }
    }
    
2、抽象建造者

    public abstract class Builder {
        public abstract void buildCpu(String cpu);
        public abstract void buildMainboard(String mainboard);
        public abstract void buildRam(String ram);
        public abstract Computer create();
    }
    
3、具体建造者

    public class MoonComputerBuilder extends Builder {
        private Computer mComputer = new Computer();
        
        @Override
        public void buildCpu(String cpu) {
            mComputer.setmCpu(cpu);
        }
        
        @Override
        public void buildMainboard(String mainboard) {
            mComputer.setmMainboard(mainboard);
        }
        
        @Override
        public void buildRam(String ram) {
            mComputer.setmRam(ram);
        }
        
        @Override
        public Computer create() {
            return mComputer;
        }
    }
    
4、导演类

    public class Director {
        Builder mBuilder = null;
        public Director (Builder builder) {
            this.mBuilder = builder;
        }
        
        public Computer createComputer(String cpu, String mainboard, String ram) {
            this.mBuilder.buildCpu(cpu);
            this.mBuilder.buildMainboard(mainboard);
            this.mBuilder.buildRam(ram);
            return mBuilder.create();
        }
    }

- 屏蔽产品内部组成细节。
- 具体建造者类之间相互独立，容易扩展。
- 会产生多余的建造者对象和导演类。

### 2、结构型设计模式

#### 1、代理模式

为其它对象提供一种代理以控制这个对象的访问。

代理模式中有以下角色：

- 抽象主题类：声明真实主题和代理的共同接口方法。
- 真实主题类。
- 代理类：持有对真实主题类的引用。
- 客户端类。

##### 静态代理示例代码

1、抽象主题类

    public interface IShop {
        void buy();
    }
    
2、真实主题类

    public class JsonChao implements IShop {
        @Override 
        public void buy() {
            ...
        }
    }
    
3、代理类

    public class Purchasing implements IShop {
        private IShop mShop;
        public Purchasing(IShop shop) {
            this.mShop = shop;
        }
        
        @Override 
        public void buy() {
            mShop.buy();
        }
    }
    
4、客户端类

    public class Clent {
        
        public static void main(String[] args) {
            IShop jsonChao = new JsonChao();
            IShop purchasing = new Purchasing(jsonChao);
            purchasing.buy();
        }
    }
    
##### 动态代理

在代码运行时通过反射来动态地生成代理类的对象，并确定到底来代理谁。

##### 动态代理示例代码

改写静态代理的代理类和客户端类，如下所示：

1、动态代理类

    public class DynamicPurchasing implements InvocationHandler {
        private Object obj;
        public DynamicPurchasing(Object obj) {
            this.obj = obj;
        }
        
        @Overrdie
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            return method.invoke(obj, args);
        }
    }
    
2、客户端类

    public class Clent {
        
        public static void main(String[] args) {
            IShop jsonChao = new JsonChao();
            DynamicPurchasing mDynamicPurchasing = new DynamicPurchasing(jsonChao);
            ClassLoader cl = jsonChao.getClass.getClassLoader();
            IShop purchasing = Proxy.newProxyInstance(cl, new Class[]{IShop.class}, mDynamicPurchasing);
            purchasing.buy();
        }
    }

- 真实主题类发生变化时，由于它实现了公用的接口，因此代理类不需要修改。

#### 装饰模式

动态地给一个对象添加一些额外的职责。

装饰模式有以下角色：

- 抽象组件：接口/抽象类，被装饰的最原始的对象。
- 具体组件：被装饰的具体对象。
- 抽象装饰者：扩展抽象组件的功能。
- 具体装饰者：装饰者具体实现类。

##### 示例代码

1、抽象组件

    public abstract class Swordsman {
        public abstract void attackMagic();
    }
    
2、具体组件

    public class YangGuo extends Swordsman {
        @Override
        public void attackMagic() {
            ...
        }
    }
    
3、抽象装饰者

抽象装饰者必须持有抽象组件的引用，以便扩展功能。

    public abstract class Master extends Swordsman {
        private Swordsman swordsman;
        public Master(Swordsman swordsman) {
            this.swordman = swordman;
        }
        
        @Override
        public void attackMagic() {
            swordsman.attackMagic();
        }
    }
    
4、具体装饰者

    public class HongQiGong extends Master {
        public HongQiGong(Swordsman swordsman) {
            this.swordsman = swordsman;
        }
        
        public void teachAttackMagic() {
            ...
        }
        
        @Override
        public void attackMagic() {
            super.attackMagic();
            teackAttackMagic();
        }
    }
    
5、使用
    
    YangGuo mYangGuo = new YangGuo();
    HongQiGong mHongQiGong = new HongQiGong(mYangGuo);
    mHongQiGong.attackMagic(); 
    
- 使用组合，动态地扩展对象的功能，在运行时能够使用不同的装饰器实现不同的行为。
- 比继承更易出错，旨在必要时使用。

#### 外观模式（门面模式）

一个子系统的内部和外部通信必须通过一个统一的对象进行。即提供一个高层的接口，方便子系统更易于使用。

外观模式有以下角色：

- 外观类：将客户端的请求代理给适当的子系统对象。
- 子系统类：可以有一个或多个子系统，用于处理外观类指派的任务。注意子系统不含外观类的引用。

##### 简单示例

1、子系统类（这个有三个子系统）

    public class ZhaoShi {
        public void TaiJiQuan() {
            ...
        }
        
        public void QiShanQuan() {
            ...
        }
        
        public void ShengHuo() {
            ...
        }
    }
    
    public class NeiGong {
        public void JiuYang() {
            ...
        }
        
        public void QianKun() {
            ...
        }
    }
    
    public class JingMai {
        public void JingMai() {
            ...
        }
    }
    
2、外观类

    public class ZhangWuJi {
        private ZhaoShi zhaoShi;
        private JingMai jingMai;
        pirvate Neigong neiGong;
        
        public ZhangWuJi() {
            zhaoShi = new ZhaoShi();
            jingMai = new JingMai();
            neiGong = new NeiGong();
        }
        
        public void qianKun() {
            jingMai.JingMai();
            neiGong.QianKun();
        }
        
        public void qiShang() {
            jingMai.JingMai();
            neiGong.JiuYang();
            zhaoShi.QiShangQuan();
        }
    }

3、使用

    ZhangWuJi zhangWuJi = new ZhangWuJi();
    zhangWuJi.QianKun();
    zhangWuJi.QiShang();
    
- 将对子系统的依赖转换为对外观类的依赖。
- 对外部隐藏子系统的具体实现。
- 这种外观特性增强了安全性。

#### 享元模式

使用共享对象有效支持大量细粒度（性质相似）的对象。

额外的两个概念：

- 1、内部状态：共享信息，不可改变。
- 2、外部状态：依赖标记，可以改变。

享元模式有以下角色：

- 抽象享元角色：定义对象内部和外部状态的接口。
- 具体享元角色：实现抽象享元角色的任务。
- 享元工厂：管理对象池及创建享元对象。

##### 简单示例

1、抽象享元角色

    public interface IGoods {
        public void showGoodsPrice(String name);
    }

2、具体享元角色

    public class Goods implements IGoods {
        private String name;
        private String price;
        
        Goods (String name) {
            this.name = name;
        }
        
        @Override
        public void showGoodsPrice(String name) {
            ...
        }
    }

3、享元工厂

    public class GoodsFactory {
        private static Map<String, Goods> pool = new HashMap<String, Goods>();
        public static Goods getGoods(String name) {
            if (pool.containsKey(name)) {
                return pool.get(name);
            } else {
                Goods goods = new Goods(name);
                pool.put(name, goods);
                return goods;
            }
        }
    }

4、使用

    Goods goods1 = GoodsFactory.getGoods("Android进阶之光");
    goods1.showGoodsPrice("普通版");
    Goods goods2 = GoodsFactory.getGoods("Android进阶之光");
    goods1.showGoodsPrice("普通版");
    Goods goods3 = GoodsFactory.getGoods("Android进阶之光");
    goods1.showGoodsPrice("签名版");
    
goods1为新创建的对象，后面的都是从对象池中取出的缓存对象。


#### 适配器模式

将一个接口转换为另一个需要的接口。

适配器有以下角色：

- 要转换的接口。
- 要转换的接口的实现类。
- 转换后的接口。
- 转换后的接口的实现类。
- 适配器类。


##### 简单示例

1、要转换的接口（火鸡）

    public interface Turkey {
        public void gobble();
        public void fly();
    }

    

2、要转换的接口的实现类

    public class WildTurkey implements Turkey {
        @Override
        public void gobble() {
            ...
        }
        
        @Override
        public void fly() {
            ...
        }
    }

3、转换后的接口（鸭子）

    public interface Duck {
        public void quack();
        public void fly();
    }

4、转换后的接口的实现类。

    public class MallardDuck implements Duck {
        @Override
        public void quack() {
            ...
        }
        
        @Overrdie
        public void fly() {
            ...
        }
    }

5、适配器类

    public class TurkeyAdapter implements Duck {
        Turkey turkey;
        
        public TurkeyAdapter(Turkey turkey) {
            this.turkey = turkey;
        }
        
        @Override
        public void quack() {
            turkey.gobble();
        }
        
        @Override
        public void fly() {
            // 火鸡没有鸭子飞的远，因此多飞几次，达到适配鸭子fly的作用
            for(int i;i < 5;i++) {
                turkey.fly();
            }
        }
    }
    
6、使用

    WildTurkey wildTurkey = new WildTurkey();
    TurkeyAdapter turkeyAdapter = new TurkeyAdapter(wildTurkey);
    turkeyAdapter.quack();
    turkeyAdapter.fly();
    
- 注重适度使用即可。

### 3、行为型设计模式

#### 1、策略模式

定义一系列的算法，将每一个算法都封装起来，并且可相互替换。这使得算法可以独立于调用者而单独变化。

策略模式有以下角色：

- 上下文角色：用来操作策略使用的上下文环境。屏蔽了高层模块对策略和算法的直接访问。
- 抽象策略角色。
- 具体策略角色。

##### 简单示例

1、抽象策略角色

    public interface FightingStrategy {
        public void fighting();
    }
    
2、具体策略角色

    public class WeakRivalStrategy implements FightingStrategy {
        
        @Override
        public void fighting() {
            ...
        }
    }
    
    public class CommonRivalStrategy implements FightingStrategy {
        
        @Override
        public void fighting() {
            ...
        }
    }
    
    public class StrongRivalStrategy implements FightingStrategy {
        
        @Override
        public void fighting() {
            ...
        }
    }

3、上下文角色

    public class Context {
        private FightingStrategy mFightingStrategy;
        
        public void Context(FightingStrategy fightingStrategy) {
            this.mFightingStrategy = fightingStrategy;
        }
        
        public void fighting() {
            mFightingStrategy.fighting();
        }
    }
    
4、使用

    Context context;
    context = new Context(new WeakRivalStrategy());
    context.fighting();
    context = new Context(new CommonRivalStategy());
    context.fighting();
    context = new Context(new StrongRivalStategy());
    context.fighting();
    
- 隐藏具体策略中算法的实现细节。
- 避免使用多重条件语句。
- 易于扩展
- 每一个策略都是一个类，复用性小。
- 上层模块必须知道有哪些策略类，与迪米特原则相违背。


#### 2、模板方法模式

定义了一套算法框架，将某些步骤交给子类去实现。使得子类不需改变框架结构即可重写算法中的某些步骤。

模板方法模式有以下角色：

- 抽象类：定义了一套算法框架。
- 具体实现类。

##### 简单示例

1、抽象类

    public abstract class AbstractSwordsman {
    
        public final void fighting() {
            neigong();
            
            // 这个是具体方法
            jingmai();
            
            if (hasWeapons()) {
                weapons();
            }
            
            moves();
            
            hook();
        }
        
        protected void hook() { };
        protected void abstract neigong();
        protected void abstract weapons();
        protected void abstract moves();
        public void jingmai() {
            ...
        }
        
        protected boolean hasWeapons() {
            return ture;
        }
    }
    
2、具体实现类

    public class ZhangWuJi extends AbstractSwordsman {
        
        @Override
        public void neigong() {
            ...
        }
        
        @Override 
        public void weapons() {
            // 没有武器，不做处理
        }
        
        @Override 
        public void moves() {
            ...
        }
        
        @Override
        public boolean hasWeapons() {
            return false;
        }
    }
    
    punlc class ZhangSanFeng extends AbstractSwordsman {
        
        @Override
        public void neigong() {
            ...
        }
        
        @Override 
        public void weapons() {
            ...
        }
        
        @Override 
        public void moves() {
            ...
        }
        
        @Override
        public void hook() {
            // 额外处理
            ...
        }
    }
    
3、使用

    ZhangWuJi zhangWuJi = new ZhangWuJi();
    zhangWuJi.fighting();
    ZhangSanFeng zhangSanFeng = new ZhangSanFeng();
    zhangSanFeng.fighting();
    
    
- 可以使用hook方法实现子类对父类的反向控制。
- 可以把核心或固定的逻辑搬移到基类，其它细节交给子类实现。
- 每个不同的实现都需要定义一个子类，复用性小。


#### 3、观察者模式（发布 - 订阅模式）

定义对象间的一种1对多的依赖关系，每当这个对象的状态改变时，其它的对象都会接收到通知并被自动更新。

观察者模式有以下角色：

- 抽象被观察者：将所有已注册的观察者对象保存在一个集合中。
- 具体被观察者：当内部状态发生变化时，将会通知所有已注册的观察者。
- 抽象观察者：定义了一个更新接口，当被观察者状态改变时更新自己。
- 具体被观察者：实现抽象观察者的更新接口。

##### 简单示例

1、抽象观察者

    public interface observer {
        
        public void update(String message);
    }
    
2、具体观察者

    public class WeXinUser implements observer {
        private String name;
        
        public WeXinUser(String name) {
            this.name = name;
        }
        
        @Override
        public void update(String message) {
            ...
        }
    }
    
3、抽象被观察者

    public interface observable {
        
        public void addWeXinUser(WeXinUser weXinUser);
        
        public void removeWeXinUser(WeXinUser weXinUser);
        
        public void notify(String message);
    }
    
4、具体被观察者

    public class Subscription implements observable {
        private List<WeXinUser> mUserList = new ArrayList();
        
        @Override
        public void addWeXinUser(WeXinUser weXinUser) {
            mUserList.add(weXinUser);
        }
        
        @Override
        public void removeWeXinUser(WeXinUser weXinUser) {
            mUserList.remove(weXinUser);
        }
        
        @Override
        public void notify(String message) {
            for(WeXinUser weXinUser : mUserList) {
                weXinUser.update(message);
            }
        }
    }
    
5、使用

    Subscription subscription = new Subscription();
    
    WeXinUser hongYang = new WeXinUser("HongYang");
    WeXinUser rengYuGang = new WeXinUser("RengYuGang");
    WeXinUser liuWangShu = new WeXinUser("LiuWangShu");
    
    subscription.addWeiXinUser(hongYang);
    subscription.addWeiXinUser(rengYuGang);
    subscription.addWeiXinUser(liuWangShu);
    subscription.notify("New article coming");
    

- 实现了观察者和被观察者之间的抽象耦合，容易扩展。
- 有利于建立一套触发机制。
- 一个被观察者卡顿，会影响整体的执行效率。采用异步机制可解决此类问题。



