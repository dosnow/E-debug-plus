# E-debug Plus

起初E-debug是由xjun开发的,但是由于我并不满足E-debug的函数识别率,于是基于xjun的源码进行了修改、重构,在此对Xjun哥表示感谢

### 关于E-debug Plus的工作原理

E-debug Plus的目的是识别函数命令,为此首先开发了ECodeMake,ECodeMake是将易语言的支持库函数转换为Esig的一款工具:https://github.com/fjqisba/CodeMake

例如取字节集长度这个命令
![001](/IMG/001.png)

ECodeMake会将之转换为8B44240C8B0085C075078B4C24048901C38B5424048B40048902C3

其中为了识别函数的尾部,运用到了Bea反汇编引擎.

而同时E-debug则是一款OD的插件,能够提取蕴含在易语言静态编译程序里的命令地址信息,将之逐个与上面十六进制进行对比,就能得出函数的命令了.

很明显有了这两个东西,绝大多数函数命令都能识别出来了.

然而易语言支持库命令中不乏有这样的函数:
![002](/IMG/002.png)

这样的函数
![003](/IMG/003.png)

甚至是这样的函数
![004](/IMG/004.png)

起初我的做法比较愚蠢,就是通过在CODEMAKE中对比函数来得到这些无法识别的函数,再对它们进行额外的记录,像下面这样.
```C
typedef struct
{
	char	m_CommandName[64];              //命令名称
	int  	m_CallType;                     //函数类型,0代表无重复CALL,1代表需要判断第二个CALL,2代表需要判断IAT函数的CALL
                                            //3代表判断CALL之前有一段特殊代码
	ULONG   m_CallOffset;                   //记录需要判断的call的偏移
	int		m_size;                         //程序一阶函数的字节大小
	UCHAR   m_opcode[128];                  //匹配的字节
	int     m_size2;                        //程序二阶函数的字节大小
	UCHAR	m_opcode2[128];                 //类型1或3时的opcode
	char    m_IATEAT[128];                  //call为类型2时为IAT与EAT
}ESTATICLIBOPCODE, *PESTATICLIBOPCODE;
```

这就是E-debug3.0当中的做法,现在看来是非常不好的,即使通过一段时间的完善,能够识别出易语言系统核心支持库的600多条命令,

后续批量制作特征文本、对任意函数生成特征文本又该怎么办呢?

于是我思考了很久,想到了得自己制作一个简易的函数<->文本转换引擎,即ECODEMAKE负责将函数转换为文本,E-debug则负责将文本转换为函数进行识别.

### Esig介绍:
Esig就是ECODEMAKE生成的函数文本化格式,为了应付各种各样的函数,有以下文本格式,  
<>:CALL 命令
-->:jmp 跳转指令  
<[]>:IAT 函数调用指令,即FF15  
[]:IAT 函数jmp指令,即FF25  
[]>:jmp 函数类跳转,也是FF25  
??:通配符

通过以上一些格式,我们可以想象出易语言的大部分函数终于能表示出来了.

在生成出来的Esig文件中,采用了函数命名的方式来记录,上半部分是子函数列表,下半部分是主函数列表.
在可视化的Esig文件中,每个人都可以对识别出错的函数进行修改,这正是我的目的.
同时由于易语言fne支持库的函数和实际编译出来的程序代码有出入,导致Esig中有一些目前识别错误的易语言命令,如果有的话可以反馈给我完善.
对于有一些ECODEMAKE很难制作出来的Esig,可以考虑人工生成.

例如飘零金盾8.0的Esig就是我配合CODEMAKE+人工制作的一份Esig,
需要注意的是,不应该出现类似于
A8A6A这样的特征码,因为存在半个字节,不过FF5?56EE这种却是可以的.


### E-debug Plus的匹配
E-debug Plus对于函数的匹配要求相当严格,一个字节出错也会匹配失败，所以此款软件目前不适合用于分析那些经过加壳处理的软件,  
因为这些软件会修改调用函数的IAT CALL,导致函数字节发生变化.
这个问题从理论上能用模糊匹配来解决,即通过计算匹配字节的百分比来判断函数,然而目前这并不是我想要研究的方向.
脱壳或许是更好的处理方式,然而部分壳会将整个支持库函数进行VM,如此则软件完全失效.

目前只做出了定点分析,未实现扫描算法,所以此款软件无法应付黑月编译的易语言程序.
而至于编译或者独立编译,由于软件数目少,研究难度低,暂时也懒得去写了.
现在我想进一步收集足够多的支持库函数,提高匹配率.等收集到足够多的信息后，再做进一步打算.

### 使用说明:

将插件与Esig文件夹放置于OD目录下

OD\plugin\E-debug.dll

OD\plugin\Esig即可

未加壳的程序,在text区段可直接分析.
加壳的程序,可待text区段解码后，CPU窗口到达text区段再开始分析.


同时Esig也是可以修改和制作的,

另外如果有分析崩溃的样本,也可以发给我看看.

