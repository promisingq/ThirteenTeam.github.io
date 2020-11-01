[TOC]




# 问题：写段寄存器时，只给了16位数，这个16位的数是什么？
- 这个16位的数是段选择子。通过这个选择子在GDT表中找一个段描述符写到段寄存器中。
- 当我们执行段操作指令：MOV, POP, LDS, LES, LSS, LGS, LFS指令时会查表
- 没有LCS指令。CS为代码段，CS的改变会导致EIP的改变，要改CS必须保证CS与EIP一起改。
# 段选择子
![](index_files/d51c156b-39f2-4eaf-bce8-53a43f026cf3.png)
- RPL 请求权限级
- TI 在Windows下 = 0。
- Index 指示GDT表的索引
![](index_files/0356ddea-3029-46c2-85eb-acefffbbdc06.png)
```
假设给的段选择子是0x0008，那么它就是距离GDT偏移8个字节的位置
0x0008 = 0000 0000 0000 1000
          Index TI RPL
0 0000 0000 0001 0 00
```
# 全局描述符表GDT
![](index_files/74e47b01-5a43-4bb1-99ae-dd5656149e37.png)`2019-08-27 17:29:50 星期二`
# 局部描述符表LDT
- Windows没有使用LDT
# 问题：段寄存器有96位，有16位段选择子，需要填充的还有80位，其中有64位是通过段选择子找到的段描述符，那么它是怎么从64位转变到80位的呢？
# 段描述符
![](index_files/84baf22d-cb3a-44d5-b739-cbc59f5987b7.png)
```
struct SegMent
{
    WORD Selector;
    WORD Attribute;        // 段描述符的高4字节第8~23位 共16位
    DWORD Base;            // 段描述符的高4字节24~31位、0~7、低4字节的16~31位 共32位
    DWORD Limit;        // 段描述符的高4字节16~19位，低4字节0~16位 共20位，那么最大值是0xFFFFF
}
```
## 段描述符P位
- P = 1 段存在于内存中
- P = 0 段不存在于内存中
## 段描述符G位
- G = 0 Limit的单位为字节，其最大值为0x000FFFFF;（剩余的补0）
- G = 1 Limit的单位为4KB，4KB = 4096B，即从0~4095。最大值用十六进制表示为：0xFFF。最大值达到0xFFFFFFFF
- 确定Limit技巧：如果G= 0,前边补3个0，如果G=1,后边被3个F
# 实验
## 实验1：构造不同的段选择子，并与GDT表中对应的描述符对比
![](index_files/53b1cdd1-c438-44bb-aea5-57fbc64d7dea.png)
# 实验：将当前虚拟机中的段选择子对应的段描述符找出来（不找FS）
![](index_files/f53c3026-9618-4450-84c5-7f8944852528.png)
## 解释dg指令得到的内容
- Sel:Selector
- Base:Base
- Limit:Limit
- Type:Type
![](index_files/a0548d09-076d-4374-901a-839e6203d149.png)
- Pl:Privilege Level
    0：0环权限
    3：3环权限
- Size:段大小
    Bg:32位
    Nb:
- Gran:granularity,粒度（以字节为单位还是4K为单位）
    By:字节
    Pg:4K
- Pres:Present是否存在于内存中
    P：存在于内存中
- Long:Nl非64位代码段
- Flags:除去Base、Limit的其余部分（Attribute）

| 段选择子  | Base  | Limit  | Attribute  |
| :------------: | :------------: | :------------: | :------------: |
| 0023  | 0  | 0xFFFFFFFF  | 0xCF3  |
| 001B  | 0  | 0xFFFFFFFF  | 0xCFB  |

## 拆分段选择子
0023 = 0000 0000 0010 0011 -> RPL = 3,TI = 0,Index = 4
001B = 0000 0000 0001 1011 -> RPL = 3,TI = 0,Index = 3
## 从GDT表分别找到对应的段描述符
![](index_files/64cd0947-00ed-4c3d-a399-56414df83e93.png)
## 根据描述符格式识别段寄存器的各个字段内容
- 拆分：00cff300`0000ffff
0000 0000 1100 1111 1111 0011 0000 0000
BASE1:0000 0000
G:1
D/B:1
L:0
AVL:0
LIMIT1:F
P:1
DPL:3
S:1
TYPE:3
BASE2:0000 0000

0000 0000 0000 0000 1111 1111 1111 1111‬
BASE3:0000 0000
LIMIT2:FFFF

- 组合
BASE：三个合起来也是0
LIMIT：由于G = 1,所以单位为4KB，后补3个F即，0xFFFFFFFF
ATTRITUTE:1100 1111 0011 即：0xCF3

- 拆分：00cffb00`0000ffff
0000 0000 1100 1111 1111 1011 0000 0000
BASE1:0000 0000
G:1
D/B:1
L:0
AVL:0
LIMIT1:F
P:1
DPL:3
S:1
TYPE:B
BASE2:0000 0000
0000 0000 0000 0000 1111 1111 1111 1111‬
BASE3:0000 0000
LIMIT2:FFFF

- 组合
BASE：三个合起来也是0
LIMIT：由于G = 1,所以单位为4KB，后补3个F即，0xFFFFFFFF
ATTRITUTE:1100 1111 1011,即：0xCFB
