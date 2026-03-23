```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
        printf("Welcome to pwnable.kr\n");
        printf("Let's see if you know how to give input to program\n");
        printf("Just give me correct inputs then you will get the flag :)\n");

        // argv
        if(argc != 100) return 0;
        if(strcmp(argv['A'],"\x00")) return 0;
        if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
        printf("Stage 1 clear!\n");

        // stdio
        char buf[4];
        read(0, buf, 4);
        if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
        read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
        printf("Stage 2 clear!\n");

        // env
        if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
        printf("Stage 3 clear!\n");

        // file
        FILE* fp = fopen("\x0a", "r");
        if(!fp) return 0;
        if( fread(buf, 4, 1, fp)!=1 ) return 0;
        if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
        fclose(fp);
        printf("Stage 4 clear!\n");

        // network
        int sd, cd;
        struct sockaddr_in saddr, caddr;
        sd = socket(AF_INET, SOCK_STREAM, 0);
        if(sd == -1){
                printf("socket error, tell admin\n");
                return 0;
        }
        saddr.sin_family = AF_INET;
        saddr.sin_addr.s_addr = INADDR_ANY;
        saddr.sin_port = htons( atoi(argv['C']) );
        if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
                printf("bind error, use another port\n");
                return 1;
        }
        listen(sd, 1);
        int c = sizeof(struct sockaddr_in);
        cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
        if(cd < 0){
                printf("accept error, tell admin\n");
                return 0;
        }
        if( recv(cd, buf, 4, 0) != 4 ) return 0;
        if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
        printf("Stage 5 clear!\n");

        // here's your flag
        setregid(getegid(), getegid());
        system("/bin/cat flag");
        return 0;
}
```

`python3 -c 'with open("\x0a", "wb") as f: f.write(b"\x00\x00\x00\x00")'`
```
input2@ubuntu:/tmp/hoangnh$ ls
''$'\n'
input2@ubuntu:/tmp/hoangnh$ ln -s /home/input2/flag flag
input2@ubuntu:/tmp/hoangnh$ (python3 -c 'import os, subprocess; \
argv = ["A"]*100; \
argv[0] = "/home/input2/input2"; \
argv[65] = ""; \
argv[66] = "\x20\x0a\x0d"; \
argv[67] = "5555"; \
env = {b"\xde\xad\xbe\xef": b"\xca\xfe\xba\xbe"}; \
r1, w1 = os.pipe(); r2, w2 = os.pipe(); \
p = subprocess.Popen(argv, stdin=r1, stderr=r2, env=env); \
os.write(w1, b"\x00\x0a\x00\xff"); \
os.write(w2, b"\x00\x0a\x02\xff"); \
p.wait()' &); sleep 2; python3 -c 'import socket; s = socket.socket(); s.connect(("127.0.0.1", 5555)); s.send(b"\xde\xad\xbe\xef")'
```
