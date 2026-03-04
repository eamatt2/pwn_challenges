**This is a writeup for the collision challenge**

```
 if(argc<2){
                printf("usage : %s [passcode]\n", argv[0]);
                return 0;
        }
```

This code segment is looking at the arguments that are passed in when the program is run. We need to pass in our password string.

```
 if(strlen(argv[1]) != 20){
                printf("passcode length should be 20 bytes\n");
                return 0;
        }
```

This code segment is telling us that our password string needs to be exactly 20 bytes long.

```
  if(hashcode == check_password( argv[1] )){
                setregid(getegid(), getegid());
                system("/bin/cat flag");
                return 0;
        }
```
If we look at this code segment. We are comparing our haschode value of 0x21DD09EC to the passcode that we input. We can also see that it is calling the function "check_password".

```
unsigned long check_password(const char* p){
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
                res += ip[i];
        }
        return res;
}
```

In this function, we are 