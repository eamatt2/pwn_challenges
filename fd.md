In the fd challenge, we are learnign about file descriptors in order to obtain the flag.

What are file descriptors? File descriptors are non-negative integers that interact with I/O. 

Common file descriptors include 0 which is stdin, 1 which is stdout, and 2 which is stderr.

```
if(argc<2){
                printf("pass argv[1] a number\n");
                return 0;
        }
```

This code segment is looking at the arguments that are passed in when the program is run. We need to pass in at msot one integer. We cannot pass in anything other than an integer.

```
int fd = atoi( argv[1] ) - 0x1234;
        int len = 0;
        len = read(fd, buf, 32);
        if(!strcmp("LETMEWIN\n", buf)){
                printf("good job :)\n");
                setregid(getegid(), getegid());
                system("/bin/cat flag");
                exit(0);
        }
```

atoi takes a string of numbers (EX: 32, 433, 560, etc.) and either converts the value into an int or returns 0 on error. Then after that we are subtracting 0x1234 from the value we get from atoi. We need to convert the hexadecimal 0x1234 to an int.

We  can do (1x16^1) + (2x16^2) + (3x16^3) + (4x16^4) = 4660

Now the hint to what our argv[1] value should be lies within the line
```
len = read(fd, buf, 32);
```

The read function takes in three parameters. A file descriptor, the buffer, and a count. We already know the buf is buf and our count is 32 which is established in earlier in our code. All we need left is a file descriptor value. 

Lets choose 0 as our file descriptor value. So when we run the fd command, we should be running it like so
```
./fd 4660
```
that was when we do atoi(4660) - 4660 we get 0 as our value amd so that means we are using the file descriptor 0 in our read function call.

Since 0 means stdin, we must type what our stdin value is going to be. This value is going to be passed into the buf because of the read function.

So now lets look at our if statement so we can determine what our stdin value needs to be.
```
 if(!strcmp("LETMEWIN\n", buf))
 ```

If we look at the man page, strcmp we can see that we are comparing the strings **LETMEWIN** and the value in **buf**.
strmp will return 0 if the strings are equal, a negative value if s1 < s2, and a positive if s1 > s2.

If we want the if statement to pass, then strcmp needs to equal 0 meaning that our stdin value that we are passing to the buf needs to be **LETMEWIN**.

Once we use **LETMEWIN** as the value, the rest if the code executes and we are able to get the flag 
```
Mama! Now_I_understand_what_file_descriptors_are!
```