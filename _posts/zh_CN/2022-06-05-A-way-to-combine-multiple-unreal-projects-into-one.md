---
title: 一种把多个虚幻工程解决方案合而为一的方法
date: 2022-06-05 11:15:13 +0800
categories: [Unreal Engine]
tags: [tooling]     # TAG names should always be lowercase
lang: zh_CN
---

## 使用引擎目录的`GenerateProjectFiles.bat`和`.uprojectdirs`文件

- 首先参考`Default.uprojectdirs`文件中的注释
- 新建`my.uprojectdirs`或直接修改`Default.uprojectdirs`文件,并写入例如`Project/`
- 此时运行`GenerateProjectFiles.bat`生成解决方案时,会额外从`引擎目录/Project`目录下的**1级子目录**寻找虚幻工程目录, 例如`UnrealEngine/Project/LyraGame`
- 完成

> 因为`.uprojectdirs`文件中只支持指定子目录,若想要外部目录的工程也被包含进解决方案文件,可以使用系统提供的`symbol-link`功能
{: .prompt-tip }

<details markdown=1>

<summary markdown="span">已废弃方法</summary>
  
## 初衷

因为工作,学习的原因,我本地会有多个虚幻工程,并且它们**共用了同一个引擎**

它们的工程文件默认都是由**UBT(Unreal Build Tool)**生成的,独立的解决方案文件

这给我带来的一些问题:

- 由于打开解决方案的成本较高,同时打开多个解决方案时的等待时间,内存占用可能都是成倍增加的

- 在IDE抽风,或者引擎代码更新而触发符号**重新解析**的时候,等待时间少则15分钟起步,那叫一个折磨

我认为不同解决方案中,物理路径相同的工程,应该视为**同一个工程**,虚幻引擎的工程不应该在每个解决方案都重新解析一遍

但目前用过的`Rider`和`Visual Studio`(宇宙第一IDE)都是会在每个解决方案中重新解析一遍.非常折磨,其中原由,令人费解,希望能有大佬告知一下

## 需求

所以我就在想,既然这些个IDE这么"愚笨",靠不住,那我能否找到一个方法,手动把多个解决方案合成一个呢?

## 步骤

下面分享的是,我使用`Rider`时的操作步骤:

1. 在想要多开的解决方案中,随意选择一个作为"**主解决方案**",并打开

2. 添加其它工程文件

- 在`Explorer`解决方案浏览器中,右键任意工程文件夹,比如`Games`,选择`Add`,`Add Existing Project...`,然后选择相关的工程文件

- 假如要添加的工程是`LyraStarterGame`,那么它的工程文件应该位于`LyraStarterGame_Folder\\Intermediate\\ProjectFiles\\LyraStarterGame.vcxproj`

3. **在"解决方案配置"中,修改刚刚添加的工程的配置**

- 点击右上角的工具栏中的`DebugGameEditor | Win64`按钮(这是我的``解决方案编译配置``,根据你的实际情况,文字会有所不同),然后选择`Edit Solution Comfigurations...`

- 找到刚刚添加的工程,可以看到它们当前都是默认的配置,可能是`DebugClient | Arm64`什么的,把它改成你需要的配置,一般是与`解决方案编译配置`一致

4. **修改工程文件参数**

- 回到`Explorer`解决方案浏览器

- 右键刚刚添加的工程(这里我拿`LyraStarterGame`举例),选择`Edit`,`Edit LyraStarterGame.vcxproj`

- 将`LyraStarterGame.vcxproj`文件中所有`$(SolutionDir)`替换为`$(ProjectDir)..\\..\\` (`$(SolutionDir)`是**当前解决方案的根目录**,`$(ProjectDir)`是**当前工程的工程文件所在目录**,即`LyraStarterGame_Folder\\Intermediate\\ProjectFiles\\`,所以这里使用两次`..\\`,获得了**当前工程的根目录**)

- 如果不想一次性全都替换,也可以搜索你的"`编译配置`",比如 `DebugGame_Editor|x64`,找到对应编译配置的相关配置,只替换三个`NMake`相关的命令行中的文本,**但可能会存在工程文件无法正常解析符号的问题**:比如`Switch Header/Source`功能无法使用,语法着色失效等,如遇到问题,进行全量替换即可

## 不足

每次通过`UBT`重新生成解决方案文件时,相关的工程文件会被覆盖,上面的操作需要重新执行一次.如果能有IDE的支持,或者自动化的工具就好了

</details>


