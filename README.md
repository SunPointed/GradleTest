# GradleTest

## 目标
```
1.对相应配置文件进行配置后，执行gradle build就完成项目中所有aar到代码仓库的上传且成功build
```
## 说明
```
1.提供一种build与uploadArchives共存的思路
2.使用见 说明.txt
3.本项目只用来说明，并没有真正上传代码仓库
```
## 项目结构及用途

### 项目结构如下('-'代表依赖)
![5.png](https://github.com/SunPointed/GradleTest/blob/master/img/5.png)
<br>
在这个项目中，library_1和library_2两个module代表最终会提供给第三方的两个aar；
以library_1为例，由于它会被打包为aar（作为sdk提供给第三方使用），它在项目中依赖base，business，other；
依赖module直接打包aar的话，在别的项目中引入library_1的aar后会提示找不到base，business，other这三个project，所以在我们
打包library_1的aar时，base，business，other这三个module不能直接依赖project（如implementation project(':base')），
需要先将这三个module打包为相应aar然后再在library_1引入（如implementation 'lqy.com.graldetest:base:xxx'），再打包library_1
的aar；
这样在别的项目中使用library_1的sdk（aar）就能找到所有的依赖。

### 集成、调试、上传
现在另一个项目（假设为LibraryTest），LibraryTest需要使用library_1和library_2的sdk（aar），然后对两个sdk进行一些测试（注，
假设此处因为某些原因不能在GradleTest的app中进行测试）；
当测试出某个问题时，在GradleTest中修改完成后，根据修改的module不同，重新打包library_1和library_2的sdk（aar）的情况如下：
```
    1.修改library_1或library_2 -》 打包library_1或library_2相应module的aar -》
      修改LibraryTest使其依赖新的aar
    2.修改other -》 打包other module对应的aar -》修改library_1和library_2使其依赖新的other的aar -》打包
      library_1和library_2相应module的aar  -》 修改LibraryTest使其依赖新的aar
    3.修改business（情况和other一致）
    4.修改base -》 打包base module对应的aar -》 修改business使其依赖新的base的aar
      -》 打包business module对应的aar -》修改library_1和library_2使其依赖新的base
      和business的aar -》打包library_1和library_2相应module的aar  -》 修改LibraryTest使其依赖新的aar
```
其中每次打包aar，都需要修改版本号，再手动执行（如上传base命令 gradle -p clean build uploadArchives），可见这个修改会非常的频繁；
就算直接在GradleTest的app中测试，发布Release版本aar时，也需要这样根据相互的依赖关系逐个module打包aar，非常繁琐。

### 思路
#### 1.依赖处理
pre:<br> 
```
    implementation project(':base')
    //implementation 'lqy.com.graldetest:base:xxx'
    //implementation 'lqy.com.graldetest:base:xxx-SNAPSHOT'
```

问题：有的时候需要依赖project，有的时候需要依赖aar，aar又分为Debug还是Release<br>
思考：首先思考的是在一个统一的地方，通过设置布尔变量确定当前项目是依赖project还是aar；再设置一个布尔变量区分使用Debug还是Release得aar<br>
idea: 
```
implementation useProject ? project(':base') : ( isRelease ? 'lqy.com.graldetest:base:xxx' : 'lqy.com.graldetest:base:xxx-SNAPSHOT')
```
解决：因为gradle能直接拿到gradle.properties文件中的属性，所以在该文件中添加两个变量useProject和isRelease，但是取得的值是String，所以需要转换成布尔值<br>
now: 
```
implementation Boolean.valueOf(useProject) ? project(':base') : ( Boolean.valueOf(isRelease) ? 'lqy.com.graldetest:base:xxx' : 'lqy.com.graldetest:base:xxx-SNAPSHOT')
```
#### 2.版本处理
pre: 'lqy.com.graldetest:base:xxx'和'lqy.com.graldetest:base:xxx-SNAPSHOT'这样<br>
问题：每次版本变化都需要到使用到相应aar的gradle文件中逐一修改<br>
思考：在一个地方把所有需要打包为aar的module的版本号统一管理，当然要区分Debug和Release<br>
解决：在gradle.properties中添加相应版本变量，如下：<br>
![1.png](https://github.com/SunPointed/GradleTest/blob/master/img/1.png)
<br>
在根项目的build.gradle中添加ext，如下：<br>
![2.png](https://github.com/SunPointed/GradleTest/blob/master/img/2.png)
<br>
相应依赖修改为：
```
    implementation Boolean.valueOf(useProject) ? project(':base') :(rootProject.ext.appPackageName + ':' + rootProject.ext.baseModuleName + ':' + rootProject.ext.baseModuleVersion)
```
同时uploadArchives修改为，如下：<br>
![3.png](https://github.com/SunPointed/GradleTest/blob/master/img/3.png)
<br>
其中username和password也在gradle.properties文件中定义
#### 3.任务执行
针对某一个module，要将代码上传到仓库需要执行[a. gradle -p 'module name' clean build uploadArchives]命令；现在想在根目录的[b. gradle clean build]中，完成a类似命令的执行
那么此时考虑在根目录build.gradle文件中添加一个task，如下：
```
        task uploadBase(type: Exec) {
            doFirst {
                println('uploadBase doFirst')
                //commandLine 'gradle', '-p base clean build uploadArchives'
                //不知为何这样直接写执行命令会报错，在base.sh中写相同命令执行这个shell脚//本就能运行，用这种方式规避这个问题
                commandLine './base.sh'
            }
            println('uploadBase configure')
            doLast {
                println('uploadBase doLast')
            }
        }
```

这样写后，确实能上传base module的aar文件，但是存在如下问题：<br>
1.由于该task在根目录下，实际上执行[gradle -p base clean build uploadArchives]<br>命令也会触发这个task，相当于会一直递归uploadArchives<br>
2.build并不会等待uploadArchives上传完毕才进行，所以会报错找不到相应aar（还没上传完成）<br>
3.自定义task之间的依赖（如base改变会导致依赖它的module都改变）<br>
4.确保每个module的uploadArchives只执行一次<br>

##### 先说问题1
在根目录创建一个upload.properties文件，内容如下：<br>
![4.png](https://github.com/SunPointed/GradleTest/blob/master/img/4.png)
<br>
然后把uploadBase修改如下：
```
        def baseProperty = new Properties()
        def file = new File('upload.properties')
        def stream = file.newDataInputStream()
        baseProperty.load(stream)
        def isBase = Boolean.valueOf(baseProperty.getProperty('base'))
        task uploadBase(type: Exec) {
            doFirst {
                baseProperty.setProperty('base', 'false')
                baseProperty.store(file.newWriter(), null)
                commandLine './base.sh'
            }
        }
        uploadBase.enabled = isBase
```

最开始从upload.properties读取isBase的值，根据isBase的值决定uploadBase是否上传，若上传，将upload.properties文件中的值改为false（即下一次执行时uploadBase.enabled = false），这样uploadBase这个task就只会执行一次

##### 针对问题2
在根目录gralde文件的allprojects中添加如下代码：
```
        afterEvaluate {
            for (def task in it.tasks) {
                if (isDependTask(task)) {
                    if (isBase) {
                        task.dependsOn(uploadBase)
                    }
                }
            }
        }
```
这样，一次gradle build中的所有任务就都会dependsOn自定义task uploadBase，即会在uploadBase完成后再执行（注：isDependTask定义[static def isDependTask(Task t) {return t.getName() != 'uploadBase'}]，防止uploadBase dependsOn自己）

##### 针对问题3
我们需要让没有依赖其他module的module最先上传（如base），有依赖的module必须在其依赖的module上传完之后再上传。结合问题2，就需要使用dependsOn了；此时不能简单的dependsOn，需要处理特殊情况，如只修改了business module，此时task uploadBusiness是不能dependsOn task uploadBase的，所以我们会用到问题1中定义的变量isBase，用它来确定uploadBusiness是否依赖uploadBase（其他module间的依赖处理相同）。所以修改后如下：
```
        task uploadBase(type: Exec) {
            doFirst {
                baseProperty.setProperty('base', 'false')
                baseProperty.store(file.newWriter(), null)

                commandLine './base.sh'
            }
        }
        uploadBase.enabled = isBase

        task uploadBusiness(type: Exec) {
            doFirst {
                baseProperty.setProperty('business', 'false')
                baseProperty.store(file.newWriter(), null)

                commandLine './business.sh'
            }
        }
        uploadBusiness.enabled = isBusiness
        if (isBase) {
            uploadBusiness.dependsOn(uploadBase)
        }

        task uploadOther(type: Exec) {
            doFirst {
                baseProperty.setProperty('other', 'false')
                baseProperty.store(file.newWriter(), null)

                commandLine './other.sh'
            }
        }
        uploadOther.enabled = isOther

        task uploadLib1(type: Exec) {
            doFirst {
                baseProperty.setProperty('library_1', 'false')
                baseProperty.store(file.newWriter(), null)

                commandLine './library_1.sh'
            }
        }
        uploadLib1.enabled = isLib1
        if (isBusiness) {
            uploadLib1.dependsOn(uploadBusiness)
        }
        if (isOther) {
            uploadLib1.dependsOn(uploadOther)
        }

        task uploadlib2(type: Exec) {
            doFirst {
                baseProperty.setProperty('library_2', 'false')
                baseProperty.store(file.newWriter(), null)

                commandLine './library_2.sh'
            }
        }
        uploadlib2.enabled = isLib2
        if (isBusiness) {
            uploadlib2.dependsOn(uploadBusiness)
        }
        if (isOther) {
            uploadlib2.dependsOn(uploadOther)
        }
```
##### 最后说问题4
先考虑base module改变需要上传aar，那么显然uploadBase，uploadBusiness，uploadLib1，uploadlib2这4个task都要执行；当执行task uploadBase中的shell脚本时，虽然不会继续触发task uploadBase（isBase为false了），但是uploadBusiness，uploadLib1，uploadlib2这3个task还会触发，这会导致重复上传，显然是不允许的；我们要求在意gralde build命令执行过程中，改动影响到的所有需要上传aar的module都只上传一次，所以此处在upload.properties文件中添加一个属性RUN_UPLOAD，每次需要上传时，把RUN_UPLOAD置为true，修改为如下：
```
        def baseProperty = new Properties()
        def file = new File('upload.properties')
        def stream = file.newDataInputStream()
        baseProperty.load(stream)

        def runUpload = Boolean.valueOf(baseProperty.getProperty('RUN_UPLOAD'))

        def isOther = Boolean.valueOf(baseProperty.getProperty('other'))
        def isBase = Boolean.valueOf(baseProperty.getProperty('base'))
        def isLib1 = Boolean.valueOf(baseProperty.getProperty('library_1'))
        def isLib2 = Boolean.valueOf(baseProperty.getProperty('library_2'))
        def isBusiness = Boolean.valueOf(baseProperty.getProperty('business'))

        if (runUpload) {
            baseProperty.setProperty('RUN_UPLOAD', 'false')
            baseProperty.store(file.newWriter(), null)

            task uploadBase(type: Exec) {
                doFirst {
                    baseProperty.setProperty('base', 'false')
                    baseProperty.store(file.newWriter(), null)

                    commandLine './base.sh'
                }
            }
            uploadBase.enabled = isBase

            task uploadBusiness(type: Exec) {
                doFirst {
                    baseProperty.setProperty('business', 'false')
                    baseProperty.store(file.newWriter(), null)

                    commandLine './business.sh'
                }
            }
            uploadBusiness.enabled = isBusiness
            if (isBase) {
                uploadBusiness.dependsOn(uploadBase)
            }

            ...
        }
```
这样保证了只有脚本中的命令不会触发任何upload task。注意doFirst闭包中的[baseProperty.setProperty('base', 'false')和baseProperty.store(file.newWriter(), null)]两行代码实际不需要了，但是为了upload.properties文件在执行一次gradle build后所有属性置为false（避免给下次gralde build前配置upload.properties文件产生困扰），保留了这两行.<br>

最后因为upload task的增减，还需做如下修改：
```
        isDependTask修改
        static def isDependTask(Task t) {
            return t.getName() != 'uploadOther' && t.getName() != 'uploadBase' && t.getName() != 'uploadBusiness' && t.getName() != 'uploadLib1' && t.getName() != 'uploadLib2'
        }
        afterEvaluate修改
        afterEvaluate {
            if (runUpload) {
                for (def task in it.tasks) {
                    if (isDependTask(task)) {
                        if (isOther) {
                            task.dependsOn(uploadOther)
                        }

                        if (isBase) {
                            task.dependsOn(uploadBase)
                        }

                        if (isBusiness) {
                            task.dependsOn(uploadBusiness)
                        }

                        if (isLib1) {
                            task.dependsOn(uploadLib1)
                        }

                        if (isLib2) {
                            task.dependsOn(uploadLib2)
                        }
                    }
                }
            }
        }
```
### 总结：虽然配置一次显得很麻烦，但是配置一次后，打包比之前方便很多，感觉解脱了；如果有新的module，也只需添加相应配置；


