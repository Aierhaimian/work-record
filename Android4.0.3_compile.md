# Android 4.0.3 源码编译指南

首先，在进行以下工作之前嫩，您最好可以科学上网，关于如何科学上网的方法，在此不做赘述，如有需求自行度娘。

进入安卓Source官网：[Android](https://source.android.com/setup)，仔细阅读这网站中的相关内容，只要认真阅读了，我想对于任何版本的安卓源码编译都不在话下。

在这里，我将以Android 4.0.3 源码编译为例介绍安卓源码编译的主要步骤以及我踩过的坑，并不打算详细列举其中的细节，因为这些都在官方网站中进行了详细说明。

## 环境配置

安卓源码编译环境的配置主要看这两个网页：[Requirements](https://source.android.com/setup/build/requirements)  and [Establishing a Build Environment](https://source.android.com/setup/build/initializing) 。根据您选择的安卓源码版本，一定要按照以上网页的说明进行配置，尤其是java、make、gcc等工具的版本要与您选择的安卓源码版本相匹配。

哦，对了，如果你不太清楚安卓有多少版本和分支，以及安卓源码版本和其API的对应关系，没关系，官网依然为我们做了详细的说明，看这里：[代号、标记和细分版本号](https://source.android.com/setup/start/build-numbers.html#source-code-tags-and-builds) 。

我在这里遇到的问题就是，我的环境中有多个java版本，make版本和gcc版本。因此，您可能需要：

	sudo update-alternatives --config xxx
	
进行适当版本的切换，但是对于java来说仅仅切换java、javac是不够的，还需要切换jar、javadoc、javah的版本，不然会报API过时的错误：

	******************************
	You have tried to change the API from what has been previously released in
	an SDK.  Please fix the errors listed above.
	******************************

	make: *** [out/target/common/obj/PACKAGING/checkapi-last-timestamp] 错误 38

这个时候您需要进行一下操作：

	sudo ln -s /usr/lib/jvm/jdk1.6.0_45/bin/jar  /bin/jar 
	sudo ln -s /usr/lib/jvm/jdk1.6.0_45/bin/java  /bin/java 
	sudo ln -s /usr/lib/jvm/jdk1.6.0_45/bin/javac  /bin/javac 
	sudo ln -s /usr/lib/jvm/jdk1.6.0_45/bin/javah  /bin/javah 
	sudo ln -s /usr/lib/jvm/jdk1.6.0_45/bin/javadoc  /bin/javadoc

使用这个命令切换正确的java、javac、javah、javadoc、jar版本：

	sudo update-alternatives --config xxx

然后进行一下操作重新进行源码编译：

	make clean
	rm framework/base/api/curent.txt
	make update-api
	lunch full-eng
	make -j8 > 1.log 2>&1
	
嗯，这样应该就没有问题了。

还有一个问题就是，在编译源码的时候默认会使用clang作为编译器，而官方安卓源码使用gcc进行编译，因此作以下替换：

	sudo ln -s /usr/bin/cc /usr/bin/gcc
	
环境配置这块暂时就这么多，接下来我们来看看源码下载。

## 源码下载

怎样进行安卓源码下载？当然是看这里啦：[Download Source Code](https://source.android.com/setup/build/downloading) ，不过在国内下载安卓源码还是不能从安卓官方源码仓库获取，这里再赐你一件神器——推荐清华镜像站：[AOSP](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/) ，有了此神器，下载源码便也不在话下。

## 源码编译

源码都下载下来啦，接下来就是编译咯，你问我怎么编译？当然是继续看这里啦：[Preparing to Build](https://source.android.com/setup/build/building) 。在源码编译过程中因为大家编译的版本不同，各自的环境不同，遇到的问题也不会相同，此时建议大家多多百度、goolge，总会找到问题的答案的。

## 运行模拟器

如果你编译了一个模拟器版本，就可以在你的主机上通过安卓源码自带、随着编译过程编译出来的模拟器运行你编译的安卓系统了。

	$ source build/envsetup.sh
	$ lunch full-eng
	$ emulator
	
## CTS测试

这是针对安卓系统的一个兼容性测试，关于此我会另外写一个指南，这里记录一下我在模拟器上进行CTS测试时遇到的一个小问题，就是使用adb命令向安卓的/mnt/sdcard目录下写文件时发现文件系统只读，写不进去？呵呵，开什么玩笑，速速将你拿下：

	mount -o remount rw /  
	
好吧，我只是集网络上安卓源码编译相关之精华于此，希望对有志于安卓源码研究的兄弟们有所帮助，在下告辞！
