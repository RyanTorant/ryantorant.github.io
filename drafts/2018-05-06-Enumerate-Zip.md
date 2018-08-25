---
layout: post
title: A bit more functional - Enumerate and Zip on C++
---


```asm
        int r = 0;
	auto iter_v = v0.begin();
00007FF7256F10A6  mov         rsi,qword ptr [rbp+37h]  
00007FF7256F10AA  mov         rax,rsi  
	auto iter_l = l0.cbegin();
00007FF7256F10AD  mov         rdi,qword ptr [rbp+27h]  
00007FF7256F10B1  mov         rcx,qword ptr [rdi]  
	while (iter_v != v0.end() && iter_l != l0.end())
00007FF7256F10B4  mov         r9,qword ptr [rbp+3Fh]  
00007FF7256F10B8  cmp         rsi,r9  
00007FF7256F10BB  je          main+0DEh (07FF7256F10DEh)  
00007FF7256F10BD  mov         rdx,rcx  
00007FF7256F10C0  cmp         rdx,rdi  
00007FF7256F10C3  je          main+0DEh (07FF7256F10DEh)  
	{
		r += *iter_v + *iter_l;
00007FF7256F10C5  mov         r8d,dword ptr [rcx+10h]  
00007FF7256F10C9  add         r8d,dword ptr [rax]  
00007FF7256F10CC  add         ebx,r8d  
		iter_v++;
00007FF7256F10CF  add         rax,4  
		iter_v++;
00007FF7256F10D3  mov         rdx,qword ptr [rcx]  
		iter_l++;
00007FF7256F10D6  mov         rcx,rdx  
	while (iter_v != v0.end() && iter_l != l0.end())
00007FF7256F10D9  cmp         rax,r9  
00007FF7256F10DC  jne         main+0C0h (07FF7256F10C0h)  
	}
```

```asm
        int r = 0;
00007FF7468010A1  mov         r8d,ebx  
	for (auto[x, y] : zip(v0, l0))
00007FF7468010A4  mov         rax,qword ptr [rbp+37h]  
00007FF7468010A8  mov         r9,qword ptr [rbp+27h]  
00007FF7468010AC  mov         rcx,qword ptr [r9]  
00007FF7468010AF  mov         r10,qword ptr [rbp+3Fh]  
00007FF7468010B3  cmp         rax,r10  
00007FF7468010B6  je          main+0CEh (07FF7468010CEh)  
	for (auto[x, y] : zip(v0, l0))
00007FF7468010B8  cmp         rcx,r9  
00007FF7468010BB  je          main+0CEh (07FF7468010CEh)  
		r += x + y;
00007FF7468010BD  mov         edx,dword ptr [rcx+10h]  
00007FF7468010C0  add         edx,dword ptr [rax]  
00007FF7468010C2  add         r8d,edx  
	for (auto[x, y] : zip(v0, l0))
00007FF7468010C5  add         rax,4  
00007FF7468010C9  mov         rcx,qword ptr [rcx]  
00007FF7468010CC  jmp         main+0B3h (07FF7468010B3h)  
```

| Version    | Time (ms) |
| ---------- | :-------: |
| Direct     |    309    |
| Zip        |    302    |