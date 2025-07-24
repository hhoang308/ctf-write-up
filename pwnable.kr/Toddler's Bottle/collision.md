# CTF Challenge: `collision`

## Challenge Description

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
                res += ip[i];
        }
        return res;
}

int main(int argc, char* argv[]){
        if(argc<2){
                printf("usage : %s [passcode]\n", argv[0]);
                return 0;
        }
        if(strlen(argv[1]) != 20){
                printf("passcode length should be 20 bytes\n");
                return 0;
        }

        if(hashcode == check_password( argv[1] )){
                setregid(getegid(), getegid());
                system("/bin/cat flag");
                return 0;
        }
        else
                printf("wrong passcode.\n");
        return 0;
}
```

---

## Objective

The goal is to find a 20-byte `passcode` such that the program will print the flag. The correct passcode must:

- Be exactly 20 bytes long
- When interpreted as five 4-byte integers, their sum must equal the `hashcode` value: `0x21DD09EC`

---

## Source Code Behavior

1. The `main` function checks if a passcode is provided via `argv[1]`.
2. It verifies that the length of `argv[1]` is exactly 20 bytes.
3. It casts the `char*` input to an `int*`, then sums five consecutive integers using:
   ```c
   int* ip = (int*)p;
   for(i = 0; i < 5; i++) res += ip[i];
   ```
4. If the total matches `0x21DD09EC`, the program prints the flag.

---

## Exploitation

1. Read the code and identify the condition that must be satisfied: the sum of 5 integers (each from 4 bytes of input) must be `0x21DD09EC`.
2. Understand pointer casting from `char*` to `int*`: each `int` is 4 bytes (32 bits), so 20 bytes of input will be interpreted as 5 `int` values.
3. Convert `0x21DD09EC` to decimal:
   ```
   0x21DD09EC = 568134124
   ```
4. Divide the target into 5 integers:
   ```
   568134124 = 4 * 113626824 + 113626828
   ```
5. Convert these integers to 4-byte little-endian sequences. Since typing arbitrary bytes from the keyboard is not feasible, we use `echo -n -e` to build the byte string:
   - `-n`: suppress newline
   - `-e`: enable interpretation of escape sequences like `\xNN`
6. Be aware that C uses little-endian format on most platforms, so the bytes must be in reverse order.

7. Final exploit command:

   ```bash
   ./col `echo -n -e "\xc8\xce\xc5\x06\xc8\xce\xc5\x06\xc8\xce\xc5\x06\xc8\xce\xc5\x06\xcc\xce\xc5\x06"`
   ```

8. Output:
   ```
   Two_hash_collision_Nicely
   ```

---
