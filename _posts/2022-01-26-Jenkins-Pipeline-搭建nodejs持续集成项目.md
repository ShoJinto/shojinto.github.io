---
title: jenkins-pipeline搭建nodejs持续集成项目
date: 2022-01-26 13:38:03 +0800
categories: [运维, 环境搭建]
tags: [运维]
author: 
  name: ShoJinto
  link: https://shojinto.github.io
---
# Jenkins-pipeline + gitlab + nodejs 自动发布vue项目

### 环境

OS | SOFTWARE | NOTE
:---:|:---:|:---:
CentOS7.4|Jenkins2.327 |
  x |nodejs16.3.2 | 实施版本`15.14.0`
  x |gitlab13.2.0 |

### 操作步骤
#### 1. 编写`Pipeline`脚本

公司新上项目前端项目采用`VUE`写成，需要接入`Jenkins-Pipeline`进行持续集成与发布。由于`Jenkins`上还没有`nodejs`环境所以需要配置。不上不知道以上才知道。先写好`Pipeline`脚本：
```java
pipeline {
    agent any
	
	environment {
		serviceName = "compoment-official"
	}
    
	tools {
        nodejs "nodejs"
    }

    stages {
        stage('Preparation codes') {
            steps {
				dir("${serviceName}") {
					git branch: "develop", credentialsId: 'ad3a4389-3f1f-42c8-86fe-aee55f362a8e', url: 'git@git.gitrepostory.com:compoment-web/compoment-official.git'
				}
            }
        }

        stage('Install') {
            steps {
				dir("${serviceName}") {
					sh '''
					source /opt/rh/devtoolset-7/enable
					npm config set python /usr/local/python3/bin/python3
					npm config set registry https://registry.npm.taobao.org
					npm install
					'''
				}
            }
        }

        stage('Build') {
            steps {
				dir("${serviceName}") {
					sh """
					npm run build:test
					"""
                }
            }
        }
		stage('Build ansible-playbook') {
			steps {
				script {
					sh """
					rm -rf roles/${serviceName} # 清理release
					mv roles/projectname roles/${serviceName}
					cd ${serviceName}/dist/
					zip -9rv ${WORKSPACE}/roles/${serviceName}/files/${serviceName}.zip *
					cd ${WORKSPACE}
					sed -i 's/groupname/${serviceName}/' hosts
					sed -i 's/\\(projectname: \\).*/\\1${serviceName}/g' site.yml
					sed -i 's/\\(programfile: \\).*/\\1${serviceName}.zip/g' site.yml
					"""
				}
			}
		}

        stage('Deploy To Remote Host') {
            steps {
				script {
					sh "ansible-playbook -i hosts site.yml"
				}
            }
        }
    }
}

```
#### 2. 解决`canvas`依赖问题

`Jenkins`控制面板上一跑才发现，错误一大堆！！！
```bash
+ npm config set python /usr/1ocal/python3/bin/python3
+ npm --registry=https://registry. npm. taobao. org install
npm WARN deprecated 9hapi/topo03. 1.6: This version has been deprecated and is no longer supported or maintained
.....
npm WARN deprecated core-js02.6.12: core-js0<3.3 is no longer maintained and not recommended for usage due to the number of issues. Because of the V8 engine whims， feature detection in old core-js versions could Ci
to the actual version of core-js.
npm ERR! code 1
npm BRR! path /var/1ib/ jenkins/ jobs/tfxing-official-dev/workspace/ode_modu1es/canvas
apm ERR! command failed  # 关键错误信息
ipm ERR! command sh -c node-pre-gyp install -- fallback-to-build
npm BRR! Failed to execute '/var/1ib/ jenkins/tools/ jenkins. plugins. nodejs. tools. NodeJSInstallation/nodejs/bin/node /var/1ib/ jenkins/tools/ jenkins. plugins. nodejs. tools. NodeJSInstallation/nodejs/1ib/node. modules/npm
module=/var/lib/ jenkins/ jobs/tfxing-official-dev/workspace/node_ _modules/canvas/bui1d/Release/canvas. node --module_name=canvas --module_ path=/var/1ib/ jenkins/ jobs/tfxing-official-dev/workspace/node_modules/canvas/bl
node_ napi 1abel=node-v93 --python=/usr/1ocal/python3/bin/python3’ (1)
npm ERR! node-pre-gyp info it worked if it ends with ok
.....
```
怎么会出现`canvas`模块无法安装呢？明明项目的`package.json`中已经把相关依赖都安装上了。重试多吃无果，只能对依赖模块进行单独安装。但仍旧报错：
```bash
+ npm config set python /usr/1ocal/python3/bin/python3
npm config set registry https://registry. npm. taobao. org
+ npm install canvas02.9.0 # 单独安装canvas
npm WARN deprecated @hapi/topo03.1.6: This version has been deprecated and is no longer supported or maintained
npm WARN deprecated Chapi/bourne01.3.2: This version has been deprecated and is no longer supported or maintained
npm WARN deprecated urix0.1.0: Please see https://github. com/1yde11/urix#deprecated
npm WARN deprecated har-validator05.1.5: this library is no 1onger supported
npm WARN deprecated eslint-loader02.2. 1: This loader has been deprecated. Please use eslint-webpack-plugin
npm WARN deprecated resolve-ur100.2. 1: https://github. com/lyde1l/resolve-ur1#deprecated
npm WARN deprecated chokidar02.1.8: Chokidar 2 will break on node v14+. Upgrade to chokidar 3 with 15x less dependencies.
npm WARN deprecated chokidar02. 1.8: Chokidar 2 will break on node v14+. Upgrade to chokidar 3 with 15x less dependencies.
npm WARN deprecated querystring0.2.0: The querystring API is considered Legacy. new code should use the URLSearchParams API instead.
npm WARN deprecated html-webpack-plugin03.2.0: 3.x is no longer supported
npm WARN deprecated babel-eslint010.1.0: babel-eslint is now @babel/eslint-parser. This package will no longer receive updates.
npm WARN deprecated Qhapi/address02. 1.4: Moved to。 npm install esideway/address '
npm WARN deprecated uuid03.4.0: Please upgrade to version 7 or higher. 01der versions may use Math. random() in certain circumstances, which is known to be problematic. See https://v8. dev/b1og/math-random :
npm WARN deprecated request02. 88.2: request has been deprecated， see https://github. com/request/request/issues/3142
npm WARN deprecated Qhapi/hoek08.5.1: This version has been deprecated and is no 1onger supported or maintained
npm WARN deprecated Qhapi/joi015.1.1: Switch to 'npm install joi'
npm WARN deprecated svg01.3.2: This SVGO version is no longer supported. Upgrade to v2.x.x.
npm WARN deprecated core-js02. 6.12: core-js0<3.3 is no longer maintained and not recommended for usage due to the number of issues. Because of the V8 engine whims，feature detection in old core-js versions could
to the actual version of core-js.
npm BRR! code 1
npm BRR! path /var/1ib/ jenkins/jobs/tfxing-official-dev/workspace/node_modules/node-sass # 此时又报依赖node-sass模块错误
npm ERR! command failed
npm ERR! command sh -c node scripts/build. js
```
#### 3. 解决`node-sass`依赖

没办法继续对`node-sass`模块进行独立安装，但仍旧报错：
```bash
+ npm install --python=/usr/bin/python ，node-sass0^4.0.0'
npm WARN deprecated Chapi/topo@3. 1.6: This version has been deprecated and is no longer supported or maintained
npm WARN deprecated Chapi/bourne01. 3.2: This version has been deprecated and is no longer supported or maintained
....
npm ERR! gyp info spawn args [ 'BUILDTYPE=Release', ' -C*，’huild' 1
npm ERR! g++: error: unrecognized command line option( ‘-std=gnu++14’ # 此处是关键错误信息
npm ERR! make: *** [Release/obj. target/libsass/src/libsass/src/ast.o] Error 1
npm ERR! gyp ERR! build error
npm ERR! gyp ERR! stack Error:
make failed with exit code: 2
....
```
#### 4. 解决`gnu++14`依赖
到此处问题开始明朗起来了！！！再接再厉。
从错误信息中可以得知系统上没有`gnu++14`，原因很简单`CentOS7.4`默认的`GCC4.8`不支持，需要升级GCC。在网上找到一篇说得很清楚的问文章:[centos 升级gcc - tycoon3 - 博客园 (cnblogs.com) ](https://www.cnblogs.com/dream397/p/14148796.html) 根据这篇文章升级到`GCC7`——查看`devtoolset`源可得知目前其最低版本已经到7最高到11。
```bash
> sudo yum list |grep gcc
...
devtoolset-10-gcc.x86_64                   10.2.1-11.2.el7        centos-sclo-rh
devtoolset-10-libgccjit-devel.x86_64       10.2.1-11.2.el7        centos-sclo-rh
devtoolset-10-libgccjit-docs.x86_64        10.2.1-11.2.el7        centos-sclo-rh
...
devtoolset-11-annobin-plugin-gcc.x86_64    9.82-1.el7.1           centos-sclo-rh
devtoolset-11-libgccjit-devel.x86_64       11.2.1-1.2.el7         centos-sclo-rh
devtoolset-11-libgccjit-docs.x86_64        11.2.1-1.2.el7         centos-sclo-rh
...
devtoolset-7-gcc-gdb-plugin.x86_64         7.3.1-5.16.el7         centos-sclo-rh
devtoolset-7-gcc-gfortran.x86_64           7.3.1-5.16.el7         centos-sclo-rh
devtoolset-7-gcc-plugin-devel.x86_64       7.3.1-5.16.el7         centos-sclo-rh
...
devtoolset-8-gcc.x86_64                    8.3.1-3.2.el7          centos-sclo-rh
devtoolset-8-gcc-c++.x86_64                8.3.1-3.2.el7          centos-sclo-rh
devtoolset-8-libgccjit-docs.x86_64         8.3.1-3.2.el7          centos-sclo-rh
...
devtoolset-9-libgccjit.x86_64              9.3.1-2.2.el7          centos-sclo-rh
devtoolset-9-libgccjit-devel.x86_64        9.3.1-2.2.el7          centos-sclo-rh
devtoolset-9-libgccjit-docs.x86_64         9.3.1-2.2.el7          centos-sclo-rh
...
```
这里直接安装devtoolset-7
```bash
yum install -y devtoolset-7-gcc devtoolset-7-gcc-c++
```

值得说明的是这些软件包可以同时安装，不会相互覆盖和冲突，也不会覆盖系统的版本。即可以在系统中可同时存在gcc 6, gcc 7, gcc 8等多个版本。

因为不会覆盖系统默认的gcc，使用这些软件的方法有四种：

1. 使用绝对路径；
2. 添加可执行文件路径到PATH环境变量；
3. 使用官方推荐的加载命令：`scl enable devtoolset-x bash`, x为要启用的版本;
4. 执行安装软件自带的脚本： `source /opt/rh/devtoolset-x/enable`，x为要启用的版本。

实践推荐使用最后两种方式。例如启用gcc 6: `source /opt/rh/devtoolset-6/enable`，接着输入gcc -v查看版本已经变成gcc 6.3.1。如果希望长期使用某个高版本，可将此命令写入.bashrc等配置文件。

以上说明引用自[centos 升级gcc - tycoon3 - 博客园 (cnblogs.com)](https://www.cnblogs.com/dream397/p/14148796.html)

经过测试在`Jenkins-Pipeline`中使用：`source /opt/rh/devtoolset-x/enable` 才有效

#### 5. 解决`python3`的依赖

此时还要注意组件编译过程中需要python3.6+的支持:
```bash
...
npm ERR! gyp ERR! find Python checking Python explicitly set from command line or npm configuration
npm ERR! gyp ERR! find Python -”- - python=”or' 'npm config get python” is" /usr/bin/python'
npm ERR! 8yp ERR! find Python - executable path is" /usr/bin/python
npm ERR! gyp ERR! find Python - version is "2. 7.5"
npm ERR! gyp ERR! find Python - version is 2.7.5 - should be >=3.6.0.
npm ERR! gyp ERR! find Python - THIS VBRSION OF PYTHON IS NOT SUPPORTED
npm ERR! gyp ERR! find Python Python is not set from environment variable PYTHON
npm ERR! gyp ERR! find Python checking if "python3” can be used
npm ERR! gyp ERR! find Python -" python3” is not in PATH or produced an error
npm ERR! gyp ERR! find Python checking if" python" can be used
npm ERR! gyp ERR! find Python - executable path is" /bin/python'
npm ERR! gyp ERR! find Python - version is "2. 7.5”
npm ERR! 8yp ERR! find Python - version is 2. 7.5 - should be >=3. 6.0
npm ERR! gyp ERR! find Python - THIS VERSION 0F PYTHON IS NOT SUPPORTED
npm ERR! gyp ERR! find Python
npm ERR! gyp ERR! find Python **********************************************
npm ERR! gyp ERR! find Python You need to install the latest version of Python.
npm BRR! 8Yp ERR! find Python Node-gyp should be able to find and use Python. If not,
npm ERR! gyp ERR! find Python you can try one of the following options:
npm ERR! gyp ERR! find Python - Use the switch --python=" /path/to/pythonexecutable"
npm ERR! gyp ERR! find Python
(accepted by both node-gyp and npm)
npm ERR! gyp ERR! find Python - Set the environment variable PYTHON
npm ERR! 8yP ERR! find Python - Set the npm configuration variable python:
npm ERR! gyp ERR! find Python
npm config set python "/path/to/pythonexecutable"
npm ERR! gyp ERR! find Python For more information consult the documentation at:
npm ERR! gyp ERR! find Python https://github. com/nodejs/node-gyp#installation
npm ERR! gyp ERR! find Python *******************************************
nom ERR1 gvp ERR1 find Pvthon
...
```
因此只需要在执行`install`之前机上如下代码即可解决`python3`依赖的问题
```bash
+ npm config set python /usr/1ocal/python3/bin/python3
+ npm --registry-https:/ / registry. npm. taobao. org install
```
#### 6. 解决`nodejs`与`node-sass`版本匹配的问题

另外对nodejs的版本也有要求，不然就会报如下错误：
```bash
npm EKK! /var/ 11b/ jenkins/. cache/noae-gyp/ 10.13. 2/1ncluae/ noae/v8-1nternal. h:492:58: note: suggestea alternative:" remove_cv
npm BRR! 
!std::is_ same<Data， std::remove_Cv_t<T>>::value>::Perform(data) :
npm BRR
npm BRR!
remove_ _cv
npm ERR! /var/1ib/jenkins/. cache/node-gyp/16. 13. 2/include/node/v8-internal. h:492:38: error: 'remove_ _cv_t’is not a member of 'std' 
npm ERR! /var/lib/ jenkins/. cache/node-gyp/16. 13. 2/include/node/v8-interna1. h:492:38: note: suggested alternative:‘ remove_cv'
npm ERR
!std::is_same<Data， std::remove_cv_t<T>>::value>::Perform(data) ;
npm BRR
npm ERR!
remove_cv
npm ERR! /var/1ib/ jenkins/.cache/node-gyp/16. 13. 2/ include/node/v8-internal. h:492:50: error: template argument 2 is invalid
rpm RR!
!std::is_ same<Data， std::remove_cv_t<T>>::value>::Perform (data) ;
npm BRR!
npm BRR! /var/lib/jenkins/. cache/node-gyp/16. 13. 2/include/node/v8-internal. h:492:63: error: ‘::Perform’ has not been declared
npm ERR!
!std::is_ same<Data，std::remove_cv_t<T>>::value>::Perform(data) :
npm BRR!
npm BRR! /var/lib/ jenkins/.cache/node-gyp/16.13. 2/include/node/v8-internal. h:492:63: note: suggested alternative: herror '
npm ERR!
!std::is_ same<Data， std::remove_Cv_t<T>>::value>: :Perform(data) :
npm ERR!
npm ERR!
herror
npm BRR! make: *** [Release/obj. target/binding/src/binding.o] Error 1 # 此处为关键错误信息
npm BRR! gYP ERR! build error
npm BERR! gyp ERR! stack Brror:、 make failed with exit code: 2
npm ERR! gyp ERR! stack
at ChildProcess. onExit (/var/1ib/ jenkins/ jobs/ tfxing-official-dev/workspace/node_ modules/node-gyp/1ib/build. js:194:23)
npm BRR! gyp BRR! stack
npm ERR! gyp BRR! stack
at Process. ChildProcess.. _handle.onexit (node:internal/chi1d_ process:290:12) .
npm BRR! gyp ERR! System Linux 3. 10.0-327.36. 3. e17.x86_ _64
npm ERR! BYP ERR! command' /var/1ib/jenkins/tools/ jenkins. plugins. nodejs. tools. NodeJSInstallation/nodejs/bin/node”" /var/lib/ jenkins/ jobs/tfxing-official-dev/workspace/node. _modules/node-8yp/bin/node-gYP
npm ERR! gyp ERR! cwd /var/1ib/jenkins/ jobs/tfxing-official-dev/workspace/node_ modules/node-sass
npm ERR! 8yp BERR! node -v v16. 13.2
npm BRR! gyp BRR! node-gyp -v v7.1.2
npm ERR! gyp ERR! not ok
npm ERR! Build failed with error code: 1
```
遇事不决问Google：谷歌出奇迹！！！在这里：[https://www.codenong.com/cs122036023/](https://www.codenong.com/cs122036023/)
找到一张`node`与`node-sass`的匹配图：
![](https://i2.wp.com/img-blog.csdnimg.cn/img_convert/5d3fd2526f22a1c25768eb46a9ccb290.png)

出处: [https://github.com/sass/node-sass](https://github.com/sass/node-sass)

再通过项目中的`packages.json`文件找到`node-sass`的版本是`5.0.0`因此正确的`nodejs`版本是`15`，需要对`nodejs`进行降级。
降级之后仍然报错！！此时头顶飘过一万条`草泥马、草泥马、草泥马、草泥马`！！！
```bash
npm ER! Package pixman-1 was not found in the pkg-config search path.
npm ER! Perhaps you should add the directory containing pixman- 1.pc'
npm ERR! to the PKG_CONFIG_ PATH envir onment variable
npm ERR! No package'pixman-1’ found # 此为何方妖物呀！！！:( :(
npm ERR! gyP: Call to’ pkg-config pixman-1 --libs’ returned exit status 1 while in binding. gyp. while trying to load binding. gyPp
npm ERR! gyP ERR! configure error
npm ERR! gyP ERR! stack Error:、 gyP failed with exit code: 1
```
No package 'pixman-1' found ？ 什么鬼。。。。

在StackOverflow上有人已经回答了：

https://stackoverflow.com/questions/64562563/how-can-i-ovecome-this-error-in-npm-install 

跟随跳转：

https://github.com/Automattic/node-canvas

在`node-canvas`的GitHub仓库中找到解决方案：

`sudo yum install gcc-c++ cairo-devel pango-devel libjpeg-turbo-devel giflib-devel`

最终编译成功：
```bash
gcc version 7.3. 120180303 (Red Hat 7.3. 1-5) (GCC)
+ npm config set python /usr/1ocal/ python3/bin/ python3
+ npm config set registry https:/fregistry. npm. taobao. org
+ npm install
npm WARN deprecated @hapi/topo@3. 1. 6: This version has been deprecated and is no longer supported or maintained
npm WARN deprecated @hapi/bourne@1.3. 2: This version has been deprecated and is no longer supported or maintained
npm WARN deprecated urix@0. 1.0: Please see https: 1{ github. com/ 1ydell/ur ix#deprecated
npm WARN deprecated har- -validator@5. 1.5: this library is no longer supported
npm WARN deprecated eslint- loader@2.2. 1: This loader has been deprecated. Please use eslint- -webpack- plugin
npm WARN deprecated resolve- url@0. 2.1: https: 11 github. com/ 1yde11/ resolve- -url#deprecated
npm WARN deprecated chokidar@2. 1.8: Chokidar 2 will break on node v14+. Upgrade to chokidar 3 with 15x less dependencies.
npm WARN deprecated chokidar@2. 1.8: Chokidar 2 will break on node v14+. Upgrade to chokidar 3 with 15x less dependencies.
npm WARN deprecated querystring@0. 2.0: The querystring API is considered Legacy. new code should use the URL SearchParams API instead.
npm WARN deprecated html-webpack- -plugin@3. 2.0: 3.x is no longer supported
npm WARN deprecated babel-eslint@10. 1.0: babel-eslint is now @babel/eslint-parser. This package will no longer receive updates.
npm WARN deprecated @hapi/ address@2. 1.4: Moved to’npm install @sideway/ address'
npm WARN deprecated uuid@3. 4. 0: Please upgrade
to version 7 or higher.
01der versions may use Math. random() in certain circumstances
npm WARN deprecated request@2. 88. 2: request has been deprecated, see https: //github. com/ request/ request/issues/3142
npm WARN deprecated @hapi/hoek@8.5. 1: This version has been deprecated and is no longer supported or maint ained
npm WARN deprecated @hapi/joi@15. 1. 1: Switch to’npm install joi'
npm WARN deprecated svgo@1. 3.2: This SVGO version is no longer supported. Upgrade to v2. x. x.
npm WARN deprecated core- js@2. 6.12: core-js@<3.3 is no longer maintained and not recommended for usage due to the number of issues. Be
to the actual version of core- js.
added 1475 packages in 4m
87 packages are looking for funding
run、npm fund~ for details
```
哎呀！妈呀！！这下成功了！！！

### 总结

喜大普奔，总结下经验教训：

`nodejs`相关的项目编译，需要以来的东西大致有：`GCC=5+` 、`Python3.6+` 、（"`sudo yum install gcc-c++ cairo-devel pango-devel libjpeg-turbo-devel giflib-devel`" 如果项目没有`node-canvas`依赖应该可以跳过）。编译`nodejs`项目查看日志错误信息，`从下往上`更有助于发现问题的根源。就拿本次经历来说：根源问题还是编译node-sass的时候所依赖的
```bash
npm ERR! gyp info spawn args [ 'BUILDTYPE=Release', '-C', 'build' ]
npm ERR! g++: error: unrecognized command line option ‘-std=gnu++14’
```
解决了这个报错后面nodejs与node-sass版本匹配度的问题才会浮出水面，到最后node-canvas的系统依赖库。将这些解决了编译才最终完成。 