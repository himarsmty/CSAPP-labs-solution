# CSAPP-bomblab
This is a note for the solution of bomblab, our task is to disassemble the program **bomb** in **bomblab-handout** and solve six puzzles. Attention that the source file **bomb.c** makes none sense.

The Recommended way is to use **GDB**. Here is a [GDB commands list](http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf). Also you can use the tool **objdump** to get the full assemble code by using this command. 
```
objdump -d bomb
```
As the program uses some library functions like *sscanf*, use command **man** to learn the usage, or here is a [C function tutorial](https://www.tutorialspoint.com/c_standard_library).



The file **defused** is the answer I defused. Some answers are not unique.

Here is what I get during soving the puzzles

```
(gdb) r defused 
Starting program: /home/tianyun/CSAPP/labs/bomblab/bomb defused
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Phase 1 defused. How about the next one?
That's number 2.  Keep going!
Halfway there!
So you got that one.  Try this one.
Good work!  On to the next...
Congratulations! You've defused the bomb!
[Inferior 1 (process 15459) exited normally]
```

Pretty fun! Gook luck and be careful not to trigger the bomb :)