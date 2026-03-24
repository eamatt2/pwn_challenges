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

Our main func initializes a boolean variable of "didPurchase" to false. We initialize a characrter array with 50 bytes of allocated space.

