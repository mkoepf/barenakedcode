# Bare naked code - binary, at last

What's the simplest program I can come up with? How about this one:

minim00.c
```c
void main() {}
```

Let's compile and see what _gcc_ makes of it. 

```bash
> gcc minim00.c -o minim00
```

What does the resulting binary look like? With _gdb_ we disassemble the binary to see the program's low level 
operations.

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

This looks like an awful lot of code to realize the empty function '{}'. In fact, though, the actual function _body_ is 
translated to 
 
```bash
0x00000000000005fe <+4>:	nop
```

which tells the computer to do nothing, [as we'd expect for an empty set of instructions](https://en.wikipedia.org/wiki/NOP_(code)#C_and_derivatives).
What's the point of the remaining lines?

```bash
push   %rbp
mov    %rsp,%rbp
...
[function body]
...
pop    %rbp
retq   
```

They nicely show the anatomy of a simple function call. First,

```bash
push   %rbp
```

puts the base pointer's current value onto the stack. This is to remember this value because the next statement 

```bash
mov    %rsp,%rbp
```

overwrites %rbp with the _stack_ pointer's current value. In the end, when the function body has been executed, 

```bash
pop    %rbp
```

puts the top of the stack (base pointer's old value) back into %rbp. You could say that the base pointer moved back to 
its original position. 

Finally, 

```bash
retq
```

is the x86 shorthand for _pop %rip_, i.e. %rip now points to the address stored in the top of the stack.

Interlude: It turns out, whole %rbp gymnastics is purely optional and the compiler will actually omit it when you run it 
with optimization flags. If you rebuild the binary by

```bash
> gcc -fomit-frame-pointer minim00.c -o minim00nofp
```

and disassemble minim00nofp, you see that main in the new binary has been condensed to a neat 2-operation program:

```bash
0x00000000000005fa <+0>:	nop
0x00000000000005fb <+1>:	retq  
```

From now on though, we'll always use _gcc_ without any fancy options, i.e. 

```bash
> gcc prog.c -o prog
```

and afterwards disassemble with 

```bash
> gdb -batch -ex 'file prog' -ex 'disassemble main'
```

until explicitely mentioned otherwise.

How will the assembly code change when the _void_ return type is changed to _int_? 

minim0.c
```c
int main() {}
```

The assembly code of the main function then changes to

```bash
0x00000000000005fa <+0>:	push   %rbp
0x00000000000005fb <+1>:	mov    %rsp,%rbp
0x00000000000005fe <+4>:	mov    $0x0,%eax
0x0000000000000603 <+9>:	pop    %rbp
0x0000000000000604 <+10>:	retq   
```

Ok, looks like the _nop_ on position <+4> has been replaced by the expression

```bash
mov    $0x0,%eax
```

which writes 0 into the _eax_ register. Why zero? Since we defined main's return type as int but didn't provide a return 
statement, gcc automatically infers an implicit return 0_ by default. This can be easily confirmed by comparing to

minim1.c
```c
int main() {
	return 0;
}
```

which disassembles to the very same assembly:

```bash
0x00000000000005fa <+0>:	push   %rbp
0x00000000000005fb <+1>:	mov    %rsp,%rbp
0x00000000000005fe <+4>:	mov    $0x0,%eax
0x0000000000000603 <+9>:	pop    %rbp
0x0000000000000604 <+10>:	retq   
```

From this we see that eax is used to store the return value. Why zero _dollar_? According to 
[Yale's x86 Assembly Guide](https://www.cs.yale.edu/flint/cs421/papers/x86-asm/asm.html#instructions), the '$' indicates 
constants. 

Only the expression at position +4 changes, when we change the return type, e.g.

minim2.c
```c
int main() {
	return 42;
}
```

yields

```bash
...
0x00000000000005fe <+4>:	mov    $0x2a,%eax
...
```

and 

minim3.c
```c
int main() {
	return -42;
}
```

yields

```bash
...
0x00000000000005fe <+4>:	mov    $0xffffffd6,%eax
...
```

So far, so good.

How about variables? Let's start with a simple integer variable. The program

minim4.c
```c
int main() {
	int a = 42;
	return 0;
}
```

disassembles to

```bash
0x00000000000005fa <+0>:	push   %rbp
0x00000000000005fb <+1>:	mov    %rsp,%rbp
0x00000000000005fe <+4>:	movl   $0x2a,-0x4(%rbp)
0x0000000000000605 <+11>:	mov    $0x0,%eax
0x000000000000060a <+16>:	pop    %rbp
0x000000000000060b <+17>:	retq   
```

The new line here is

```bash
movl   $0x2a,-0x4(%rbp)
```

which stores the constant 0x2a (=42) in the stack at a position 4 bytes below the base pointer rbp. Looks like we found
out where the compiler is storing local data!

Interlude: Didn't we say the base pointer was optional and can be omitted without problems? How does that work in this 
case? Well, let's see. Run

```bash
> gcc -fomit-frame-pointer  minim4.c -o minim4nofp
```

and disassemble the result. You'll obtain

```bash
0x00000000000005fa <+0>:	movl   $0x2a,-0x4(%rsp)
0x0000000000000602 <+8>:	mov    $0x0,%eax
0x0000000000000607 <+13>:	retq
```

Aha. The local variable's value is simply stored relative to the _stack_ pointer instead of the base pointer.

So far, we've seen how the return value is handled and we've seen how a single local variable is handled. Let's go crazy
and combine the two:

minim5.c
```c
int main() {
	int a = 42;
	return a;
}
```

This disassembles to

```bash
0x00000000000005fa <+0>:	push   %rbp
0x00000000000005fb <+1>:	mov    %rsp,%rbp
0x00000000000005fe <+4>:	movl   $0x2a,-0x4(%rbp)
0x0000000000000605 <+11>:	mov    -0x4(%rbp),%eax
0x0000000000000608 <+14>:	pop    %rbp
0x0000000000000609 <+15>:	retq   
```

and - no big surprise! - we see that the local variable's value is copied to eax, the register used for the return 
value. But wait! This is hardly optimal, since _a_ is only used once, in the return statement, so why store it at 
all? Can the compiler figure that out?

Let's try the gcc optimization flag -O1.

```bash
> gcc -O1 minim5.c -o minim5opt
```

yielding, after disassembly,

```bash
0x00000000000005fa <+0>:	mov    $0x2a,%eax
0x00000000000005ff <+5>:	retq   
```

That's better! 

Now let's see how more than one local variable is stored.

minim6.c
```c
int main() {
	int a = 42;
	int b = 13;
	return a + b;
}
```

yields

```bash
0x00000000000005fa <+0>:	push   %rbp
0x00000000000005fb <+1>:	mov    %rsp,%rbp
0x00000000000005fe <+4>:	movl   $0x2a,-0x8(%rbp)
0x0000000000000605 <+11>:	movl   $0xd,-0x4(%rbp)
0x000000000000060c <+18>:	mov    -0x8(%rbp),%edx
0x000000000000060f <+21>:	mov    -0x4(%rbp),%eax
0x0000000000000612 <+24>:	add    %edx,%eax
0x0000000000000614 <+26>:	pop    %rbp
0x0000000000000615 <+27>:	retq 
```

The two integers are stacked on top of each other into 4 byte (Long) "compartments". Note, that the variable b which is
initialized second comes first on the stack. 
Their addition is done by first loading them into registers edx, eax and finally adding them in a way that the result
is already in eax, as the return value.

Of course, this is again suboptimal. In fact, everything about minim6 is already known at compile time, because only
constants are added. The compiler should understand that, as well. And indeed, when we compile with 
optimization using

```bash
gcc -O1 minim6.c -o minim6opt
```

disassembly yields

```bash
0x00000000000005fa <+0>:	mov    $0x37,%eax
0x00000000000005ff <+5>:	retq   
gcc 
```

i.e. the compiler simply computes the result 42+13=55(=0x37) at compile time. At runtime, the result is simply placed 
into the proper register and that's it.

## Data types

float.c
```c
void main() {
	float a = 3.14;
	return;
}
```

```bash
0x00000000000005fa <+0>:	push   %rbp
0x00000000000005fb <+1>:	mov    %rsp,%rbp
0x00000000000005fe <+4>:	movss  0x8e(%rip),%xmm0        # 0x694
0x0000000000000606 <+12>:	movss  %xmm0,-0x4(%rbp)
0x000000000000060b <+17>:	nop
0x000000000000060c <+18>:	pop    %rbp
0x000000000000060d <+19>:	retq   
```

Compiled with -O1 this becomes a one-liner:

```bash
0x00000000000005fa <+0>:	repz retq 
```

intarray.c
```c
void main() {
    int ia[3] = {1, 2, 3};
    return;
}
```

```bash
0x000000000000066a <+0>:	push   %rbp
0x000000000000066b <+1>:	mov    %rsp,%rbp
0x000000000000066e <+4>:	sub    $0x20,%rsp
0x0000000000000672 <+8>:	mov    %fs:0x28,%rax
0x000000000000067b <+17>:	mov    %rax,-0x8(%rbp)
0x000000000000067f <+21>:	xor    %eax,%eax
0x0000000000000681 <+23>:	movl   $0x1,-0x14(%rbp)
0x0000000000000688 <+30>:	movl   $0x2,-0x10(%rbp)
0x000000000000068f <+37>:	movl   $0x3,-0xc(%rbp)
0x0000000000000696 <+44>:	nop
0x0000000000000697 <+45>:	mov    -0x8(%rbp),%rax
0x000000000000069b <+49>:	xor    %fs:0x28,%rax
0x00000000000006a4 <+58>:	je     0x6ab <main+65>
0x00000000000006a6 <+60>:	callq  0x540 <__stack_chk_fail@plt>
0x00000000000006ab <+65>:	leaveq 
0x00000000000006ac <+66>:	retq   
```
