https://elixir.bootlin.com/linux/latest/source/Documentation/networking/filter.txt  
```text

   ja               6                    Jump to label
 
  BPF_JUMP(BPF_JMP + BPF_JA, 0, 0, 0),
```
像上面那样插入一个jmp 跳转到到下一行代码 ，最后应该会被jit生成器给删掉，也就是空指令的效果吧，不过和nop指令还是有区别的。

https://elixir.bootlin.com/linux/latest/source/arch/x86/net/bpf_jit_comp.c
```c
		case BPF_JMP | BPF_JA:
			if (insn->off == -1)
				/* -1 jmp instructions will always jump
				 * backwards two bytes. Explicitly handling
				 * this case avoids wasting too many passes
				 * when there are long sequences of replaced
				 * dead code.
				 */
				jmp_offset = -2;
			else
				jmp_offset = addrs[i + insn->off] - addrs[i];

			if (!jmp_offset)
				/* Optimize out nop jumps */
				break;
emit_jmp:
			if (is_imm8(jmp_offset)) {
				EMIT2(0xEB, jmp_offset);
			} else if (is_simm32(jmp_offset)) {
				EMIT1_off32(0xE9, jmp_offset);
			} else {
				pr_err("jmp gen bug %llx\n", jmp_offset);
				return -EFAULT;
			}
			break;
```
