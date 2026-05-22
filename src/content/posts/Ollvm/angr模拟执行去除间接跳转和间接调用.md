---
title: angr模拟执行去除间接跳转和间接调用
published: 2026-05-22
description: angr模拟执行去除间接跳转和间接调用
image: ./cover.jpg
tags: [逆向]
category: Ollvm混淆
draft: false
---


---
# 前言

以2026polarisCTF easyre([easyre | CTF+](https://www.ctfplus.cn/problem-detail/2045446370139049984/description))为例，展示angr模拟执行去除间接跳转和间接调用混淆的过程。其中的间接跳转并非标准的间接跳转，但思路大抵相同。然后我的解决方案是将idapython切换为装了angr的python虚拟环境，并在ida中使用脚本进行去除。

---


---
# 安装angr并切换idapython

## 安装angr

使用conda安装:
```
#创建名为angr的python3.11.9虚拟环境
conda create -n angr python=3.11.9 -y
#激活环境
conda activate angr
#pip 安装angr
pip install angr
```

## idapython切换

使用conda env list查看angr安装的目录找到python3.dll 复制该路径
然后找到ida安装的根目录运行命令
```
 ./idapyswitch.exe --force-path python3.dll的路径
```
然后点击idapyswitch可以看idapython路径和版本


### 问题

打开ida会报错：
(报错信息忘记是啥了，反正是跟libz3.dll有关的)

这个问题的根源在于 IDA Pro 自带的 `libz3.dll` 与你在 Conda 环境中安装的 `z3` Python 包版本不匹配，造成了 DLL 文件加载冲突。

具体来说，当你在 IDA 中运行 Python 脚本时，IDA 会将其安装目录下的 `libz3.dll` 优先加载。而 angr 依赖的 `z3` 库如果版本更新，就会因为调用了这个旧版 DLL 中不存在的函数（比如 `Z3_solver_register_on_clause`）而报错。

### 解决办法：
   
   **重命名 IDA 目录下的 DLL**：找到 IDA 安装根目录下的 `libz3.dll`，将其重命名为类似 `libz3.dll.bak` 的文件。这会强制让 IDA 中的 Python 使用你系统里通过 pip 或 conda 安装的版本。

如果不用有angr的idapython了可以把libz3.dll.bak命名改回去。


---



# 寄存器间接跳转


## 代码分析
首先看server.exe的main函数

main函数控制流程图:
![](assets/Pasted%20image%2020260521231637.png)

注意到最上面的块汇编代码是有规律的

![](assets/Pasted%20image%2020260521232207.png)
只有我用红色矩形标记的部分代码是不一样的其他的全部相同，可以猜测这部分决定跳转关系，并不包含业务逻辑代码。仔细分析汇编代码能够发现在已知栈变量的情况下所有寄存器的值都能够被推测出来。我们对栈变量rsp+arg_190和rsp+arg_194进行分析:

arg_190= dword ptr  198h(根据函数起始部分显示)
var_370= dword ptr -370h(根据函数起始部分显示)
rsp+508h+var_370 = rsp + 508h - 370h = rsp + 198h = arg_190
所以**\[rsp+508h+var_370] = \[rsp + arg_190]**
它们指向的时同一个地址
同理得到**\[rsp+508h+var_36C] = \[rsp+arg_194]**

搜索一下两个栈变量rsp+508h+var_370 , rsp + arg_190

![](assets/Pasted%20image%2020260521233125.png)

![](assets/Pasted%20image%2020260521233832.png)

发现对该栈地址赋值的指令只有一处
![](assets/Pasted%20image%2020260521234054.png)

loc_14000108A时主分发器。该栈地址由\[rsp+508h-360h]决定，分析主分发器块指令可以发现最后jmp r8 寄存器的值其实由\[rsp+508h-360h]和\[rsp+508h+var_36C] = \[rsp+arg_194]决定

搜索rsp+508h-360h
![](assets/Pasted%20image%2020260521234437.png)
而这些对rsp+508h-360h赋值的地方都在中间那些块的末尾处。所以可以推测中间那部分的块大概率是真实块而且\[rsp+508h-360h]和\[rsp+508h+var_36C] = \[rsp+arg_194]共同决定下一个跳转地址。

搜索rsp+508h+var_36C和rsp+arg_194
![](assets/Pasted%20image%2020260521235650.png)
![](assets/Pasted%20image%2020260521235708.png)
发现只有序言块对该栈地址进行赋值
![](assets/Pasted%20image%2020260521235744.png)
最后发现\[rsp+508h+var_36C] = \[rsp+arg_194] = 2E0F1794h
所以主分发器跳转地址实则只由\[rsp+508h-360h]决定，而对\[rsp+508h-360h]赋值的操作只存在于中间部分的块的尾部指令。

总结栈变量关系:
arg_190 = var_370 = var_360; 
arg_194 = var_36C = 2E0F1794h
arg_190, var_370, var_360指向同一个栈地址，而程序通过这个地址里的值决定跳转关系.
最终得到如下关系
![](assets/Pasted%20image%2020260522104053.png)

主分发器和子分发器通过jmp 寄存器间接跳转，真实块统一跳转到循环头。知道了混淆原理后就可以开始写去混淆代码了

## 去混淆代码

流程如下:
获取所有真实块地址 -> nop掉无用块 -> angr模拟执行获取真实块间关系并进行patch.
由于我把三个流程的代码封装成了一个脚本，所以先讲思路，后面再把脚本一起贴出。

### 获取真实块地址

循环头的前驱全是真实块除了包含了一个jmp跳转指令块，然后在此基础上增加序言块（一般是函数首地址）和ret块（循环每个块查找含retn指令的块并检查其是否含有后继块）

找循环头(有很多个前驱块)：
其实该程序有两个循环头，一个是真实块的后继，一个是一堆无用单jmp指令的后继。所以无法用平常的思路即检测某个块的前驱数量来判断循环头。这里我观察了多个函数后发现循环头统一是在序言块的后继块的前驱块中。所以直接获取序言块的后继块然后找该后继块的前驱块排除掉序言块后就能获得循环头了。
![](assets/Pasted%20image%2020260522105800.png)

### nop无用块

除真实块外的全是无用块，循环遍历nop掉就行了。


### angr模拟执行获取真实块间关系并进行patch.

分两种情况一种是无条件跳转，另一种是条件跳转。这部分跟普通的ollvm控制流平坦化去除思路很像。含条件跳转的块包含着cmov指令进行判断跳转关系。

#### 无条件跳转
取这一个块分析

![](assets/Pasted%20image%2020260522110546.png)
![](assets/Pasted%20image%2020260522110601.png)
直接跳转关系获取到下一个真实块地址后直接patch掉jmp指令后面4个字节（偏移地址）
偏移地址 = 下一个真实块地址 - jmp下一条指令地址(如图中 rva = 0x140003888(假设) - 0x14000356F)

#### 条件跳转

含有cmov指令（可能是cmovz, cmovnz, cmovg等等) 尽管cmov指令不同但是思路是一样的
以这一个为例
![](assets/Pasted%20image%2020260522111404.png)
![](assets/Pasted%20image%2020260522111516.png)
获取跳转关系：
在cmp eax, 0后cmovnz根据 zf等标志寄存器的值决定是否执行 mov edx, ecx。所以我们在angr模拟执行到cmovnz指令时分别获取edx, ecx寄存器的值然后分裂出两个状态（1.edx值不变 2.edx = ecx)分别单步到下一个真实块地址获得条件跳转关系。
patch:
cmp后面的指令都是无用的，直接nop掉，然后在cmp后建立跳转关系：
jz(或者jnz,然后把地址顺序改了也行) 地址1
jmp 地址2
![](assets/Pasted%20image%2020260522112730.png)
0F 84 (0x840F)是jz 机器码 后面4字节是偏移地址，E9是 jmp机器码 后面四字节是偏移地址
偏移地址计算跟上面一样。

### 脚本


```python
import angr

from idaapi import *

from idc import *

import sys

  

#====================== 通用函数功能 ========================

def get_basic_block(ea):

    func = get_func(ea)

    flowchart = FlowChart(func)

    for block in flowchart:

        if (block.start_ea <= ea < block.end_ea):

            return block

    raise ValueError("get_basic_block error")

  

def get_block_size(block):

    return block.end_ea - block.start_ea

  

def nop_code(addr, length):

    for i in range(length):

        patch_byte(addr + i, 0x90)

    return

  

def CalRva(first_addr, second_addr, inst_size):

    #first_addr是要patch的起始地址, inst_size是first_addr指令数量, second_addr是目标地址

    rva = second_addr - (first_addr + inst_size)

    if rva < 0:

        rva = 0x100000000 - ((first_addr + inst_size) - second_addr)

    return rva & 0xffffffff

  

def get_block_all_disasm(block):

    disasm_string = ""

    start_ea = block.start_ea

    end_ea = block.end_ea

    current_ea = block.start_ea

    #循环获取当前块所有汇编指令字符串

    while current_ea < end_ea:

        disasm_string += GetDisasm(current_ea)

        current_ea = next_head(current_ea, end_ea)

  

    return disasm_string

  

#====================== 通用函数功能 ========================

  
  

#====================== 寻找真实块函数 ============================

def find_loop_head(first_block):

    #根据程序特征进行设置

    succs_block = list(first_block.succs())#获取序言块的后继

    if len(succs_block) > 1:

        raise ValueError("错误 -> find_loop_head错误")

    pred_block = list(succs_block[0].preds())#序言块后继的前驱中包含两个块,一个是序言块,另一个是循环头

    if len(pred_block) > 2:

        raise ValueError("错误 -> find_loop_head出错")

    return pred_block[1].start_ea

  

def find_ret_addr(blocks):

    ret_addr = []

    for block in blocks:

        block_disasm = get_block_all_disasm(block)

        if "retn" in block_disasm and len(list(block.succs())) == 0: #当前块存在retn且没有后继块

            ret_addr.append(block.start_ea)

  

    return ret_addr

  

#====================== 寻找真实块函数 =============================

  
  

#====================== patch真实块关系函数 =========================

def judge_cmov(ea):

    block = get_basic_block(ea)

    start_ea = block.start_ea

    end_ea = block.end_ea

    current_ea = start_ea

  

    #查找cmov类型

    while current_ea < end_ea:

        mnem = print_insn_mnem(current_ea).lower()

        #匹配cmov类型并返回

        if "cmovz" == mnem:

            return "cmovz"

        if "cmovnz" == mnem:

            return "cmovnz"

        if "cmovg" == mnem:

            return "cmovg"

        if "cmovge" == mnem:

            return "cmovge"

        if "cmovl" == mnem:

            return "cmovl"

        current_ea = next_head(current_ea,end_ea)

  

    #再次循环一遍查找是否包含其他cmov类型

    current_ea = start_ea

    while current_ea < end_ea:

        mnem = print_insn_mnem(current_ea).lower()

        if "cmov" in mnem:

            raise ValueError(f"找到cmov但没有处理该cmov类型的功能, 请添加该cmov类型处理功能: {mnem}")

        current_ea = next_head(current_ea, end_ea)

  
  

    #如果不存在cmov则返回None

    return "None"

  

def get_reg_value(proj, current_state, reg_name, addr):

    #单步到目标地址处

    simgr = proj.factory.simgr(current_state)

    while simgr.active[0].addr < addr:

        if len(simgr.active) > 1:

            raise ValueError("get_reg_value错误 -> 出现多个分支")

        simgr.step(num_inst=1)

    current_state = simgr.active[0]

    reg_value = getattr(current_state.regs, reg_name)#等价于current_state.regs.reg_name

    if reg_value == None:

        raise ValueError("get_reg_value错误 -> 获取寄存器值失败")

    return reg_value

  

def find_cmovblock_succs(proj, current_state, finding_blocks, cmov_addr, reg_value):

    #单步执行到cmov指令处

    simgr = proj.factory.simgr(current_state)

    while simgr.active[0].addr < cmov_addr:

        if len(simgr.active) > 1:

            raise ValueError("get_reg_value错误 -> 出现多个分支")

        simgr.step(num_inst=1)    

  

    simgr.step(num_inst=1)#单步跳过cmov指令,然后再对寄存器赋值

    current_state = simgr.active[0]

    reg_name = print_operand(cmov_addr, 0)#获取cmov第一个寄存器名字

    setattr(current_state.regs, reg_name, reg_value) #对第一个寄存器赋值

  

    #寻找后继块

    target_addr = 0

    while len(simgr.active):

        if len(simgr.active) > 1:

            raise ValueError("find_cmovblock_succs错误 -> 出现多个分支")

        for real_block_addr in finding_blocks:

            if simgr.active[0].addr == real_block_addr:

                target_addr = real_block_addr

        if target_addr:

            break

        simgr.step(num_inst=1)

    if target_addr == 0:

        raise ValueError(

            f"find_cmovblock_succs: 未找到后继真实块, cmov={hex(cmov_addr)}"

        )

    return target_addr

  

#特别处理序言块

def for_preface_special(proj, state, simgr, current_real_block, finding_blocks):

    #单步循环

    while len(simgr.active):

        if len(simgr.active) > 1:

            print(simgr.active)

            raise ValueError("patch_block_realation 错误 -> 出现多个分支")

        current_state = simgr.active[0]

        if current_state.addr == current_real_block:

            print(f"找到当前真实块{hex(current_real_block)}, 开始寻找后继真实块")

            #保存当前状态

            currentstate_copy = current_state.copy()

  

            #cmt call 为call 函数进行注释

            #cmt_call(proj, currentstate_copy.copy())

            #这个功能单独写一个脚本运行,因为patch之后保存文件并不会保存在ida里面的注释内容

  

            #寻找后继真实块

            #先判断是否包含cmov指令

            cmov_type = judge_cmov(current_real_block)

  

            #=======================================================================

            #直接跳转

            if cmov_type == "None":

                #获取当前地址块并找到patch地址(jmp 偏移)

                block = get_basic_block(current_state.addr)

                start_ea = block.start_ea

                end_ea = block.end_ea

                patch_addr = end_ea #序言块的下一个块首地址作为patch_addr

                jmp_addr = 0

  

                #查找后继块

                while len(simgr.active):

                    if len(simgr.active) > 1:

                        raise ValueError("错误 -> 直接跳转出现两个分支")

                    for target_real_block in finding_blocks:

                        if simgr.active[0].addr == target_real_block:

                            jmp_addr = target_real_block

                            break

                    if jmp_addr:

                        break

  

                    simgr.step(num_inst=1)

  

                if jmp_addr == 0:

                    raise ValueError(f"未找到 {hex(current_real_block)} 的后继真实块")

                #patch

                #计算偏移rva e9 + 4字节偏移

                rva = CalRva(patch_addr, jmp_addr, 5)

  

                patch_byte(patch_addr, 0xe9)

                patch_dword(patch_addr + 1, rva)

                print(f"直接跳转: {hex(current_real_block)} -> {hex(jmp_addr)}")

                return

  

def patch_block_relation(proj, state, simgr, current_real_block, finding_blocks):

  

    #单步循环

    while len(simgr.active):

        if len(simgr.active) > 1:

            print(simgr.active)

            raise ValueError("patch_block_realation 错误 -> 出现多个分支")

        current_state = simgr.active[0]

        if current_state.addr == current_real_block:

            print(f"找到当前真实块{hex(current_real_block)}, 开始寻找后继真实块")

            #保存当前状态

            currentstate_copy = current_state.copy()

  

            #cmt call 为call 函数进行注释

            #cmt_call(proj, currentstate_copy.copy())

            #这个功能单独写一个脚本运行,因为patch之后保存文件并不会保存在ida里面的注释内容

  

            #寻找后继真实块

            #先判断是否包含cmov指令

            cmov_type = judge_cmov(current_real_block)

  

            #=======================================================================

            #直接跳转

            if cmov_type == "None":

                #获取当前地址块并找到patch地址(jmp 偏移)

                block = get_basic_block(current_state.addr)

                start_ea = block.start_ea

                end_ea = block.end_ea

                patch_addr = prev_head(end_ea)

                jmp_addr = 0

  

                #查找后继块

                while len(simgr.active):

                    if len(simgr.active) > 1:

                        raise ValueError("错误 -> 直接跳转出现两个分支")

                    for target_real_block in finding_blocks:

                        if simgr.active[0].addr == target_real_block:

                            jmp_addr = target_real_block

                            break

                    if jmp_addr:

                        break

  

                    simgr.step(num_inst=1)

  

                if jmp_addr == 0:

                    raise ValueError(f"未找到 {hex(current_real_block)} 的后继真实块")

                #patch

                #计算偏移rva e9 + 4字节偏移

                rva = CalRva(patch_addr, jmp_addr, 5)

  

                patch_byte(patch_addr, 0xe9)

                patch_dword(patch_addr + 1, rva)

                print(f"直接跳转: {hex(current_real_block)} -> {hex(jmp_addr)}")

                return

            #直接跳转end======================================================================

  
  

            #处理cmov条件跳转==================================================================

            match cmov_type:

                #cmovz ==================================================================

                case "cmovz":

                    #获取当前地址块并找到patch地址(jmp 偏移)

                    block = get_basic_block(current_state.addr)

                    start_ea = block.start_ea

                    end_ea = block.end_ea

                    current_ea = start_ea

  

                    #获取cmovz 对应寄存器的值 然后进行模拟执行获取后继块的地址

                    comvz_reg_name = ""

                    cmovnz_reg_name = ""

                    cmovz_reg_value = 0

                    cmovnz_reg_value = 0

                    cmov_addr = 0

                    jz_addr = 0

                    jnz_addr = 0

                    jz_rva = 0 #patch时用的相对偏移地址

                    jnz_rva = 0

  

                    #patch地址和patch大小(cmp的下一条指令)

                    patch_addr = 0

                    patch_size = 0

  

                    #遍历指令并对regname, patch_addr/size变量赋值

                    while current_ea < end_ea:

                        disasm_inst = GetDisasm(current_ea)

                        if "cmovz" in disasm_inst:

                            cmov_addr = current_ea

                            comvz_reg_name = print_operand(current_ea, 1)

                            cmovnz_reg_name = print_operand(current_ea, 0)

  

                        if "cmp" in disasm_inst:

                            patch_addr = next_head(current_ea)

                            patch_size = end_ea - patch_addr

  

                        current_ea = next_head(current_ea,end_ea)

  

                    if cmov_addr == 0:

                        raise ValueError(f"未找到 cmovg: {hex(current_real_block)}")

  

                    if patch_addr == 0:

                        raise ValueError(f"未找到 cmp, 无法生成 jg/jmp: {hex(current_real_block)}")

  

                    #查找cmovreg的值

                    cmovz_reg_value = get_reg_value(proj, currentstate_copy.copy(), comvz_reg_name, cmov_addr)

                    cmovnz_reg_value = get_reg_value(proj, currentstate_copy.copy(), cmovnz_reg_name, cmov_addr)

  

                    #分裂出两条分支，分别单步执行获取后继块地址

                    jz_addr = find_cmovblock_succs(proj, currentstate_copy.copy(), finding_blocks, cmov_addr, cmovz_reg_value)

                    jnz_addr = find_cmovblock_succs(proj, currentstate_copy.copy(), finding_blocks, cmov_addr, cmovnz_reg_value)

  

                    #计算rva jz: 0F 84 加上 4字节偏移 一共6字节

                    jz_rva = CalRva(patch_addr, jz_addr, 6)

                    #jmp : e9 加上 4字节偏移 一共5字节

                    jnz_rva = CalRva(patch_addr + 6, jnz_addr, 5)

  

                    #patch跳转关系0F 84 4字节偏移 + e9 4字节偏移

                    nop_code(patch_addr, patch_size)

                    #patch jz: 0F 84 + jz_rva

                    patch_word(patch_addr, 0x840F)

                    patch_dword(patch_addr + 2, jz_rva)

                    #patch jmp: e9 + jnz_rva

                    patch_byte(patch_addr + 6, 0xe9)

                    patch_dword(patch_addr + 7, jnz_rva)

  

                    print(f"{cmov_type}间接跳转: {hex(current_real_block)} -> {hex(jz_addr), hex(jnz_addr)}")

                    return

  

                #cmovnz==================================================================

                case "cmovnz":

                    #获取当前地址块并找到patch地址(jmp 偏移)

                    block = get_basic_block(current_state.addr)

                    start_ea = block.start_ea

                    end_ea = block.end_ea

                    current_ea = start_ea

  

                    #获取cmovz 对应寄存器的值 然后进行模拟执行获取后继块的地址

                    comvz_reg_name = ""

                    cmovnz_reg_name = ""

                    cmovz_reg_value = 0

                    cmovnz_reg_value = 0

                    cmov_addr = 0

                    jz_addr = 0

                    jnz_addr = 0

                    jz_rva = 0 #patch时用的相对偏移地址

                    jnz_rva = 0

  

                    #patch地址和patch大小(cmp的下一条指令)

                    patch_addr = 0

                    patch_size = 0

  

                    #遍历指令并对regname, patch_addr/size变量赋值

                    while current_ea < end_ea:

                        disasm_inst = GetDisasm(current_ea)

                        if "cmovnz" in disasm_inst:

                            cmov_addr = current_ea

                            comvz_reg_name = print_operand(current_ea, 0)

                            cmovnz_reg_name = print_operand(current_ea, 1)

  

                        if "cmp" in disasm_inst:

                            patch_addr = next_head(current_ea)

                            patch_size = end_ea - patch_addr

  

                        current_ea = next_head(current_ea,end_ea)

  

                    if cmov_addr == 0:

                        raise ValueError(f"未找到 cmovg: {hex(current_real_block)}")

  

                    if patch_addr == 0:

                        raise ValueError(f"未找到 cmp, 无法生成 jg/jmp: {hex(current_real_block)}")

  

                    #查找cmovreg的值

                    cmovz_reg_value = get_reg_value(proj, currentstate_copy.copy(), comvz_reg_name, cmov_addr)

                    cmovnz_reg_value = get_reg_value(proj, currentstate_copy.copy(), cmovnz_reg_name, cmov_addr)

  

                    #分裂出两条分支，分别单步执行获取后继块地址

                    jz_addr = find_cmovblock_succs(proj, currentstate_copy.copy(), finding_blocks, cmov_addr, cmovz_reg_value)

                    jnz_addr = find_cmovblock_succs(proj, currentstate_copy.copy(), finding_blocks, cmov_addr, cmovnz_reg_value)

  

                    #计算rva jz: 0F 84 加上 4字节偏移 一共6字节

                    jz_rva = CalRva(patch_addr, jz_addr, 6)

                    #jmp : e9 加上 4字节偏移 一共5字节

                    jnz_rva = CalRva(patch_addr + 6, jnz_addr, 5)

  

                    #patch跳转关系0F 84 4字节偏移 + e9 4字节偏移

                    nop_code(patch_addr, patch_size)

                    #patch jz: 0F 84 + jz_rva

                    patch_word(patch_addr, 0x840F)

                    patch_dword(patch_addr + 2, jz_rva)

                    #patch jmp: e9 + jnz_rva

                    patch_byte(patch_addr + 6, 0xe9)

                    patch_dword(patch_addr + 7, jnz_rva)

  

                    print(f"{cmov_type}间接跳转: {hex(current_real_block)} -> {hex(jz_addr), hex(jnz_addr)}")

                    return

  

                #comvg====================================================================

                case "cmovg":

                    #获取当前地址块并找到patch地址(jmp 偏移)

                    block = get_basic_block(current_state.addr)

                    start_ea = block.start_ea

                    end_ea = block.end_ea

                    current_ea = start_ea

  

                    #获取cmovg 对应寄存器的值 然后进行模拟执行获取后继块的地址

                    comvg_reg_name = ""

                    cmovng_reg_name = ""

                    cmovg_reg_value = 0

                    cmovng_reg_value = 0

                    cmov_addr = 0

                    jg_addr = 0

                    jng_addr = 0

                    jg_rva = 0 #patch时用的相对偏移地址

                    jng_rva = 0

  

                    #patch地址和patch大小(cmp的下一条指令)

                    patch_addr = 0

                    patch_size = 0

  

                    #遍历指令并对regname, patch_addr/size变量赋值

                    while current_ea < end_ea:

                        disasm_inst = GetDisasm(current_ea)

                        if "cmovg" in disasm_inst:

                            cmov_addr = current_ea

                            comvg_reg_name = print_operand(current_ea, 1)

                            cmovng_reg_name = print_operand(current_ea, 0)

  

                        if "cmp" in disasm_inst:

                            patch_addr = next_head(current_ea)

                            patch_size = end_ea - patch_addr

  

                        current_ea = next_head(current_ea,end_ea)

  

                    if cmov_addr == 0:

                        raise ValueError(f"未找到 cmovg: {hex(current_real_block)}")

  

                    if patch_addr == 0:

                        raise ValueError(f"未找到 cmp, 无法生成 jg/jmp: {hex(current_real_block)}")

  

                    #查找cmovreg的值

                    cmovg_reg_value = get_reg_value(proj, currentstate_copy.copy(), comvg_reg_name, cmov_addr)

                    cmovng_reg_value = get_reg_value(proj, currentstate_copy.copy(), cmovng_reg_name, cmov_addr)

  

                    #分裂出两条分支，分别单步执行获取后继块地址

                    jg_addr = find_cmovblock_succs(proj, currentstate_copy.copy(), finding_blocks, cmov_addr, cmovg_reg_value)

                    jng_addr = find_cmovblock_succs(proj, currentstate_copy.copy(), finding_blocks, cmov_addr, cmovng_reg_value)

  

                    #计算rva jg: 0F 8F 加上 4字节偏移 一共6字节

                    jg_rva = CalRva(patch_addr, jg_addr, 6)

                    #jmp : e9 加上 4字节偏移 一共5字节

                    jng_rva = CalRva(patch_addr + 6, jng_addr, 5)

  

                    #patch跳转关系0F 8F 4字节偏移 + e9 4字节偏移

                    nop_code(patch_addr, patch_size)

                    #patch jg: 0F 8F + jg_rva

                    patch_word(patch_addr, 0x8F0F)

                    patch_dword(patch_addr + 2, jg_rva)

                    #patch jmp: e9 + jng_rva

                    patch_byte(patch_addr + 6, 0xe9)

                    patch_dword(patch_addr + 7, jng_rva)

  

                    print(f"{cmov_type}间接跳转: {hex(current_real_block)} -> {hex(jg_addr), hex(jng_addr)}")

                    return

  

                #cmovge===============================================================

                case "cmovge":

                    #获取当前地址块并找到patch地址(jmp 偏移)

                    block = get_basic_block(current_state.addr)

                    start_ea = block.start_ea

                    end_ea = block.end_ea

                    current_ea = start_ea

  

                    #获取cmovg 对应寄存器的值 然后进行模拟执行获取后继块的地址

                    comvg_reg_name = ""

                    cmovng_reg_name = ""

                    cmovg_reg_value = 0

                    cmovng_reg_value = 0

                    cmov_addr = 0

                    jg_addr = 0

                    jng_addr = 0

                    jg_rva = 0 #patch时用的相对偏移地址

                    jng_rva = 0

  

                    #patch地址和patch大小(cmp的下一条指令)

                    patch_addr = 0

                    patch_size = 0

  

                    #遍历指令并对regname, patch_addr/size变量赋值

                    while current_ea < end_ea:

                        disasm_inst = GetDisasm(current_ea)

                        if "cmovge" in disasm_inst:

                            cmov_addr = current_ea

                            comvg_reg_name = print_operand(current_ea, 1)

                            cmovng_reg_name = print_operand(current_ea, 0)

  

                        if "cmp" in disasm_inst:

                            patch_addr = next_head(current_ea)

                            patch_size = end_ea - patch_addr

  

                        current_ea = next_head(current_ea,end_ea)

  

                    if cmov_addr == 0:

                        raise ValueError(f"未找到 cmovg: {hex(current_real_block)}")

  

                    if patch_addr == 0:

                        raise ValueError(f"未找到 cmp, 无法生成 jg/jmp: {hex(current_real_block)}")

  

                    #查找cmovreg的值

                    cmovg_reg_value = get_reg_value(proj, currentstate_copy.copy(), comvg_reg_name, cmov_addr)

                    cmovng_reg_value = get_reg_value(proj, currentstate_copy.copy(), cmovng_reg_name, cmov_addr)

  

                    #分裂出两条分支，分别单步执行获取后继块地址

                    jg_addr = find_cmovblock_succs(proj, currentstate_copy.copy(), finding_blocks, cmov_addr, cmovg_reg_value)

                    jng_addr = find_cmovblock_succs(proj, currentstate_copy.copy(), finding_blocks, cmov_addr, cmovng_reg_value)

  

                    #计算rva jg: 0F 8F 加上 4字节偏移 一共6字节

                    jg_rva = CalRva(patch_addr, jg_addr, 6)

                    #jmp : e9 加上 4字节偏移 一共5字节

                    jng_rva = CalRva(patch_addr + 6, jng_addr, 5)

  

                    #patch跳转关系0F 8F 4字节偏移 + e9 4字节偏移

                    nop_code(patch_addr, patch_size)

                    #patch jg: 0F 8F + jg_rva

                    patch_word(patch_addr, 0x8D0F)

                    patch_dword(patch_addr + 2, jg_rva)

                    #patch jmp: e9 + jng_rva

                    patch_byte(patch_addr + 6, 0xe9)

                    patch_dword(patch_addr + 7, jng_rva)

  

                    print(f"{cmov_type}间接跳转: {hex(current_real_block)} -> {hex(jg_addr), hex(jng_addr)}")                

  

                    return  

                #cmovl================================================================

                case "cmovl":

                    #获取当前地址块并找到patch地址(jmp 偏移)

                    block = get_basic_block(current_state.addr)

                    start_ea = block.start_ea

                    end_ea = block.end_ea

                    current_ea = start_ea

  

                    #获取cmovg 对应寄存器的值 然后进行模拟执行获取后继块的地址

                    comvl_reg_name = ""

                    cmovnl_reg_name = ""

                    cmovl_reg_value = 0

                    cmovnl_reg_value = 0

                    cmov_addr = 0

                    jl_addr = 0

                    jnl_addr = 0

                    jl_rva = 0 #patch时用的相对偏移地址

                    jnl_rva = 0

  

                    #patch地址和patch大小(cmp的下一条指令)

                    patch_addr = 0

                    patch_size = 0

  

                    #遍历指令并对regname, patch_addr/size变量赋值

                    while current_ea < end_ea:

                        disasm_inst = GetDisasm(current_ea)

                        if "cmovl" in disasm_inst:

                            cmov_addr = current_ea

                            comvl_reg_name = print_operand(current_ea, 1)

                            cmovnl_reg_name = print_operand(current_ea, 0)

  

                        if "cmp" in disasm_inst:

                            patch_addr = next_head(current_ea)

                            patch_size = end_ea - patch_addr

  

                        current_ea = next_head(current_ea,end_ea)

  

                    if cmov_addr == 0:

                        raise ValueError(f"未找到 cmovg: {hex(current_real_block)}")

  

                    if patch_addr == 0:

                        raise ValueError(f"未找到 cmp, 无法生成 jl/jmp: {hex(current_real_block)}")

  

                    #查找cmovreg的值

                    cmovl_reg_value = get_reg_value(proj, currentstate_copy.copy(), comvl_reg_name, cmov_addr)

                    cmovnl_reg_value = get_reg_value(proj, currentstate_copy.copy(), cmovnl_reg_name, cmov_addr)

  

                    #分裂出两条分支，分别单步执行获取后继块地址

                    jl_addr = find_cmovblock_succs(proj, currentstate_copy.copy(), finding_blocks, cmov_addr, cmovl_reg_value)

                    jnl_addr = find_cmovblock_succs(proj, currentstate_copy.copy(), finding_blocks, cmov_addr, cmovnl_reg_value)

  

                    #计算rva jg: 0F 8F 加上 4字节偏移 一共6字节

                    jl_rva = CalRva(patch_addr, jl_addr, 6)

                    #jmp : e9 加上 4字节偏移 一共5字节

                    jnl_rva = CalRva(patch_addr + 6, jnl_addr, 5)

  

                    #patch跳转关系0F 8F 4字节偏移 + e9 4字节偏移

                    nop_code(patch_addr, patch_size)

                    #patch jg: 0F 8F + jg_rva

                    patch_word(patch_addr, 0x8C0F)

                    patch_dword(patch_addr + 2, jl_rva)

                    #patch jmp: e9 + jng_rva

                    patch_byte(patch_addr + 6, 0xe9)

                    patch_dword(patch_addr + 7, jnl_rva)

  

                    print(f"{cmov_type}间接跳转: {hex(current_real_block)} -> {hex(jl_addr), hex(jnl_addr)}")        

                    return

            #处理cmov条件跳转end=============================================================================

  

        simgr.step(num_inst=1)

  

#====================== patch真实块关系函数 =========================

  
  

#========================== 主函数 ===================================

def get_real_block():

    here_addr = here() #获取当前鼠标光标所在地址

    func = get_func(here_addr) #获取当前地址的函数

    blocks = FlowChart(func) #获取当前函数所有块

    first_block = blocks[0] #获取第一块地址 (一般是序言块)

  

    #增加序言块

    real_block.append(first_block.start_ea)#序言

    print(f"序言块 -> {hex(first_block.start_ea)}")

  

    #寻找循环头, 根据程序特征进行修改

    loop_head = find_loop_head(first_block)

    print(f"循环头 -> {hex(loop_head)}")

    loop_block = get_basic_block(loop_head)#循环头所在块

  

    #查找真实块（循环头前驱）

    for block in loop_block.preds():

  

        if get_block_size(block) < 6:

            continue

        real_block.append(block.start_ea)

  

    real_block.sort()

    #增加ret块,把它放在最后

    ret_addr = find_ret_addr(blocks)

    real_block.extend(ret_addr)#可能有多个ret块

    ret_block.extend(ret_addr) #为后面patch提供ret块

    for retAddr in ret_addr:

        print(f"ret块 -> {hex(retAddr)}")

  

    print("-------------------------------real block-----------------------------")

    print(f"real_block: {real_block} , 真实块数量{len(real_block)}")

  

    for block in real_block:

        print(hex(block))

  

def angr_main():

    preface_startAddr = get_basic_block(real_block[0]).start_ea#获取序言块起始地址

    hook_addr = 0x14000108A# hook地址，取序言块的后继块的第一条指令地址

  

    #初始化项目

    proj = angr.Project("D:\\00wangan\\00competition\\2026PolarisCTF\\easyre\\server.exe", auto_load_libs=False)

    init_state = proj.factory.blank_state(addr=preface_startAddr)#preface_startAddr

    init_state.options.add(angr.options.CALLLESS)

  
  

    #循环真实块

    for current_real_block in real_block:

        #每次循环使用初始化状态模拟

        state = init_state.copy()

        simgr = proj.factory.simgr(state)

  

        #在所有真实块中去除掉当前真实块的地址,为了后面遍历匹配后继真实块

        finding_blocks = real_block.copy()

        finding_blocks.remove(current_real_block)

  

        #ret块直接继续循环

        if current_real_block in ret_block:

            continue

        #处理序言块

        if current_real_block == real_block[0]:

            #nop最后一条指令

            block = get_basic_block(current_real_block)

            patch_addr = block.end_ea #因为序言最后一条指令不是跳转指令而且有用, 所以patch下一个块的第一条指令

            nop_size = block.end_ea - patch_addr #该情况下nop_size为0但不影响后面patch

            nop_code(patch_addr, nop_size)

  

            for_preface_special(proj, state, simgr, current_real_block, finding_blocks) #特殊处理序言块

            continue

  

        #hook序言最后一条指令

        def jmp_to_address(state, target = current_real_block):

            print("hook: ",simgr.active[0]) #0x14000109D

            proj.unhook(hook_addr)

            state.regs.ip = target

        proj.hook(hook_addr, jmp_to_address)

  

        #patch_block_relation

        patch_block_relation(proj, state, simgr, current_real_block, finding_blocks)

    print("patch_relation end")

  

def nop_useless_block():

    func_addr = real_block[0]

    func = get_func(func_addr)

    flowchart = FlowChart(func)

    for block in flowchart:

        useless_Flag = True

        for real_block_addr in real_block:

            if block.start_ea == real_block_addr:

                useless_Flag = False

        if useless_Flag:

            block_size = get_block_size(block)

            nop_code(block.start_ea, block_size)

    return

#========================== 主函数 ===================================

  
  
  
  
  

def main():

    #获取所有真实块地址

    get_real_block()

    nop_useless_block()

    angr_main()

  
  

#base = 0x140000000 #基址

#真实块地址

real_block = []

ret_block = []

main()
```






## 关于去混淆脚本

在我看来，并没有什么通用脚本能够去除所有不同类型的ollvm混淆。在遇到不同类型的ollvm混淆时需要根据其特征进行脚本的编写。



# 寄存器间接调用

在去除寄存器间接跳转混淆后虽然f5能够反编译了，但是由于call reg寄存器间接调用的存在，我们无法直接知道代码调用了哪个函数。

![](assets/Pasted%20image%2020260522122923.png)![](assets/Pasted%20image%2020260522122950.png)

## 思路:
我尝试了几个思路最后只有直接记录call reg寄存器的值并添加注释能行。
### 1.往call后面patch:
一开始我的思路是直接把call reg改为 call 偏移地址，但是因为call reg占2个字节而call 偏移地址占 5个字节，这样直接patch可能会影响到后面有用的代码。
![](assets/Pasted%20image%2020260522123907.png)

![](assets/Pasted%20image%2020260522123936.png)


### 2.理论可行但脚本编写十分困难

#### 2.1完全理解传参方式后patch

call reg混淆会增加很多多余计算，我们把这些全nop掉然后手动模拟传参形式如: mov rcx, xxxx, mov rdx ,xxxx, 等等把所有要传入的参数提前写进对应寄存器和栈中，然后call 偏移地址实现函数调用。但是我无法准确的确认每个函数在调用前到底传入了多少个参数，又是否依赖之前的寄存器等等因素，脚本写起来十分困难

#### 2.2记录所有寄存器在call之前的值

无需理解传参过程，记录call之前所有寄存器和栈最后一次的值，然后patch掉计算过程，把每个寄存器最后一次的值直接写入。但是寄存器和栈变量太多，脚本写起来很麻烦

#### 2.3把整个块中无用字节码用来补充call eax -> call 偏移 多出的字节

前面提到了call 偏移会可能会覆盖掉后面有用的指令，所以我们可以nop掉一些有规律的无用指令（指在每个块中特定位置固定出现的无用指令）用来补充call 偏移地址 所需要的字节码数量。但是我并没有找到这个规律。


### 3.记录call reg 寄存器值并进行注释

这是我能想到不麻烦而且能比较清晰得看到反编译代码中所调用函数的地址了。
直接用angr模拟执行，遇到call reg后记录reg值并进行注释就行了
脚本如下

```python
import angr

from idaapi import *

from idc import *

  

def get_basic_block(ea):

    func = get_func(ea)

    flowchart = FlowChart(func)

    for block in flowchart:

        if (block.start_ea <= ea < block.end_ea):

            return block

    raise ValueError("get_basic_block error")

  

def get_block_size(block):

    return block.end_ea - block.start_ea

  
  

def cmt_call(proj, current_state):

    #初始化simgr, 获取块起始地址和终止地址

    simgr = proj.factory.simgr(current_state)

    state_addr = current_state.addr

    current_block = get_basic_block(state_addr)

    start_ea = current_block.start_ea

    end_ea = current_block.end_ea

  

    while len(simgr.active) and start_ea < simgr.active[0].addr < end_ea:

        if len(simgr.active) > 1:

            raise ValueError("cmt_call错误 -> 出现多个分支")

        current_ea = simgr.active[0].addr

        current_disasm = GetDisasm(current_ea) # 读取当前地址汇编指令字符串

        current_disasm_sz = next_head(current_ea,end_ea) - current_ea #获取call指令大小

        current_reg_name = print_operand(current_ea, 0) #获取call 寄存器的名字

  

        if "call" in current_disasm and current_disasm_sz < 4: #call 寄存器间接调用 字节码大小为2

            current_state = simgr.active[0]

            print(current_reg_name)

            call_reg_value = getattr(current_state.regs, current_reg_name)#获取当前寄存器跳转地址

            set_cmt(current_ea, f"-> sub_{(call_reg_value)}", 1)#添加注释

        simgr.step(num_inst=1)

  

    return

  

func_addr = 0x140001000

func = get_func(func_addr)

blocks = FlowChart(func)

  

file_path = get_input_file_path()

proj = angr.Project(file_path, auto_load_libs=False)

init_state = proj.factory.blank_state(addr=0x140001000)#0x140001000

init_state.options.add(angr.options.CALLLESS)

  

#获取所有块地址

block_addrs = []

for block in blocks:

    block_addrs.append(block.start_ea)

  

#循环处理每个块的call

for block_addr in block_addrs:

    state = init_state.copy()

    simgr = proj.factory.simgr(state)

  

    #hook序言尾地址到目标块地址

    def hook_addr(state, target = block_addr):

        proj.unhook(0x14000107F)

        state.regs.ip = target

    proj.hook(0x14000107F, hook_addr)

  

    while len(simgr.active):

        if len(simgr.active) > 1:

            raise ValueError("main错误 -> 出现多个分支")

        #hook到目标块地址

        if simgr.active[0].addr == block_addr:

            simgr.step(num_inst=1)#更新hook状态

            cmt_call(proj, simgr.active[0].copy())

            break

        simgr.step(num_inst=1)

  
  

print("---------------ok----------------")
```

效果:
双击注释地址可以跳转到对应函数，然后发现又是混淆hhhh。分析起来依旧很困难。没招了.
![](assets/Pasted%20image%2020260522130646.png)


# 最后

我尝试使用gpt让它直接生成伪C代码，发现效果挺好的
```c
// Architecture: Windows PE32+ x86-64, little-endian

// --- Function: main_server @ 0x140001000 ---
int main(int argc, char **argv) {
    int ret = 0;
    WSADATA wsa;
    SOCKET listen_sock = INVALID_SOCKET;
    SOCKET client_sock = INVALID_SOCKET;

    /*
     * server main 同样是控制流平坦化：
     *   初始 state = 0x4459A900
     *   WSAStartup 后根据返回值进入 0xE56A5596 / 0x54D5780D
     * 已还原为正常 if/else。
     */

    if (WSAStartup(0x0202, &wsa) != 0) {
        char msg[19];

        // table +0x10 -> 0x140006EF0
        // decoded from server string pool:
        //   "WSAStartup failed\n"
        decode_xor_20_19_server(msg, SERVER_OBF_STR_WSASTARTUP_FAILED);

        server_print_or_log(msg);
        return 1;
    }

    listen_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (listen_sock == INVALID_SOCKET) {
        server_print_or_log(server_decode_string_socket_failed());
        WSACleanup();
        return 1;
    }

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));

    addr.sin_family = AF_INET;

    /*
     * 原始代码通过：
     *   htonl(...)
     *   htons(...)
     *   bind(...)
     * 其中地址/端口常量被拆分在状态块与字符串池中。
     */
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(server_decode_listen_port());

    if (bind(listen_sock, (struct sockaddr *)&addr, sizeof(addr)) != 0) {
        server_print_or_log(server_decode_string_bind_failed());
        closesocket(listen_sock);
        WSACleanup();
        return 1;
    }

    if (listen(listen_sock, SOMAXCONN) != 0) {
        server_print_or_log(server_decode_string_listen_failed());
        closesocket(listen_sock);
        WSACleanup();
        return 1;
    }

    server_print_or_log(server_decode_string_waiting_client());

    client_sock = accept(listen_sock, NULL, NULL);
    if (client_sock == INVALID_SOCKET) {
        server_print_or_log(server_decode_string_accept_failed());
        closesocket(listen_sock);
        WSACleanup();
        return 1;
    }

    uint8_t request[512];
    int recv_len = recv(client_sock, (char *)request, sizeof(request), 0);
    if (recv_len <= 0) {
        server_print_or_log(server_decode_string_recv_failed());
        closesocket(client_sock);
        closesocket(listen_sock);
        WSACleanup();
        return 1;
    }

    /*
     * 服务端业务逻辑：
     *   1. 使用 BCrypt 派生/生成对称密钥；
     *   2. 解密客户端数据；
     *   3. 对关键字段做 hash / compare；
     *   4. 加密结果并 send 回客户端。
     *
     * 服务端额外导入了 hash 相关 BCrypt API：
     *   BCryptCreateHash
     *   BCryptHashData
     *   BCryptFinishHash
     *   BCryptDestroyHash
     */
    ServerCryptoContext crypto;
    if (!server_crypto_init(&crypto)) {
        server_print_or_log(server_decode_string_crypto_failed());
        closesocket(client_sock);
        closesocket(listen_sock);
        WSACleanup();
        return 1;
    }

    ServerRequest parsed;
    bool ok = server_decrypt_and_parse_request(
        &crypto,
        request,
        (size_t)recv_len,
        &parsed
    );

    if (!ok) {
        uint8_t fail_packet[256];
        size_t fail_len = server_build_encrypted_reply(
            &crypto,
            false,
            fail_packet,
            sizeof(fail_packet)
        );

        if (fail_len != 0) {
            send(client_sock, (const char *)fail_packet, (int)fail_len, 0);
        }

        server_crypto_free(&crypto);
        closesocket(client_sock);
        closesocket(listen_sock);
        WSACleanup();
        return 1;
    }

    bool valid = server_check_credentials(&parsed);

    uint8_t reply[256];
    size_t reply_len = server_build_encrypted_reply(
        &crypto,
        valid,
        reply,
        sizeof(reply)
    );

    if (reply_len != 0) {
        send(client_sock, (const char *)reply, (int)reply_len, 0);
    }

    server_crypto_free(&crypto);
    closesocket(client_sock);
    closesocket(listen_sock);
    WSACleanup();

    return valid ? 0 : 1;
}

```
如今结合ai去除混淆是大势所趋。ai极大得提升了逆向效率和效果，甚至能够独立完成整个逆向过程。后面我也会尝试使用ida-mcp或者别的方法辅助去混淆过程（明明ai自己就能搞定了，人类这不是捣乱吗qwq）