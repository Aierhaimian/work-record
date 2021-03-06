**C++结构体字节对齐** 

## 前言 ## 
> 在计算机中数据存储和传输以位(bit)为单位，每8个位bit组成1个字节(Byte)。32位计算机的字长为32位，即4个字节；对应的，64位计算机的字长为64位，即8个字节。计算机系统对基本类型数据在内存中存放的位置有限制，要求这些数据的起始地址的值是某个数k的倍数，这就是所谓的内存对齐，而这个k则被称为该数据类型的对齐模数(alignment modulus)。

## 结构的存储分配 ## 

编译器按照结构体成员列表的顺序为每个成员分配内存，当存储成员时需要满足正确地边界对齐要求时，成员之间可能出现用于填充地额外内存空间。32位系统每次分配字节数最多为4个字节,64位系统分配字节数最多为8个字节。
以下图表是在不同系统中基本类型数据内存大小和默认对齐模数:
![](http://i.imgur.com/Pq7ROq4.png) 
    注：此外指针所占内存的长度由系统决定，在32位系统下为32位（即4个字节），64位系统下则为64位（即8个字节）. 


## 没有#pragma pack宏的对齐 ## 

**对齐规则** :

 1.  结构体的起始存储位置必须是能够被该结构体中最大的数据类型所整除。
1.  每个数据成员存储的起始位置是自身大小的整数倍(比如int在32位机为4字节，则int型成员要从4的整数倍地址开始存储)。
1.  结构体总大小（也就是sizeof的结果），必须是该结构体成员中最大的对齐模数的整数倍。若不满足，会根据需要自动填充空缺的字节。
1.  结构体包含另一个结构体成员，则被包含的结构体成员要从其原始结构体内部最大对齐模数的整数倍地址开始存储。(比如struct a里存有struct b，b里有char,int,double等元素,那b应该从8的整数倍开始存储。)
1.  结构体包含数组成员，比如char a[3],它的对齐方式和分别写3个char是一样的，也就是说它还是按一个字节对齐。如果写：typedef char Array[3],Array这种类型的对齐方式还是按一个字节对齐，而不是按它的长度3对齐。
1.  结构体包含共用体成员，则该共用体成员要从其原始共用体内部最大对齐模数的整数倍地址开始存储。

现在给出一个结构体，我们针对win-32和Linux-32进行分析,例1：

    struct MyStruct 
    { 
      char a; 
      int b; 
      long double c; 
    }; 

**解答** ：<br/>
**win-32位系统下** ：<br/>
由上图可知该结构体的最大对齐模数为sizeof(long double)=8；假设MyStruct从地址空间0x0000开始存放。char为1个字节，所以a存放于0x0000中；int为4个字节，根据规则，b存储的起始地址必须为其对齐模数4的整数倍，所以a后面自动填充空缺字节空间0x0001-0x0003，因此b存放于0x0004-0x0007中。long double是8个字节，由于32位系统每次最多分配4个字节，则首先分配0x0008-0x000B，由于不够存储空间，则继续分配0x000C-0x000F,所以c存储在0x0008-0x000F中，由于此时总存储空间为4+4+8=16；则16满足最大对齐模数sizeof(long double)=8的整数倍；因此，sizeof(MyStruct)=16个字节。<br/>
**Linux-32位系统下：** <br/>

由上图可知该结构体的最大对齐模数为4；假设MyStruct从地址空间0x0000开始存放。char为1个字节，所以a存放于0x0000中；int为4个字节，根据规则，b存储的起始地址必须为其对齐模数4的整数倍，所以a后面自动填充空缺字节空间0x0001-0x0003，因此b存放于0x0004-0x0007中。long double是12个字节，由于32位系统每次最多分配4个字节，则首先分配0x0008-0x000B，由于不够存储空间，则继续分配0x000C-0x000F,仍然不满足存储c，则继续分配0x0010-0x0013，所以c存储在0x0008-0x0013中，由于此时总存储空间为4+4+12=20；则20满足最大对齐模数4的整数倍；因此，sizeof(MyStruct)=20个字节。<br/>

>  注：以下的所有例子都是在win-32下实现

例2：

    struct B{   
      char a;   
      int b;   
      char c;   
    }; 

由上图可知该结构体的最大对齐模数为sizeof(int)=4；假设B从地址空间0x0000开始存放。char为1个字节，所以a存放于0x0000中；int为4个字节，根据规则，b存储的起始地址必须为其对齐模数4的整数倍，所以a后面自动填充空缺字节空间0x0001-0x0003，因此b存放于0x0004-0x0007中。c也是char类型，所以c存放在0x0008中；此时结构体B总的大小为4+4+1=9个字节；则9不能满足最大对齐模数4的整数倍；因此在c的后面自动填充空间0x0009-0x000B，使其满足最大对齐模数的倍数，最终结构体B的存储空间为0x0000-0x000B；则sizeof(B)=12个字节。<br/>
**例3：空结构体** 

    struct C{   
    }; 

sizeof(C) = 0或sizeof(C);C为空结构体，在C语言中占0字节，在C++中占1字节。

**例4：结构体有静态成员** 

    struct D{   
       char a;   
       int b;   
       static double c; //静态成员   
    }; 

静态成员变量存放在全局数据区内，在编译的时候已经分配好内存空间，所以对结构体的总内存大小不做任何贡献；因此，sizeof(D)=4+4=8个字节

**例5：结构体中包含结构体** 

    struct E{   
      int a;   
      double b;   
      float c;   
    };   
    struct F{   
      char e[2];   
      int f;   
      short h;   
      struct E i;   
    }; 
     
在结构体E中最大对齐模数是sizeof(double)=8；且sizeof(E)=8+8+8=24个字节；在结构体F中，除了结构体成员E之外，其他的最大对齐模数是sizeof(int)=4；又因为结构体E中最大对齐模数是sizeof(double)=8；所以结构体F的最大对齐模数取E的最大对齐模数8；因此，sizeof(F)=4+4+8+24=40个字节。

**例6：结构体包含共用体** 

    union union1   
    {   
      long a;   
      double b;   
      char name[9];   
      int c[2];   
    };   
    struct E{   
      int a;   
      double b;   
      float c;   
      union1 MyUnion;   
    }; 

共用体中的最大对齐模式是sizeof(double)=8；则sizeof(union1)=16；结构体E的最大对齐模数也是8；则sizeof(E)=8+8+8+16=40个字节。

**例7：结构体包含指针成员** 

    typedef  struct A{   
        char a;   
        int b;   
        float c;   
        double d;   
        int *p;   
        char *pc;   
        short e;   
    }A; 

结构体包含的指针成员的大小根据系统类型决定，由于这里是在win-32位系统下分析，则指针大小为4个字节；因此，结构体A的最大对齐模数为sizeof(double)=8；则sizeof(A)=4+4+8+8+4+4+8=40个字节。
**存在#pragma pack宏的对齐** 

    #pragma pack (n)//编译器将按照n个字节对齐   
    #pragma pack () //取消自定义字节对齐方式 

**对齐规则** 
结构，联合，或者类的数据成员，第一个放在偏移为0的地方，以后每个数据成员的对齐，按照#pragma pack指定的数值和自身对齐模数中较小的那个。

**例8：按指定的对齐模数** 

     #pragma pack (2) /*指定按2字节对齐*/   
    struct G{   
        char b;   
        int a;   
        double d;   
        short c;   
    };   
     #pragma pack () /*取消指定对齐，恢复缺省对齐*/ 

在结构体G中成员变量的最大对齐模数是sizeof(double)=8；又因为指定对齐模数是2；所以取其较小者2为结构体G的最大对齐模数；则sizeof(G)=2+4+8+2=16；由于16是2的整数倍，则不需要填充。
## 总结 ## 

在分析结构体字节对齐时，首先确定有没有利用#pragma pack()宏定义指定对齐模数；根据情况对应上面进行两种情况分析，针对不同的系统会得到不同的结果。

## 补充： ## 
在Visual C++下可以用__declspec(align(#))声明数据按#字节对齐<br/>
GUN C下可以使用以下命令:<br/>
__attribute__((aligned (n)))，让所作用的结构成员对齐在n字节自然边界上。如果结构中有成员的长度大于n，则按照最大成员的长度来对齐<br/>
 __attribute__((__packed__))，取消结构在编译过程中的优化对齐，按照实际占用字节数进行对齐。<br/>
C++11新加关键字alignas(n)<br/>

**原文链接** ：http://blog.csdn.net/chenhanzhun/article/details/39641489
