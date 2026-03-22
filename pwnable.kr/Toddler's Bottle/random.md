```
#include <stdio.h>

int main(){
        unsigned int random;
        random = rand();        // random value!

        unsigned int key=0;
        scanf("%d", &key);

        if( (key ^ random) == 0xcafebabe ){
                printf("Good!\n");
                setregid(getegid(), getegid());
                system("/bin/cat flag");
                return 0;
        }

        printf("Wrong, maybe you should try 2^32 cases.\n");
        return 0;
}
```

pseudo-random number generator (PRNG) 

hàm rand() để tạo được số ngẫu nhiên sau mỗi lần chạy thì phải chạy srand(seed) trước đó, nếu không gọi thì hệ thống sẽ mặc định là đã gọi srand(1) và hàm rand() sẽ luôn trả về cùng một giá trị trong một môi trường

khi mở gdb thì chương trình chưa thực sự nạp vào RAM -> phải ấn run hoặc start (nên là start, lệnh này tự động đặt 1 breakpoint vào main)

file thực thi hiện đại được biên dịch dưới dạng PIE (position independent executable), đồng thời, để bảo mật thì hệ điều hành cũng có cơ chế không nạp chương trình vào một địa chỉ cố định mà chọn một địa chỉ ngẫu nhiên trong RAM làm gốc rồi mới nạp file vào đó

```
pwndbg> start
Temporary breakpoint 2 at 0x1211
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Temporary breakpoint 1, 0x000055a15a4b9211 in main ()
Disabling the emulation via Unicorn Engine that is used for computing branches as there isn't enough memory (1GB) to use it (since mmap(1G, RWX) failed). See also:
* https://github.com/pwndbg/pwndbg/issues/1534
* https://github.com/unicorn-engine/unicorn/pull/1743
Either free your memory or explicitly set `set emulate off` in your Pwndbg config
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]─────────────────────────────────
 RAX  0x55a15a4b9209 (main) ◂— endbr64
 RBX  0
 RCX  0x55a15a4bbd90 (__do_global_dtors_aux_fini_array_entry) —▸ 0x55a15a4b91c0 (__do_global_dtors_aux) ◂— endbr64
 RDX  0x7fff4aff19a8 —▸ 0x7fff4aff2d89 ◂— 'SHELL=/bin/bash'
 RDI  1
 RSI  0x7fff4aff1998 —▸ 0x7fff4aff2d75 ◂— '/home/random/random'
 R8   0x7f0b16a41f10 (initial+16) ◂— 4
 R9   0x7f0b16a63040 (_dl_fini) ◂— endbr64
 R10  0x7f0b16a5d908 ◂— 0xd00120000000e
 R11  0x7f0b16a78660 (_dl_audit_preinit) ◂— endbr64
 R12  0x7fff4aff1998 —▸ 0x7fff4aff2d75 ◂— '/home/random/random'
 R13  0x55a15a4b9209 (main) ◂— endbr64
 R14  0x55a15a4bbd90 (__do_global_dtors_aux_fini_array_entry) —▸ 0x55a15a4b91c0 (__do_global_dtors_aux) ◂— endbr64
 R15  0x7f0b16a97040 (_rtld_global) —▸ 0x7f0b16a982e0 —▸ 0x55a15a4b8000 ◂— 0x10102464c457f
 RBP  0x7fff4aff1880 ◂— 1
 RSP  0x7fff4aff1880 ◂— 1
 RIP  0x55a15a4b9211 (main+8) ◂— push rbx
─────────────────────────────────────────[ DISASM / x86-64 / set emulate off ]─────────────────────────────────────────
 ► 0x55a15a4b9211 <main+8>     push   rbx
   0x55a15a4b9212 <main+9>     sub    rsp, 0x18
   0x55a15a4b9216 <main+13>    mov    rax, qword ptr fs:[0x28]        RAX, [0x7f0b16823768]
   0x55a15a4b921f <main+22>    mov    qword ptr [rbp - 0x18], rax
   0x55a15a4b9223 <main+26>    xor    eax, eax                        EAX => 0
   0x55a15a4b9225 <main+28>    mov    eax, 0                          EAX => 0
   0x55a15a4b922a <main+33>    call   rand@plt                    <rand@plt>

   0x55a15a4b922f <main+38>    mov    dword ptr [rbp - 0x1c], eax
   0x55a15a4b9232 <main+41>    mov    dword ptr [rbp - 0x20], 0
   0x55a15a4b9239 <main+48>    lea    rax, [rbp - 0x20]
   0x55a15a4b923d <main+52>    mov    rsi, rax
───────────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────────
00:0000│ rbp rsp 0x7fff4aff1880 ◂— 1
01:0008│+008     0x7fff4aff1888 —▸ 0x7f0b1684fd90 (__libc_start_call_main+128) ◂— mov edi, eax
02:0010│+010     0x7fff4aff1890 ◂— 0
03:0018│+018     0x7fff4aff1898 —▸ 0x55a15a4b9209 (main) ◂— endbr64
04:0020│+020     0x7fff4aff18a0 ◂— 0x100000000
05:0028│+028     0x7fff4aff18a8 —▸ 0x7fff4aff1998 —▸ 0x7fff4aff2d75 ◂— '/home/random/random'
06:0030│+030     0x7fff4aff18b0 ◂— 0
07:0038│+038     0x7fff4aff18b8 ◂— 0xcccab18ea35ab94b
─────────────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────────────
 ► 0   0x55a15a4b9211 main+8
   1   0x7f0b1684fd90 __libc_start_call_main+128
   2   0x7f0b1684fe40 __libc_start_main+128
   3   0x55a15a4b9145 _start+37
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> break *0x55a15a4b922f
Breakpoint 3 at 0x55a15a4b922f
pwndbg> c
Continuing.

Breakpoint 3, 0x000055a15a4b922f in main ()
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]─────────────────────────────────
*RAX  0x6b8b4567
 RBX  0
*RCX  0x7f0b16a40214 (randtbl+20) ◂— 0x61048c054e508aaa
*RDX  0
*RDI  0x7f0b16a40860 (unsafe_state) —▸ 0x7f0b16a40214 (randtbl+20) ◂— 0x61048c054e508aaa
*RSI  0x7fff4aff1834 ◂— 0x549ae1006b8b4567
*R8   0
*R9   0x7f0b16a40280 (pa_next_type) ◂— 8
 R10  0x7f0b16a5d908 ◂— 0xd00120000000e
 R11  0x7f0b16a78660 (_dl_audit_preinit) ◂— endbr64
 R12  0x7fff4aff1998 —▸ 0x7fff4aff2d75 ◂— '/home/random/random'
 R13  0x55a15a4b9209 (main) ◂— endbr64
 R14  0x55a15a4bbd90 (__do_global_dtors_aux_fini_array_entry) —▸ 0x55a15a4b91c0 (__do_global_dtors_aux) ◂— endbr64
 R15  0x7f0b16a97040 (_rtld_global) —▸ 0x7f0b16a982e0 —▸ 0x55a15a4b8000 ◂— 0x10102464c457f
 RBP  0x7fff4aff1880 ◂— 1
*RSP  0x7fff4aff1860 ◂— 0
*RIP  0x55a15a4b922f (main+38) ◂— mov dword ptr [rbp - 0x1c], eax
─────────────────────────────────────────[ DISASM / x86-64 / set emulate off ]─────────────────────────────────────────
   0x55a15a4b9216 <main+13>    mov    rax, qword ptr fs:[0x28]        RAX, [0x7f0b16823768]
   0x55a15a4b921f <main+22>    mov    qword ptr [rbp - 0x18], rax
   0x55a15a4b9223 <main+26>    xor    eax, eax                        EAX => 0
   0x55a15a4b9225 <main+28>    mov    eax, 0                          EAX => 0
   0x55a15a4b922a <main+33>    call   rand@plt                    <rand@plt>

 ► 0x55a15a4b922f <main+38>    mov    dword ptr [rbp - 0x1c], eax     [0x7fff4aff1864] <= 0x6b8b4567
   0x55a15a4b9232 <main+41>    mov    dword ptr [rbp - 0x20], 0
   0x55a15a4b9239 <main+48>    lea    rax, [rbp - 0x20]
   0x55a15a4b923d <main+52>    mov    rsi, rax
   0x55a15a4b9240 <main+55>    lea    rax, [rip + 0xdc1]              RAX => 0x55a15a4ba008 ◂— 0x21646f6f47006425 /* '%d' */
   0x55a15a4b9247 <main+62>    mov    rdi, rax
───────────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────────
00:0000│ rsp 0x7fff4aff1860 ◂— 0
01:0008│-018 0x7fff4aff1868 ◂— 0xbec07105549ae100
02:0010│-010 0x7fff4aff1870 ◂— 0
03:0018│-008 0x7fff4aff1878 ◂— 0
04:0020│ rbp 0x7fff4aff1880 ◂— 1
05:0028│+008 0x7fff4aff1888 —▸ 0x7f0b1684fd90 (__libc_start_call_main+128) ◂— mov edi, eax
06:0030│+010 0x7fff4aff1890 ◂— 0
07:0038│+018 0x7fff4aff1898 —▸ 0x55a15a4b9209 (main) ◂— endbr64
─────────────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────────────
 ► 0   0x55a15a4b922f main+38
   1   0x7f0b1684fd90 __libc_start_call_main+128
   2   0x7f0b1684fe40 __libc_start_main+128
   3   0x55a15a4b9145 _start+37
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

0x6b8b4567 = 1804289383
```
random@ubuntu:~$ (python3 -c "print(1804289383^3405691582)") | ./random
```
