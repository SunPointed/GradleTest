# GradleTest

## 目标
```
对相应配置文件进行配置后，执行gradle build就完成项目中所有aar到代码仓库的上传且成功build
```

## 项目结构及用途
```
项目结构如下('-'代表依赖)
app
    -base
    -business
    -other
    -library_1
    -library_2
business
    -base
library_1
    -base
    -business
    -other
library_2
    -base
    -business
    -other

在这个项目中，library_1和library_2两个module代表最终会提供给第三方的两个aar；
以library_1为例，由于它会被打包为aar（作为sdk提供给第三方使用），它在项目中依赖base，business，other；
依赖module直接打包aar的话，在别的项目中引入library_1的aar后会提示找不到base，business，other这三个project，所以在我们
打包library_1的aar时，base，business，other这三个module不能直接依赖project（如implementation project(':base')），
需要先将这三个module打包为相应aar然后再在library_1引入（如implementation 'lqy.com.graldetest:base:xxx'），再打包library_1
的aar；
这样在别的项目中使用library_1的sdk（aar）就能找到所有的依赖。
```

## 集成、调试、上传
```
现在另一个项目（假设为LibraryTest），LibraryTest需要使用library_1和library_2的sdk（aar），然后对两个sdk进行一些测试（注，
假设此处因为某些原因不能在GradleTest的app中进行测试）；
当测试出某个问题时，在GradleTest中修改完成后，根据修改的module不同，重新打包library_1和library_2的sdk（aar）的情况如下：
    1.修改library_1或library_2 -》 打包library_1或library_2相应module的aar -》 修改LibraryTest使其依赖新的aar
    2.修改other -》 打包other module对应的aar -》修改library_1和library_2使其依赖新的other的aar -》打包library_1和library_2相应module的aar  -》 修改LibraryTest使其依赖新的aar
    3.修改business（情况和other一致）
    4.修改base -》 打包base module对应的aar -》 修改business使其依赖新的base的aar -》 打包business module对应的aar -》 修改library_1和library_2使其依赖新的base和business的aar  -》打包library_1和library_2相应module的aar  -》 修改LibraryTest使其依赖新的aar
其中每次打包aar，都需要修改版本号，再手动执行（如上传base命令 gradle -p clean build uploadArchives），可见这个修改会非常的频繁；
就算直接在GradleTest的app中测试，发布Release版本aar时，也需要这样根据相互的依赖关系逐个module打包aar，非常繁琐；
```

## 思路
```
1.依赖处理
    pre: implementation project(':base')
         //implementation 'lqy.com.graldetest:base:xxx'
         //implementation 'lqy.com.graldetest:base:xxx-SNAPSHOT'
    问题：有的时候需要依赖project，有的时候需要依赖aar，aar又分为Debug还是Release
    思考：首先思考的是在一个统一的地方，通过设置布尔变量确定当前项目是依赖project还是aar；再设置一个布尔变量区分使用Debug还是Release得aar
    idea: implementation useProject ? project(':base') : ( isRelease ? 'lqy.com.graldetest:base:xxx' : 'lqy.com.graldetest:base:xxx-SNAPSHOT')
    解决：因为gradle能直接拿到gradle.properties文件中的属性，所以在该文件中添加两个变量useProject和isRelease，但是取得的值是String，所以需要转换成布尔值
    now: implementation Boolean.valueOf(useProject) ? project(':base') : ( Boolean.valueOf(isRelease) ? 'lqy.com.graldetest:base:xxx' : 'lqy.com.graldetest:base:xxx-SNAPSHOT')
2.版本处理
    pre: 'lqy.com.graldetest:base:xxx'和'lqy.com.graldetest:base:xxx-SNAPSHOT'这样
    问题：每次版本变化都需要到使用到相应aar的gradle文件中逐一修改
    思考：在一个地方把所有需要打包为aar的module的版本号统一管理，当然要区分Debug和Release
    解决：在gradle.properties中添加相应版本变量，如下 1.png；
         在根项目的build.gradle中添加ext，如下 2.png；
         相应依赖修改为：implementation Boolean.valueOf(useProject) ? project(':base') :
                             (rootProject.ext.appPackageName + ':' + rootProject.ext.baseModuleName + ':' + rootProject.ext.baseModuleVersion)
         同时uploadArchives修改为，如下 3.png 其中username和password也在gradle.properties文件中定义
3.任务执行
```


