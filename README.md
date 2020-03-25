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
