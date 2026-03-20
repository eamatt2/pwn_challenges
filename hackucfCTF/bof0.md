**This is a writeup for the bof0 challenge**

```
int main(void) {
	int admin = 0;
	char buf[32];
	
	scanf("%s", buf);
	
	if(admin) {
		win();
	}
	else {
		puts("nope!");
	}
	
	return 0;
}
```

Above is the main function that our program will be running. We can see that we have initialize and admin variable to 0 and a char buffer of size 32.

Our attack point is the scanf that is taking a string input that does not have proper input handling. We will be able to write past the allocated space for buf and into the space for admin.

If we load the binary into ghidra, we can look at the assembly that makes up our program,

We can see that we have

```
SUB RSP,0x30
```

What this tells us is that our function reserves 48 bytes of stack space (3 x 16 = 48)

Now the next important line is

```
MOV dword ptr [RBP + admin],0x0
```

What this tells us is that we are moving the value of 0 into the value at RBP - 0x4. This is the space of our admin variable

Now we can see that we are loading the 

Now we look at where our buf is located on the stack
```
LEA RAX=>buf,[RBP + -0x30]
```

What this instruction tells us is that buf is loaded at the address space of RBP -0x30, so that is where it begins in memory.

Now in order to figure out our payload that allows us to get the flag we need to find the 

0x30 can be converted to 48 bytes by doing 3 x 16 + 0
0x4 can be converted to 4 bytes

Now we need to find the offset by subtracting 4 from 48:
48 - 4 = 44

This means that for us to overwrite into admin our payload needs to be at greater than 44 bytes since if we make it exactly 44 bytes then that would just fill up the entirety of the buf array.

So if we do our input as
```
111111111111111111111111111111111111111111111
```

We will successfully overwrite admin to equal 1 which gives us our flag