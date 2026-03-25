**This is a writeup for the Helithumper RE challenge**

Lets take a look at the main function below
```
bool main(void) {
    int stringCorrect;
    void *userInput;

    userInput = calloc(0x32,1);
    puts(Welcome to the Salty Spitoon, How touch are ya?);
    scanf("%s", userInput);
    stringCorrect = validate(userInput);
    if(stringCorrect == 0) {
        puts("Yeah right. Back to Weenie Hut Jr with ya");
    }
    else {
        put("Right this way...");
    }
    return stringCorrect == 0;
}
```

From this function we can see that we are allocating 50 bytes (16 x 3 + 2 = 50) of space for our user input, and then scanning the user input and storing it. After that we are running the validate function to see if uesrInput matches string that the program wants. If the string matches doesn't match we lose the challenge.

Now lets take a look at the validate function since that will hold the logic for how to win the challenge.
```
undefined8 validate(char *input)

{
  long lVar1;
  size_t inputLen;
  undefined8 returnValue;
  long in_FS_OFFSET;
  int i;
  int checkValues [4];
 
  lVar1 = *(long *)(in_FS_OFFSET + 0x28);
  checkValues[0] = 0x66;
  checkValues[1] = 0x6c;
  checkValues[2] = 0x61;
  checkValues[3] = 0x67;
  checkValues[4] = 0x7b;
  checkValues[5] = 0x48;
  checkValues[6] = 0x75;
  checkValues[7] = 0x43;
  checkValues[8] = 0x66;
  checkValues[9] = 0x5f;
  checkValues[10] = 0x6c;
  checkValues[0xb] = 0x41;
  checkValues[0xc] = 0x62;
  checkValues[0xd] = 0x7d;
  inputLen = strlen(input);
  i = 0;
  do {
    if ((int)inputLen <= i) {
      returnValue = 1;
LAB_001012b7:
      if (lVar1 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
        __stack_chk_fail();
      }
      return returnValue;
    }
    if ((int)input[(long)i] != checkValues[(long)i]) {
      returnValue = 0;
      goto LAB_001012b7;
    }
    i = i + 1;
  } while( true );
}
```
It seems that the main job for this function is to compare each character in our input against the checkValues array which stores the string that will get the flag.

If we translate the hex value that are stored in the array, we can see that our program is checking for the string
```
0x66 0x6c 0x61 0x67 0x7b 0x48 0x75 0x43 0x66 0x5f 0x6c 0x41 0x62 0x7d
flag{HuCf_lAb}
```

So if we run the program and use the above string as our input, we will get the output "Right this way..." which means we beat the challenge
