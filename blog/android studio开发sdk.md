---

		title:   android studio开发sdk
		date: 2017/10/22 21:08:00   
		tags: 
		- 安卓基础
		categories: 安卓基础
		thumbnail: https://ss0.bdstatic.com/94oJfD_bAAcT8t7mm9GUKT-xh_/timg?image&quality=100&size=b4000_4000&sec=1508675820&di=84bc3dab2caf3bce2962f7d9289c182e&src=http://imgsrc.baidu.com/imgad/pic/item/6609c93d70cf3bc7814060c9db00baa1cd112a56.jpg
---

---

### 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

### 一、什么是AAR，与JAR区别

*.jar：只包含了class文件与清单文件，不包含资源文件，如图片等所有res中的文件。

*.aar：包含所有资源，class以及res资源文件全部包含。

如果你只是一个简单的类库那么使用生成的.jar文件即可；如果你的是一个UI库，包含一些自己写的控件布局文件以及字体等资源文件那么就只能使用.aar文件。

### 二、AndroidStudio将项目打包成jar包的简单方法

在build.gradle文件中，修改下面两个地方：

（1）apply plugin:'com.android.application' 改为 apply plugin: 'com.android.library' 。

(2) 将defaultConfig中的applicationID这行注释掉。

完成上述两个步骤之后，执行rebuild project,就会在app\build\intermediates\bundles\debug下生成classes.jar文件，这个文件就可以提供给其他项目使用，如果需要的话可以手动修改文件名称。


### 三、引用jar文件

1.将jar文件复制、粘贴到app的libs目录中；
 
2.右键点击jar文件，并点击弹出菜单中的“Add As Library”，将jar文件作为类库添加到项目中；   

3.选择指定的类库。
    
注：如果不执行2、3步，jar文件将不起作用，并且不能使用import语句引用。

如果要将引入的第三方包也一起打包进来，需要使用ant工具打包。

使用 ANT 工具实现 将两个或多个.jar文件合并成一个.jar文件
Apache Ant是一个基于Java的生成工具。据最初的创始人James Duncan Davidson的介绍，这个工具的名称是another neat tool(另一个整洁的工具)的首字母的缩写。

1.下载ant并配置环境变量。（如果环境变量不会配置的话，建议出门右拐）

2.在cmd命令行输入ant，检测是否配置成功。
如果出现如下内容，说明安装成功： 
Buildfile: build.xml does not exist! 
Build failed 
注意：因为ant默认运行build.xml文件，这个文件需要我们创建。 
如果不想命名为build.xml，运行时可以使用 ant -buildfile test.xml 命令指明要运行的构建文件。

3.编辑build.xml文件（命名可以随意）

<?xml version="1.0" encoding="utf-8"?>
<project
    name="hosa"                            //不用改       ,注意：这里的所有注释在 build.xml文件中 都不要有  是我为了给你们看解释写的
    basedir="H:\soft\jar"                  //生成的jar存放的位置，并且将要合并的所有.jar文件也放在该目录下
    default="makeSuperJar" >               //不用改

    <target
        name="makeSuperJar"                //不用改  
        description="description" >        //不用改

        <jar destfile="standingbanlanceFB.jar" >                  //合并后的jar文件的名称
         <zipfileset src="avoscloud-sdk-v3.14.5.jar" />           // <zipfileset >标签的都是要参与合并的子jar包 
        <zipfileset src="fastjson.jar" />
        <zipfileset src="okhttp-2.6.0-leancloud.jar" />
         <zipfileset src="standingbanlance.jar" />
          <zipfileset src="okio-1.6.0-leancloud.jar" />
         </jar>
    </target>

</project>

4.命令执行合并操作

    ant -buildfile D:\Ant\apache-ant-1.10.1\build.xml
    
注意：D:\Ant\apache-ant-1.10.1\build.xml为xml文件路径

5.打开build.xml文件，即可看到合并后的jar文件。

### 四、引用so文件

1.在“src/main”目录中新建名为“jniLibs”的目录；

2.将so文件复制、粘贴到“jniLibs”目录内。
    
注：如果没有引用so文件，可能会在程序执行的时候加载类库失败，有类似如下的DEBUG提示：
java.lang.UnsatisfiedLinkError: Couldn't load library xxxx from loader dalvik.system.PathClassLoader

## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/wexin_play.jpg" width=20%><img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/Apaliy.jpg" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/wexin_qrcode.jpg" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。
