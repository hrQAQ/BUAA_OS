# lab3-2-Extra 

`Extra `的 `one thing you fail to consider` 到最后还是没`figure out`。课下大手子（此处@lyh）告诉我这是延迟槽没考虑到。然而我已然忘却延迟槽...属实给软院丢人了

**重铸软院荣光，吾辈义不容辞**（逃）

- Tips 1 延迟槽

  查阅《See Mips Run Linux》得知

  > - Unambiguous proof of guilt: 
  >
  > After any exception, the CPU control register EPC points to the correct place to restart execution after the exception is dealt with. 
  >
  > In most cases, it points to the exception victim, but **if the victim was in a branch delay slot, EPC points to the preceding branch instruction**: Returning to the branch instruction will re-execute the victim instruction, but returning to the victim would cause the branch to be ignored. When the victim is in a branch delay slot, the cause register bit Cause(BD) is set.

  加粗的语句告诉我们: **当AdEL异常发生在延迟槽时，中断返回时指令保存到EPC的是延迟槽指令上一条的指令地址,即分支跳转指令**

  Q：为什么还有这一条奇怪的规则呢，EPC难道不应该精准定位到异常发生位置吗?

  这个问题在《See Mips Run Linux》也给出了解释

  > Returning to the branch instruction will re-execute the victim instruction, but returning to the victim would cause the branch to be ignored. 

  - MIPS的分支跳转指令流是：

  	分支跳转指令 -> 延时槽指令 -> 目标跳转地址的指令

  - 如果PC在延时槽指令地址发生中断，中断写入EPC的值为延时槽指令地址的话，重新执行的指令流为：

  	延时槽指令 -> (延时槽指令地址 + 4)地址的指令 -> (延时槽指令地址 + 8)地址的指令，跳转指令没有得到正确执行

  所以为了在中断返回后正确完成跳转，当异常发生在延迟槽时，EPC的值为``异常指令的地址-4`

  那么回到本题，当我们取出EPC处指令后，判断其opcode既不是100011也不是100001时，我们就应当注意到，此时的AdEL异常发生在了分支指令的延迟槽中，真正应该处理的load指令地址在`EPC+4`

  > 《See Mips Run Linux》中说：When the victim is in a branch delay slot,  the cause register bit Cause(BD) is set. 
  >
  > 所以或许我们还可以通过Cause寄存器的BD位（第31位）是否为1来判断异常发生的位置是否在延迟槽

- Tips 2 或许本题不需要写汇编

	这次的题目暗示我们模仿`lib\genex.S`中`BUILD_HANDLER tlb  do_refill  cli`，使用模板宏新增一个异常处理函数。

	于是我就真的按照祂的指示写汇编代码去了......

	但实际上MOS代码中已经为我们给出了C语言处理异常的软件代码示例，请看`lib\traps.c`中的`page_fault_handler`函数

	```c
	void page_fault_handler(struct Trapframe *tf) {
	    u_int va;
	    u_int *tos, d;
	    struct Trapframe PgTrapFrame;
	    extern struct Env *curenv;
	    // printf("^^^^cp0_BadVAddress:%x\n",tf->cp0_badvaddr);
	
	    bcopy(tf, &PgTrapFrame, sizeof(struct Trapframe));
	    if (tf->regs[29] >= (curenv->env_xstacktop - BY2PG) && tf->regs[29] <= (curenv->env_xstacktop - 1)) {
	        // panic("fork can't nest!!");
	        tf->regs[29] = tf->regs[29] - sizeof(struct Trapframe);
	        bcopy(&PgTrapFrame, tf->regs[29], sizeof(struct Trapframe));
	    } else {
	        tf->regs[29] = curenv->env_xstacktop - sizeof(struct Trapframe);
	        //		printf("page_fault_handler(): bcopy(): src:%x\tdes:%x\n",(int)&PgTrapFrame,(int)(curenv->env_xstacktop - sizeof(struct  Trapframe)));
	
	        bcopy(&PgTrapFrame, curenv->env_xstacktop - sizeof(struct Trapframe), sizeof(struct Trapframe));
	    }
	    //	printf("^^^^cp0_epc:%x\tcurenv->env_pgfault_handler:%x\n",tf->cp0_epc,curenv->env_pgfault_handler);
	
	    tf->cp0_epc = curenv->env_pgfault_handler;
	
	    return;
	}
	```

	这就是典型的好好的C语言放着不用，非要写汇编 %%%%

	

	