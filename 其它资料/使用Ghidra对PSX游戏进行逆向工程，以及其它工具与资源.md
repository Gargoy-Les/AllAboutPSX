# 使用Ghidra对PSX游戏进行逆向工程

这是我在研究PlayStation(PSX)逆向工程时总结出的小手册。如果你对逆向工程的基本概念不熟悉，请先看什么是[反编译](https://tetracorp.github.io/tokimeki-memorial/methods/what-is-decompilation.html)?

## 读取PlayStation光盘

PlayStation光盘是以标准的ISO 9660 CD-ROM格式记录的，可以在PC上正常读取。光盘映像文件（如.ISO、.BIN/CUE）可以用AcetoneISO或PowerISO等软件进行(虚拟光驱)挂载或内容提取。

但有时会出错：

1.有些文件实际上是隐藏的。大多数PSX游戏所使用的PsyQ API能够基于数值扇区位置(LBN)而不是以文件名读取文件，导致PC无法读取。

2.故意制作光盘坏块，以防止复制和盗版。

3.一些读取光盘镜像的软件在处理多轨光盘时出错。您需要做一些怪异的事情，比如把第一轨的.BIN加载到AcetoneISO中，或者用binmerge合并多个.BIN，或者制作一个只包括第一轨的假.cue文件。

不过，至少有两个关键文件一般是容易理解的：system.cnf，明文显示主游戏可执行文件的文件名；可执行文件本身通常被命名为PSX.EXE或类似SLPM_86.053，这就是逆向分析所需要的文件。

## 使用Ghidra开始分析可执行文件

Ghidra是美国国家安全局(NSA)开发的一款逆向分析软件，在2019年免费公布。也有其他的逆向工具，如radare2和IDA Pro。而Ghidra强大且免费。

Ghidra原生支持PSX的CPU指令集(MIPS R3000, Ghidra将其理解为MIPS默认的32位小端模式)。插件ghidra_psx_ldr可辅助进行PSX专用的分析。

下载并解压Ghidra。下载ghidra_psx_ldr的最新压缩包，不解压，而将其放入Ghidra的文件夹中。通过在Windows上运行ghidraRun.bat或在Linux上运行./ghidraRun启动Ghidra。从菜单中选择File - Install Extensions并选择ghidra_psx_ldr。

创建一个新的 non-shared project (File - New Project)。单击列表中的项目入口(project entry)并单击绿龙图标以打开代码浏览器(Code Browser)。在这个新窗口中，点击File - Import(或按下I键)，并选择你的PlayStation可执行文件。

软件现在应该已经提醒您Auto-Analyze(自动分析)。如果暂时不分析，您可以稍后在Analysis菜单中运行它(或按A键)。Auto-Analyze需要不少时间。之后，按CTRL-S保存。以备将来的不时之需，请注意保存后会清除撤销记录。

默认打开了很多不需要的窗口。你比较需要的窗口是Symbol Tree(列出变量和函数名)，Listing(MIPS反汇编结果)和Decompile(反编译为C语言)。其他窗口现在不过是占据空间罢了。

##  理解汇编指令

您现在有成千上万行MIPS汇编代码，可能只有很少变量和函数拥有有意义的名字。如果您不熟悉MIPS汇编，这将使人望而生畏。

首先，我们大体地(宏观地)理解汇编代码。以下是一个片段：

```
                     **************************************************************
                     *                          FUNCTION                          *
                     **************************************************************
                     undefined myrand()
                       assume gp = 0x800f0d90
     undefined         v0:1           <RETURN>
                     myrand
800425d0 0f 80 03 3c     lui        v1,0x800f
     assume gp = <UNKNOWN>
800425d4 f0 01 63 8c     lw         v1,offset rng_800f01f0(v1)                       = ??
800425d8 0f 80 01 3c     lui        at,0x800f
800425dc 77 03 62 24     addiu      v0,v1,0x377
800425e0 08 00 e0 03     jr         ra
800425e4 f0 01 22 ac     _sw        v0,offset rng_800f01f0(at)                       = ??
```

这里的 "FUNCTION "是一个注释，代表Ghidra认为这是一个函数的起始位置，也许是因为自动分析发现有跳转指令指向这个地址。

Undefined myrand()是给出的函数名。因为可执行文件没有保留其编译前的函数名、变量名等，所以名字总是FUN_800425d0这样机械化。当你研究出一个函数或变量的功能时，你可以点击它并点击键盘上的“L”来输入新名字，改变将在整个程序中生效。

如果一个函数开始被预先标识，请感谢ghidra_psx_ldr将其识别为对PSX devkit中的一个PsyQ标准库函数的引用。这可能是理解函数功能的突破口。例如，跟踪标准rand函数可以使用理解游戏随机性的机制;print函数可用于输出调戏信息;write函数在PSX游戏中总是用于写入记忆卡(Memory Card)，这是找寻游戏关键变量的突破口，它们用于存储游戏至始至终用到的信息。

每一行汇编代码的最左边都有一个8位16进制数(例如800425d0)。这是这个指令(机器码)在内存中的地址。PSX软件的内存地址通常从0x80000000开始存，而不是0。但两者都指向相同的物理内存，例如0x800425d0和0x000425d0指向相同的内存(请阅读Nocash PSX Specifications网页的Memory Map条目以获取详细信息)。

之后是四组两位十六进制数。这是最左边的地址指向的32位空间中的四字节内容。内容或许是指令(机器码)，或许是四字节变量，或是其它。

再之后是机器码代表的汇编指令。多数情况下，阅读Decompile窗口的C语言可能会更有用。

最后是分号键(;)以插入注释。反汇编窗口和反编译窗口都可插入注释。

## 区分代码和数据

Ghidra擅长识别代码，但并不完美。由于可执行文件中所有内容都是原始字节码，所以它有时可能不能正确的识别一段代码，或混淆数据类型。

如果您猜测一大段无法识别的数据可能是代码，请选择第一个字节并按D键以进行反汇编。有时，这实际上是一个函数，但没有从任何地方直接调用，则请点击F将其转换为函数。如果发生错误(例如，产生的“代码”是不正确的)，只需按Ctrl-Z撤销。您还可以按C键将已识别的内容倒转回未识别的字节码。

您还可以右键单击一行代码，点击Data，并选择要标识的数据类型。这对于识别字符串很方便。特别是日本游戏经常使用Shift-JIS编码，而Ghidra不会自动识别。右键单击任意字符串，到数据-设置和设置字符集Shift-JIS。更好的是，数据-默认设置将为所有字符串设置此值。更好的办法是, 在Data - Default Settings中设置，将会对所有字符串生效。

Ghidra甚至允许您输入字符串的翻译。您可以参考机器翻译网站 [WWWJDIC](https://www.edrdg.org/cgi-bin/wwwjdic/wwwjdic)。

## 跟踪代码流

如果遇到函数或变量，可以双击它，或将键盘光标移到那里并按Enter键来访问它。您可以通过按工具栏上的“左”键或键盘上的“Alt+左”键返回到之前的位置。

函数的右上角也许有一个XREF列表，这是调用它的其他函数的列表。您可以选择一个变量或函数并按Ctrl-Shift-F来查看调用他的代码。一种分析方法是从已经确定功能的东西(字符串或函数)做为突破口，跟踪哪些函数使用它，以研究这些函数是如何工作的。

您做的每一个标记都仿佛是一块拼图，以帮助你理解其它代码。

## 用调试器实时监视内存

ghidra_psx_ldr有一个调试器，但在我写这篇文章时，它的功能还很有限。仿真器Mednafen有一个更强大的调试器，它可以让您实时观察和修改内存。

设置Mednafen并使用像Mednaffe这样的GUI运行它。启动一个游戏(它可以加载BIN/CUE文件，即便在PC上不能正确读取)。如果你在游戏控制器上遇到问题，请遵循以下说明(建议装个手柄，这样你就可以在调试器屏幕打开的情况下用键盘、鼠标输入信息):

1. 启动游戏
2. 按下所有控制键和方向键，把摇杆转到最大，按下所有L/R键
3. 按F3键重新选择控制器(翻译可能错误，不影响理解)。
4. 使用Alt-Shift-1启动控制器配置，按指示按下按钮。

控制器只需要配置一次。

现在，按Atl-D加载调试器。 [Mednafen debugger documentation](https://mednafen.github.io/documentation/debugger.html)提供使用调试器的全面信息，但最重要的是，Alt-1显示CPU信息，S键终止或单步步过，R键继续，Alt-3显示内存信息。这些视图都实时更新。

在内存信息窗口，按G键并输入地址以跟踪。Mednafen显示的地址与Ghidra中的地址一致，只是它们以0而不是8开头。您可以实时观察已知变量的变化。您还可以按D键将内存转储到文件中，以便以后分析。从0转储到200000(十六进制)，大小为2MB。

## 其他可执行文件（动态加载技术）

PSX只有2MB的RAM和1MB的VRAM，程序最大只能是2MB。与CPU可以直接寻址ROM的卡带游戏机不同，基于CD的游戏机必须先将程序加载到RAM中，因此需要加载时间。对于许多PlayStation游戏来说，这已经足够了，但对于复杂的游戏来说，可能还不够。

解决方案是PsyQ动态加载技术，其中多个程序块保存在单独的文件中，主可执行文件可以根据需要加载和卸载这些文件。例如，Tokimeki Memorial有大约4.9MB的可动态加载代码。

每个程序块都将被加载到内存中的固定位置。得到这个位置很麻烦，可以参考我的办法：

1. 以十六进制打开程序块，找到大约前十六个字节。
2. 在游戏加载这段程序块时，转储RAM。
3. 在RAM中搜索这些字节，以找到它们在内存中的加载位置。

得到程序块在内存中的加载位置，就可以按如下方式将其加载到Ghidra中：

1. 在code browser中，选择 File - Add to Program并选择相关二进制文件。
2. 点击“Options”。勾选“Overlay”，输入块名称，并输入相关的基址（请记住，对于PSX，它以8而不是0开始）。
3. 选择Tools - Memory Map（或点击工具栏上内存形状的图标）。选择加载的程序块并确保选中以下项：R, X, Overlay, and Initialized。“X”很重要，它代表可执行文件。
4. 双击起始地址。这应该会在代码浏览器中显示。
5. Analysis - Auto Analysis 以反编译。

处理动态加载的另一个难点是，程序块有时在CD-ROM文件列表中未列出，因为只有主EXE才能被文件系统访问。您必须检查主可执行代码，以查看加载到何处以及多少字节。或者，您可以从内存转储中提取它。

# PSX反向工程所需要的工具和资源

以下是用于分析、修改或翻译PlayStation游戏的工具和文档列表。很多软件在Linux中可以通过apt之类的包管理器方便地获得。

## PSX文件格式转换工具

[jPSXdec](https://github.com/m35/jpsxdec)

音视频转换器，可从光盘镜像中提取PSX文件格式并建立文件位置索引，并转换XA、STR、TIM文件。

[psxsdk](https://github.com/simias/psxsdk)

用于创建PlayStation游戏的非官方SDK。包括PSX格式（ISO/BIN、WAV/VAG.TIM/BMP）的一些转换工具的源代码。

## CD-ROM tools

- [PSXImager](https://github.com/cebix/psximager)

  三个工具：psxrip（从BIN/CUE提取文件和光盘元数据）、psxbuild（创建光盘镜像）和psxinject（修改光盘镜像）。但作者只提供了源代码，并且处理某些镜像有困难。

- AcetoneISO

  用于挂载、提取和刻录单轨ISO和BIN的Linux工具。

- [PowerISO](https://www.poweriso.com/)

  商用Windows/Linux光盘制作工具。

- ccd2iso

  转换IMG到ISO的Linux工具。

- iat

  将各种格式转换为ISO的Linux工具。

- bchunk

  Linux工具，可以转换BIN/CUE到ISO、音乐CD到WAV。支持PSX 2336字节块跟踪。

- isoinfo

  列出ISO文件内容的Linux工具。

- [binmerge](https://github.com/putnam/binmerge)

  Python脚本，将多轨道BIN/CUE合并为一个。

## 反汇编器和反编译器

- [Ghidra](https://ghidra-sre.org/)

  美国国家安全局(NSA)的反汇编/反编译程序，于2019年向公众发布。[ghidra_psx_ldr](https://github.com/lab313ru/ghidra_psx_ldr)

  Ghidra的PSX分析插件。

- [Radare2](https://www.radare.org/)

  免费、开源的反汇编工具。

- [IDA Pro](https://www.hex-rays.com/products/ida/)

  商用反编译器。[免费版本](https://www.hex-rays.com/products/ida/support/download_freeware/)但是只支持x64/64且不支持MIPS(PSX)。

- [This Dust Remembers What It Once Was](https://www.beneaththewaves.net/Software/This_Dust_Remembers_What_It_Once_Was.html)

  辅助Ghidra分析PSX的工具。可以将PSX-EXE转换为ELF，并对Psy Q的.sym调试符号文件进行句法分析。

## 仿真器和调试器

[Mednafen](https://mednafen.github.io/documentation/psx.html)

自带强大调试器的仿真器。

[No$psx](https://problemkaputt.de/psx.htm)

PSX仿真器和调试器。

[ePSXe](https://www.epsxe.com/)

PSX仿真器。

## 文档

[Nocash PSX Specifications](https://problemkaputt.de/psx-spx.htm)(免费的PSX技术规范)

No$psx的作者写的PSX系统技术文档。

[Everything You Have Always Wanted to Know about the Playstation But Were Afraid to Ask v1.1](https://www.raphnet.net/electronique/psx_adaptor/Playstation.txt)

2000开始编写的，关于PSX硬件信息的文档，([原版](https://web.archive.org/web/20011211221846/http://www.execpc.com/~halkun/PSX/index.html)).

[Introduction to Hacking the Sony Playstation One](https://www.retroreversing.com/ps1/)

RetroReversing网站的一系列关于PSX的文档。

[Net Yaroze Startup Guide](https://psx.arthus.net/sdk/NetYaroze/Net Yaroze Official - Startup Guide.pdf)

官方的Net Yaroze PSX开发包参考文档。

## 软件开发和杂项

[Romhacking.net PSX Utilities](https://www.romhacking.net/?page=utilities&category=&platform=17&game=&author=&os=&level=&perpage=200&title=&desc=&utilsearch=Go)

用于各种平台的PSX程序列表。

[psx.arthus.net](https://psx.arthus.net/)

一些PSX开发资源。

[PSX Links](https://ps1.consoledev.net/)

一些文档和资源。

[loveemu labo’s PSX articles](https://loveemu.hatenablog.com/archive/category/PSX)

一些文章。