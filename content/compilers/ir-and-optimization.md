---
title: 编译器中间表示与优化
date: 2026-04-13
description: 编译器IR类型、优化遍与代码生成基础
tags:
  - compilers
  - ir
  - optimization
  - code-generation
draft: false
permalink:
---

# 编译器中间表示与优化

中间表示（IR）是编译器前端与后端之间的桥梁，良好的IR设计使优化和代码生成更加高效。

## 编译器流水线

```
源代码 → 词法分析 → 语法分析 → 语义分析 → IR生成 
                                                    ↓
                                              优化遍（多个）
                                                    ↓
                                              代码生成 → 目标代码
```

## 中间表示类型

### 三地址码（Three-Address Code, TAC）

每条指令最多三个操作数：

```
t1 = a + b        // 加法
t2 = t1 * c       // 乘法
if t2 > 0 goto L1 // 条件跳转
```

```python
# TAC指令类型
from enum import Enum
from dataclasses import dataclass

class OpCode(Enum):
    # 算术运算
    ADD = "+"
    SUB = "-"
    MUL = "*"
    DIV = "/"
    MOD = "%"
    
    # 比较运算
    EQ = "=="
    NE = "!="
    LT = "<"
    GT = ">"
    LE = "<="
    GE = ">="
    
    # 逻辑运算
    AND = "&&"
    OR = "||"
    NOT = "!"
    
    # 内存操作
    LOAD = "load"
    STORE = "store"
    LOAD_ADDR = "lea"
    
    # 控制流
    GOTO = "goto"
    IF_GOTO = "if_goto"
    CALL = "call"
    RETURN = "return"
    PARAM = "param"

@dataclass
class Instruction:
    opcode: OpCode
    result: str = None
    arg1: str = None
    arg2: str = None
    label: str = None  # 用于跳转目标
    
    def __str__(self):
        if self.result and self.arg1 and self.arg2:
            return f"{self.result} = {self.arg1} {self.opcode.value} {self.arg2}"
        elif self.result and self.arg1:
            if self.opcode == OpCode.NOT:
                return f"{self.result} = {self.opcode.value}{self.arg1}"
            return f"{self.result} = {self.arg1}"
        elif self.label:
            if self.opcode == OpCode.IF_GOTO:
                return f"if {self.arg1} {self.argcode.value} {self.arg2} goto {self.label}"
            return f"{self.label}:"
        elif self.opcode == OpCode.GOTO:
            return f"goto {self.label}"
        elif self.opcode == OpCode.RETURN:
            return f"return {self.arg1}" if self.arg1 else "return"
        else:
            return str(self.opcode)

# TAC生成示例
class TACGenerator:
    def __init__(self):
        self.instructions = []
        self.temp_count = 0
        self.label_count = 0
    
    def new_temp(self):
        temp = f"t{self.temp_count}"
        self.temp_count += 1
        return temp
    
    def new_label(self):
        label = f"L{self.label_count}"
        self.label_count += 1
        return label
    
    def emit(self, opcode, result=None, arg1=None, arg2=None, label=None):
        self.instructions.append(
            Instruction(opcode, result, arg1, arg2, label)
        )
    
    def generate_if(self, condition, then_block, else_block):
        end_label = self.new_label()
        else_label = else_block if else_block else end_label
        
        # 条件跳转
        self.emit(OpCode.IF_GOTO, label=else_label, arg1=condition[0], argcode=condition[1], arg2=condition[2])
        
        # then块
        self.generate(then_block)
        self.emit(OpCode.GOTO, label=end_label)
        
        # else块
        if else_block:
            self.emit(OpCode.IF_GOTO, label=else_label)  # 标签
            self.generate(else_block)
        
        self.emit(OpCode.IF_GOTO, label=end_label)  # end标签
    
    def generate(self, ast):
        # AST到TAC的转换
        pass
```

### SSA（Static Single Assignment）

每个变量只被赋值一次：

```
// 普通TAC
t1 = a + b
t1 = t1 * c    // 同一个变量被赋值两次

// SSA
t1 = a + b
t2 = t1 * c    // 使用新变量
```

```python
# SSA生成
class SSAGenerator:
    def __init__(self):
        self.current_vars = {}  # 变量到SSA名的映射
        self.version_counter = {}
        self.phi_functions = []
    
    def get_ssa_name(self, var):
        if var in self.current_vars:
            return self.current_vars[var]
        return var
    
    def new_version(self, var):
        if var not in self.version_counter:
            self.version_counter[var] = 0
        
        self.version_counter[var] += 1
        ssa_name = f"{var}.{self.version_counter[var]}"
        self.current_vars[var] = ssa_name
        return ssa_name
    
    def emit_assignment(self, target, op1, operator=None, op2=None):
        target_ssa = self.new_version(target)
        op1_ssa = self.get_ssa_name(op1)
        op2_ssa = self.get_ssa_name(op2) if op2 else None
        
        if operator:
            return Instruction(OpCode.ASSIGN, target_ssa, op1_ssa, operator, op2_ssa)
        else:
            return Instruction(OpCode.ASSIGN, target_ssa, op1_ssa)
    
    def emit_phi(self, var, *labels):
        """生成phi函数"""
        target_ssa = self.new_version(var)
        phi = PhiInstruction(target_ssa, var, list(labels))
        self.phi_functions.append(phi)
        return phi
```

### 控制流图（CFG）

```python
class BasicBlock:
    def __init__(self, label):
        self.label = label
        self.instructions = []
        self.predecessors = []
        self.successors = []
    
    def add_instruction(self, instr):
        self.instructions.append(instr)

class ControlFlowGraph:
    def __init__(self):
        self.blocks = {}
        self.entry_block = None
        self.exit_block = None
    
    def add_block(self, block):
        self.blocks[block.label] = block
        if not self.entry_block:
            self.entry_block = block
    
    def add_edge(self, from_block, to_block):
        from_block.successors.append(to_block)
        to_block.predecessors.append(from_block)
    
    def dominators(self):
        """计算支配节点"""
        dom = {block: set(self.blocks.keys()) for block in self.blocks}
        dom[self.entry_block] = {self.entry_block}
        
        changed = True
        while changed:
            changed = False
            for block in self.blocks.values():
                if block == self.entry_block:
                    continue
                
                new_dom = {block}
                for pred in block.predecessors:
                    new_dom &= dom[pred]
                new_dom.add(block)
                
                if new_dom != dom[block]:
                    dom[block] = new_dom
                    changed = True
        
        return dom
```

## 优化遍

### 常量折叠

```python
def constant_folding(instructions):
    """常量折叠：将编译时可计算的表达式直接求值"""
    folded = []
    
    for instr in instructions:
        if instr.opcode in {OpCode.ADD, OpCode.SUB, OpCode.MUL, OpCode.DIV}:
            # 检查操作数是否都是常量
            if is_constant(instr.arg1) and is_constant(instr.arg2):
                result = compute_constant(
                    get_constant_value(instr.arg1),
                    instr.opcode,
                    get_constant_value(instr.arg2)
                )
                folded.append(Instruction(OpCode.ASSIGN, instr.result, str(result)))
            else:
                folded.append(instr)
        else:
            folded.append(instr)
    
    return folded

def is_constant(value):
    try:
        float(value)
        return True
    except:
        return False

def compute_constant(a, opcode, b):
    ops = {
        OpCode.ADD: a + b,
        OpCode.SUB: a - b,
        OpCode.MUL: a * b,
        OpCode.DIV: a / b if b != 0 else 0,
    }
    return ops.get(opcode, None)
```

### 拷贝传播

```python
def copy_propagation(instructions):
    """拷贝传播：用赋值的右值替换变量"""
    assignments = {}  # 变量 -> 值
    
    optimized = []
    for instr in instructions:
        if instr.opcode == OpCode.ASSIGN and instr.arg2 is None:
            # 简单赋值 x = y
            if instr.arg1 in assignments:
                assignments[instr.result] = assignments[instr.arg1]
            else:
                assignments[instr.result] = instr.arg1
            optimized.append(instr)
        elif instr.result:
            # 结果变量不再等于之前的赋值
            if instr.result in assignments:
                del assignments[instr.result]
            optimized.append(instr)
        else:
            optimized.append(instr)
    
    return optimized
```

### 死代码消除

```python
def dead_code_elimination(instructions):
    """死代码消除：移除不影响程序输出的代码"""
    
    # 计算活跃变量
    live_vars = set()
    used_vars = set()
    
    # 反向遍历
    for instr in reversed(instructions):
        if instr.opcode in {OpCode.ASSIGN}:
            if instr.result and instr.result not in used_vars:
                # 结果未被使用，可能是死代码
                continue
            
            if instr.result:
                used_vars.discard(instr.result)
            
            if instr.arg1:
                used_vars.add(instr.arg1)
            if instr.arg2:
                used_vars.add(instr.arg2)
        
        live_vars.update(used_vars)
    
    # 只保留活跃指令
    return [instr for instr in instructions 
            if instr.result is None or instr.result in live_vars]
```

### 循环优化

```python
def loop_invariant_code_motion(instructions):
    """循环不变代码外提：将不依赖循环变量的表达式移到循环外"""
    
    # 识别循环不变计算
    loop_invariants = []
    
    for instr in instructions:
        if is_loop_invariant(instr, loop_vars):
            loop_invariants.append(instr)
    
    # 将不变计算移到循环前
    return loop_invariants + [instr for instr in instructions 
                               if instr not in loop_invariants]

def is_loop_invariant(instr, loop_vars):
    """检查指令是否循环不变"""
    if instr.arg1 and instr.arg1 in loop_vars:
        return False
    if instr.arg2 and instr.arg2 in loop_vars:
        return False
    return True
```

## 代码生成

### 寄存器分配

```python
class RegisterAllocator:
    def __init__(self, max_registers=8):
        self.max_registers = max_registers
        self.allocated = {}  # 变量 -> 寄存器
        self.spilled = []   # 溢出到内存的变量
    
    def allocate(self, var):
        if var in self.allocated:
            return self.allocated[var]
        
        # 尝试分配寄存器
        for reg in range(self.max_registers):
            if reg not in self.allocated.values():
                self.allocated[var] = reg
                return reg
        
        # 溢出到内存
        self.spilled.append(var)
        return None  # 使用内存
    
    def spill(self, var, stack_offset):
        """生成溢出代码"""
        return [
            Instruction(OpCode.STORE, f"{stack_offset}(%rbp)", None, var),
            Instruction(OpCode.LOAD, var, None, f"{stack_offset}(%rbp)")
        ]
```

### 目标代码生成

```python
class CodeGenerator:
    def __init__(self):
        self.instructions = []
        self.reg_alloc = RegisterAllocator()
    
    def generate(self, tac_instructions):
        for tac in tac_instructions:
            if tac.opcode == OpCode.ADD:
                self.gen_add(tac)
            elif tac.opcode == OpCode.ASSIGN:
                self.gen_mov(tac)
            elif tac.opcode == OpCode.IF_GOTO:
                self.gen_cond_branch(tac)
            elif tac.opcode == OpCode.GOTO:
                self.gen_jump(tac)
            # ... 其他指令
    
    def gen_add(self, instr):
        # t3 = t1 + t2
        reg1 = self.reg_alloc.allocate(instr.arg1)
        reg2 = self.reg_alloc.allocate(instr.arg2)
        result_reg = self.reg_alloc.allocate(instr.result)
        
        self.instructions.append(f"movl %{self.reg_name(reg1)}, %{self.reg_name(result_reg)}")
        self.instructions.append(f"addl %{self.reg_name(reg2)}, %{self.reg_name(result_reg)}")
    
    def gen_mov(self, instr):
        src = self.reg_alloc.allocate(instr.arg1)
        dst = self.reg_alloc.allocate(instr.result)
        self.instructions.append(f"movl %{self.reg_name(src)}, %{self.reg_name(dst)}")
    
    def gen_cond_branch(self, instr):
        # if t1 op t2 goto label
        reg1 = self.reg_alloc.allocate(instr.arg1)
        reg2 = self.reg_alloc.allocate(instr.arg2)
        
        cond_jumps = {
            OpCode.EQ: "je",
            OpCode.NE: "jne",
            OpCode.LT: "jl",
            OpCode.GT: "jg",
            OpCode.LE: "jle",
            OpCode.GE: "jge",
        }
        
        self.instructions.append(f"cmpl %{self.reg_name(reg2)}, %{self.reg_name(reg1)}")
        self.instructions.append(f"{cond_jumps[instr.opcode]} {instr.label}")
    
    def gen_jump(self, instr):
        self.instructions.append(f"jmp {instr.label}")
    
    def reg_name(self, reg_num):
        regs = ["%eax", "%ebx", "%ecx", "%edx", "%esi", "%edi", "%ebp", "%esp"]
        return regs[reg_num] if reg_num is not None else "memory"
```

## LLVM IR简介

```llvm
; 简单函数编译后的LLVM IR
define i32 @add(i32 %a, i32 %b) {
entry:
  %result = add i32 %a, %b
  ret i32 %result
}

; 含控制流
define i32 @max(i32 %a, i32 %b) {
entry:
  %cmp = icmp sgt i32 %a, %b
  br i1 %cmp, label %then, label %else

then:
  ret i32 %a

else:
  ret i32 %b
}
```

## 参考

[^1]: Engineering a Compiler - Cooper & Torczon
[^2]: SSA Book - Cytron et al.
[^3]: LLVM Language Reference

