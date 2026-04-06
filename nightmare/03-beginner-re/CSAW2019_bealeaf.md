**This is a writeup for CSAW2019 beleaf challenge**

Lets take a look at the main function below
```
undefined8 main(void)

{
  size_t inputLen;
  long transformedInput;
  long in_FS_OFFSET;
  ulong i;
  char input [136];
  long stackCanary;
 
  stackCanary = *(long *)(in_FS_OFFSET + 0x28);
  printf("Enter the flag\n>>> ");
  __isoc99_scanf(&DAT_00100a78,input);
  inputLen = strlen(input);
  if (inputLen < 0x21) {
    puts("Incorrect!");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  i = 0;
  while (i < inputLen) {
    transformedInput = transformFunc(input[i]);
    if (transformedInput != *(long *)(&desiredOutput + i * 8)) {
      puts("Incorrect!");
                    /* WARNING: Subroutine does not return */
      exit(1);
    }
    i = i + 1;
  }
  puts("Correct!");
  if (stackCanary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}
```

The function is taking user input and storing it in our input array. Then we take the length of the input and see if it is 33 bytes (2 x 16 + 1 = 33). This means that our input string needs to be at least 33 bytes long. Next we run into a while loop that will run from 0 -> our input length. Inside this while loop, we call the trannsform function that takes our the index of our input as a parameter. After the function is called, we compare the transformedInput against our desiredOutput. With each iteration we are checking an 8byte offset in our desiredOutput. For example iteration one is an offset of 0 bytes and iteration 2 is an offset of 8 bytes.

Now lets take a look at our transformFunc function
```
long transformFunc(char input)

{
  long i;
 
  i = 0;
  while ((i != -1 && ((int)input != *(int *)(&lookup + i * 4)))) {
    if ((int)input < *(int *)(&lookup + i * 4)) {
      i = i * 2 + 1;
    }
    else {
      if (*(int *)(&lookup + i * 4) < (int)input) {
        i = (i + 1) * 2;
      }
    }
  }
  return i;
}
```
This function essentially takes a character and looks up the index in our lookup array. 

So for iteration i=0, we are looking at an offset of 0 bytes in lookup, iteration i = 1 is an offset of 4 bytes.

Here is a small segment of the array so that we can get a general idea of what is going on
```  
        00301020 77              ??         77h    w
        00301021 00              ??         00h
        00301022 00              ??         00h
        00301023 00              ??         00h
        00301024 66              ??         66h    f
        00301025 00              ??         00h
        00301026 00              ??         00h
        00301027 00              ??         00h
        00301028 7b              ??         7Bh    {
        00301029 00              ??         00h
        0030102a 00              ??         00h
        0030102b 00              ??         00h
        0030102c 5f              ??         5Fh    _
        0030102d 00              ??         00h
        0030102e 00              ??         00h
        0030102f 00              ??         00h
        00301030 6e              ??         6Eh    n
        00301031 00              ??         00h
        00301032 00              ??         00h
        00301033 00              ??         00h
        00301034 79              ??         79h    y
        00301035 00              ??         00h
        00301036 00              ??         00h
        00301037 00              ??         00h
        00301038 7d              ??         7Dh    }
        00301039 00              ??         00h
        0030103a 00              ??         00h
        0030103b 00              ??         00h
        0030103c ff              ??         FFh
        0030103d ff              ??         FFh
        0030103e ff              ??         FFh
        0030103f ff              ??         FFh
        00301040 62              ??         62h    b
        00301041 00              ??         00h
        00301042 00              ??         00h
        00301043 00              ??         00h
        00301044 6c              ??         6Ch    l
        00301045 00              ??         00h
        00301046 00              ??         00h
        00301047 00              ??         00h
        00301048 72              ??         72h    r
        00301049 00              ??         00h
        0030104a 00              ??         00h
        0030104b 00              ??         00h
        0030104c ff              ??         FFh
        0030104d ff              ??         FFh
        0030104e ff              ??         FFh
        0030104f ff              ??         FFh
        00301050 ff              ??         FFh
        00301051 ff              ??         FFh
        00301052 ff              ??         FFh
        00301053 ff              ??         FFh
        00301054 ff              ??         FFh
        00301055 ff              ??         FFh
        00301056 ff              ??         FFh
        00301057 ff              ??         FFh
        00301058 ff              ??         FFh
        00301059 ff              ??         FFh
        0030105a ff              ??         FFh
        0030105b ff              ??         FFh
        0030105c ff              ??         FFh
        0030105d ff              ??         FFh
        0030105e ff              ??         FFh
        0030105f ff              ??         FFh
        00301060 ff              ??         FFh
        00301061 ff              ??         FFh
        00301062 ff              ??         FFh
        00301063 ff              ??         FFh
        00301064 61              ??         61h    a
        00301065 00              ??         00h
        00301066 00              ??         00h
        00301067 00              ??         00h
        00301068 65              ??         65h    e
        00301069 00              ??         00h
        0030106a 00              ??         00h
        0030106b 00              ??         00h
        0030106c 69              ??         69h    i
```

Now in order to use this array we need to look at the desiredOutput array that exists in the main funtion. that desiredOutput will help us determine what letters in the lookup array we need to use and in what order.

Let's take a look at the desiredOutput array below
```
        003014e0 01              ??         01h
        003014e1 00              ??         00h
        003014e2 00              ??         00h
        003014e3 00              ??         00h
        003014e4 00              ??         00h
        003014e5 00              ??         00h
        003014e6 00              ??         00h
        003014e7 00              ??         00h
        003014e8 09              ??         09h
        003014e9 00              ??         00h
        003014ea 00              ??         00h
        003014eb 00              ??         00h
        003014ec 00              ??         00h
        003014ed 00              ??         00h
        003014ee 00              ??         00h
        003014ef 00              ??         00h
        003014f0 11              ??         11h
        003014f1 00              ??         00h
        003014f2 00              ??         00h
        003014f3 00              ??         00h
        003014f4 00              ??         00h
        003014f5 00              ??         00h
        003014f6 00              ??         00h
        003014f7 00              ??         00h
        003014f8 27              ??         27h    '
        003014f9 00              ??         00h
        003014fa 00              ??         00h
        003014fb 00              ??         00h
        003014fc 00              ??         00h
        003014fd 00              ??         00h
        003014fe 00              ??         00h
        003014ff 00              ??         00h
        00301500 02              ??         02h
```

If we look in the last column we can see that we are given numbers, each with an offset of 8 bytes between each other. If we take all of the integers in this array and use them in our transformFunc, we will get the desiredOutput.

For example the first 3 numbers in the desiredOutput array is 1, 9, and 11. Let's see what plugging them in does in our transformFunc.

```
&lookup + (1 * 4) = 00301020
```
Which if we look that up in our lookup array, we see we get the character "f"

Now lets try with 9
```
&lookup + (9 * 4) = 0x301044
```
Which translates to the character "l" in our lookup array

Now lastly lets try with 11 (Remember this is hex so we need to convert 1 * 16 + 1 = 17)
```
&lookup + (17 * 4) = 0x00301064
```
Which translates to the character a

If we keep through the array we find that our final string is 
```
flag{we_beleaf_in_your_re_future}
```