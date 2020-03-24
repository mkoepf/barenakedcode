# Bare naked code - binary, at last

minim00.c
```c
void main() {}
```

```bash
> gdb -batch -ex 'file minim00' -ex 'disassemble main'
Dump of assembler code for function main:
   0x00000000000005fa <+0>:	push   %rbp
   0x00000000000005fb <+1>:	mov    %rsp,%rbp
   0x00000000000005fe <+4>:	nop
   0x00000000000005ff <+5>:	pop    %rbp
   0x0000000000000600 <+6>:	retq   
End of assembler dump.
```

minim0.c
```c
int main() {}
```

```bash
> gdb -batch -ex 'file minim0' -ex 'disassemble main'
Dump of assembler code for function main:
   0x00000000000005fa <+0>:	push   %rbp
   0x00000000000005fb <+1>:	mov    %rsp,%rbp
   0x00000000000005fe <+4>:	mov    $0x0,%eax
   0x0000000000000603 <+9>:	pop    %rbp
   0x0000000000000604 <+10>:	retq   
End of assembler dump.
```

minim1.c
```c
int main() {
	return 0;
}
```

```bash
> gdb -batch -ex 'file minim1' -ex 'disassemble main'
Dump of assembler code for function main:
   0x00000000000005fa <+0>:	push   %rbp
   0x00000000000005fb <+1>:	mov    %rsp,%rbp
   0x00000000000005fe <+4>:	mov    $0x0,%eax
   0x0000000000000603 <+9>:	pop    %rbp
   0x0000000000000604 <+10>:	retq   
End of assembler dump.
```

The same!


minim2.c
```c
int main() {
	return 42;
}
```

```bash
> gdb -batch -ex 'file minim2' -ex 'disassemble main'
Dump of assembler code for function main:
   0x00000000000005fa <+0>:	push   %rbp
   0x00000000000005fb <+1>:	mov    %rsp,%rbp
   0x00000000000005fe <+4>:	mov    $0x2a,%eax
   0x0000000000000603 <+9>:	pop    %rbp
   0x0000000000000604 <+10>:	retq   
End of assembler dump.
```

minim3.c
```c
int main() {
	return -42;
}
```

```bash
> gdb -batch -ex 'file minim3' -ex 'disassemble main'
Dump of assembler code for function main:
   0x00000000000005fa <+0>:	push   %rbp
   0x00000000000005fb <+1>:	mov    %rsp,%rbp
   0x00000000000005fe <+4>:	mov    $0xffffffd6,%eax
   0x0000000000000603 <+9>:	pop    %rbp
   0x0000000000000604 <+10>:	retq   
End of assembler dump.
```

minim4.c
```c
int main() {
	int a = 42;
	return 0;
}
```

```bash
> gdb -batch -ex 'file minim4' -ex 'disassemble main'
Dump of assembler code for function main:
   0x00000000000005fa <+0>:	push   %rbp
   0x00000000000005fb <+1>:	mov    %rsp,%rbp
   0x00000000000005fe <+4>:	movl   $0x2a,-0x4(%rbp)
   0x0000000000000605 <+11>:	mov    $0x0,%eax
   0x000000000000060a <+16>:	pop    %rbp
   0x000000000000060b <+17>:	retq   
End of assembler dump.
```

minim5.c
```c
int main() {
	int a = 42;
	return 0;
}
```

```bash
> gdb -batch -ex 'file minim5' -ex 'disassemble main'
Dump of assembler code for function main:
   0x00000000000005fa <+0>:	push   %rbp
   0x00000000000005fb <+1>:	mov    %rsp,%rbp
   0x00000000000005fe <+4>:	movl   $0x2a,-0x4(%rbp)
   0x0000000000000605 <+11>:	mov    -0x4(%rbp),%eax
   0x0000000000000608 <+14>:	pop    %rbp
   0x0000000000000609 <+15>:	retq   
End of assembler dump.
```

Hardly optimal, since a is only used once, in the return statement. Can the compiler figure that out?

```bash
> gcc -O1 minim5.c -o minim5opt
> gdb -batch -ex 'file minim5opt' -ex 'disassemble main'
Dump of assembler code for function main:
   0x00000000000005fa <+0>:	mov    $0x2a,%eax
   0x00000000000005ff <+5>:	retq   
End of assembler dump.
```

Better!
