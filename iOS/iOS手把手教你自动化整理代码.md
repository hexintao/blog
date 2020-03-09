---
title: iOS开发之--手把手教你自动化整理代码
date: 2017-09-03 10:57:23
categories: iOS开发
tags: [spacecommander, iOS]
---
"这谁写的代码，前边的空格手敲的啊，我正常换行居然没对齐？！""这个 else 是跟哪个 if 配对的卧槽".......如果你也这样抱怨过，恭喜你，来对了，请接着往下看。
<!-- more -->
谁都想自己写的代码清清爽爽，也都想自己工作中看到的代码清清爽爽，或者最起码尽量风格统一。但是代码格式这种重复劳动的问题，如果旁边坐一个美女帮我添一个空格，做一个换行该多好啊。如果你是这样想的，可以右上角xx出门找x家、x天，地上小纸片打个电话谈价钱去了。我这里要分享的是用 [spacecommander](https://github.com/square/spacecommander) 工具在 Xcode 8.x 时代实现对 `OC` 的自动化，另外 `spacecommander` 跟 `git` 更配哦，好了，飙车正式开始！
## 1. 准备工作和环境配置
#### 1.1 下载源文件
从上边的 `github` 链接下载 `spacecommander` 源文件并解压。你可以将文件放到自己项目的工程目录下，这样 `git` 可以将 `spacecommander` 源文件自动同步到远端，其他同事更新代码之后只需要执行命令即可，更方便。我这里是先放在了电脑的文稿目录下。   
#### 1.2 配置
使用命令行 `cd` 到工程目录，然后使用 `spacecommander ` 目录下的 `setup-repo.sh` 初始化环境。命令类似于：  `$ /Users/mac/Documents/spacecommander/setup-repo.sh`，   
前边是 `spacecommander ` 源文件所在的目录，自行替换即可，或者更简单的直接将 `setup-repo.sh` 文件拖拽到命令行，系统会自动生成路径信息，然后直接回车即可。之后你的命令行大概会是这个样子：![set-up](/images/spacecommander/spacecommander-setup.png)而你的工程目录下会多出这么三个鬼东西：![setupFiles](/images/spacecommander/spacecommander-setupFile.png)
   
1. 左起第一个是个快捷方式，指向我们刚刚解压的 `spacecommander ` 源文件中的 `.clang-format` 脚本。使用编辑器打开这个脚本你会发现![clangFormat](/images/spacecommander/spacecommander-clangFormat.png)其实我们要格式化的样式都是在这文件中定义的，包括基于的代码风格（ Google / LLVM ）、tab 按键缩进的长度等等。不明白某个选项的可以搜索或者直接搜 " `clang-format` 配置"。这里不赘述。或者可以自己写个小 demo，弄几个 `if-else`、`for`循环、`block` ，方法和属性等，试着修改一下这个文件，看格式化后是什么样子，印象更深刻，活着就是折腾嘛。   
2. 中间是一个 `.git` 的文件夹，点进去会发现有一个`hooks` 的文件夹，继续点进去有一个 `pre-commit` 的可执行文件。`hooks` 顾名思义，钩子的意思，详见[git 钩子官方说明](https://git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-Git-%E9%92%A9%E5%AD%90)。说白了，其实就类似于是 `OC` 中的 `block` 模式，使用 `git` 的小伙伴们对 `git commit` 提交代码肯定再熟悉不过了，不过，假如我们想在提交每次提交代码之前进行一些操作，例如：格式化代码 : ) 。那这个钩子就相当于是 git 官方给我们暴露出来的 `block` 回调，可以在每次执行 `commit` 之前自动调用，从而达到自动化的目的。   
3. 看到最后一个文件，是不是特别熟悉？和我们工程目录下的忽略文件一毛一样！它的作用就是忽略文件（呸！废话！）。其实这个是因为我初始化时的目录没跟我原有的 `.gitignore` 所在目录一致导致的。也就是说，`setup-repo.sh` 会自动检测当前目录下是否有 `.gitignore` 文件，如果有就会自动在后边追加 `.clang-format`，如果没有就会新建一个忽略文件并添加 `.clang-format`。目的当然是为了不同步 `.clang-format` 文件。因为 `.clang-format` 只是个快捷方式，当被同步到其他电脑上时，这个快捷方式很大程度上是无效的，所以就不需要被同步到远端。所以需不需要这个文件被忽略，取决于你的 `spacecommander` 源文件的目录。如果 `spacecommander` 也在你的工程目录下并且被同步到了远端，那这个快捷方式就可以不用忽略，这样其他的协作小伙伴 `pull` 之后什么操作都不用，就可以直接使用自动整理代码的功能。否则就不需要同步，到时候喊你的小伙伴再看一遍这篇文章即可，我真是个心机婊啊 : )。    
## 2. 整理代码
#### 2.1 常用的格式化命令
看到现在是不是对于自动化整理代码的流程大致有印象了呢。接下来就是见证奇迹的时刻。在 `spacecommander` 的源文件目录下，除了 `setup-repo.sh` 之外，其实还有好几个 `.sh` 文件。它们就是我们接下来要使用的命令们。 

* `format-objc-file.sh`：整理单个文件的命令，用法同 `setup-repo.sh` 一样，也是通过路径加载命令，然后空格加需要格式化文件的路径即可。
* `format-objc-files.sh`：英语老师教我们，名词加 s 代表复数，所以这个命令是用来格式化多个文件的，用法同上。需要被格式化的文件路径之间用空格隔开即可。
* `format-objc-files-in-repo.sh`：in-repo，顾名思义，就是整理整个仓库下的所有文件。这个命令建议不要一开始就使用，请阅读完 排除整理 部分之后再使用。

说完了格式化需要用到的命令，我们暂时还不能立即使用，还要知道怎么避免某些文件不被格式化，因为现在的工程基本都会用到很多的第三方，而这些文件往往不需要再次整理。好在这些东西 `spacecommander` 都已经为我们想到了，配置起来也是相当的简单，下边会介绍到。当然你可以自己写个小 demo，或者使用`spacecommander` 目录下`Testing Support` 的测试代码进行测试。这里贴几张格式化官方例子的前后对比图吧：![行末注释前](/images/spacecommander/inlineBefore.png) ![行末注释后](/images/spacecommander/inlineAfter.png) ![block前](/images/spacecommander/blockBefore.png)  ![block后](/images/spacecommander/blockAfter.png)  ![字典前](/images/spacecommander/dictionaryBefore.png)  ![字典后](/images/spacecommander/dictionaryAfter.png)
#### 2.2 自动化整理
上边我们提到 git 钩子的问题，按照上述步骤配置完成之后，其实我们已经就已经实现了自动化整理代码的目的。试一下，修改项目中的某个文件，然后 `git add .` 之后 `git commit -m "提交备注信息"`，注意一定要是这两个命令，如果使用 `git commit -am "提交备注信息"` 的话，`spacecommander` 是无法识别要整理的文件的。正常情况下会提示![gitCommit](/images/spacecommander/gitCommit.png)。这就说明要提交的文件不符合代码规范的要求。接下来我们可以有两种操作：使用`git commit --no-verify`强制提交，这不是我们的重点。或者使用`spacecommander路径/format-objc-files.sh -s`先对代码进行格式化，这里的`-s`后缀参数是`-stage`的意思，是只对暂存区，也就是要提交的代码进行格式化。成功格式化之后会提示`Formatting complete`。然后我们再次运行 `git commit -m`对代码进行提交即可。
## 3. 排除不想整理的代码
#### 3.1 排除不需要格式化的文件夹
默认情况下，spacecommander 已经将 `Pods` 和 `Carthage` 路径下的文件排除掉了![podIgnore](/images/spacecommander/podIgnore.png)所以不用担心这些文件。如果是手动拖入的第三方，我们可以在项目工程目录下，创建一个名为 `.formatting-directory-ignore` 的文件，里边写入需要忽略的文件夹路径即可。  
#### 3.2 排除不需要格式化的单个文件
上边说到的是文件夹忽略，那如果是单个文件要怎么做呢？不用怕，spacecommander 也支持。我们只需在需要忽略的文件的头部，注意是最头部，添加 `#pragma Formatter Exempt` 或者`// MARK: Formatter Exempt` 即可。
#### 3.3 部分代码避免格式化
同样，一个文件内的某几行代码同样也可以在整理范围之外。使用到的命令为：`// clang-format off` 和 `// clang-format on`。就像这样：![](/images/spacecommander/clangFormatOffOn.png)
3.4 bug。注册推送时的代码格式化之后老会多一个分号，像这样：![clang-formatBug](/images/spacecommander/clang-formatInfile.png)，我尝试使用 `// clang-format off` 依旧不好使，无奈只能手动修改，之后使用 `git commit --no-verify` 强制提交了。   
## 4. 快捷键配置
#### 4.1 快捷键
如果我们每次提交代码都需要输入那么长的路径信息和命令，估计代码格式化工具使用的人也不多。幸运的是，我们可以通过一些设置来简化这些操作。这种简化是属于命令行层级的操作，我使用的是 zsh 命令行，如果你使用的是系统命令行，修改思想完全一致，只不过是文件不一样。   
#### 4.2 路径快捷键
zsh 命令行需要修改用户根目录下的 `.zshrc` 文件，在文件末尾输入`export SPACECOMMANDER="/Users/mac/Documents/spacecommander"`，意思是使用`SPACECOMMANDER` 代表后边的路径信息。重启命令行工具，输入space之后按 `tab` 键即可自动填充`SPACECOMMANDER`。   
#### 4.3 命令快捷键
当然我们可以在上边路径快捷键的基础上，直接在路径快捷键的后边后缀命令，例如`SPACECOMMANDER format-objc-files.sh -s` 来对整理暂存区的代码。不过这依然不够快捷，是吧？所以我们需要使用更加方便的自定义命令快捷键。还是修改 `.zshrc` 文件，这次我们添加如下内容：
` alias formatstage="/Users/mac/Documents/spacecommander/format-objc-files.sh -s"`      
`alias formatfile="/Users/mac/Documents/spacecommander/format-objc-file.sh"`       
`alias formatall="/Users/mac/Documents/spacecommander/format-objc-files-in-repo.sh"`。
重启命令行工具，输入`format`之后按`tab`我们刚刚添加的三条命令就自动提示出来了。
## 5. 结束语 
至此，关于日常开发使用 spacecommander 自动化整理代码就介绍完了，配置完之后，开发的流程基本是这个样子：
`修改代码 -> git add . -> git commit -m -> formatstage -> git commit -m`
即可实现代码自动整理，动手试一下吧，感谢您的阅读！

