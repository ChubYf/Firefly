---
title: PE反射注入(内嵌exe版)
published: 2026-05-10
description: exe嵌套exe
image: ./cover.jpg
tags: [逆向]
category: Windows逆向
draft: false
---





出于学习PE结构的目的，以ctf题目形式写了一个exe内嵌第二层exe，并手动加载第二层exe。即在第一层exe执行main函数之前执行内嵌的第二层exe程序。主要是实现手动加载第二层exe（模拟windows加载器加载pe文件的部分过程-PE拉伸-导入表修复-重定位修复）

# 在main函数之前执行自定义代码

可以看我之前写的qwq
[crt初始化注入自定义代码 - Chuby's blog](https://blog.213188.xyz/posts/windows/crt%E5%88%9D%E5%A7%8B%E5%8C%96%E6%B3%A8%E5%85%A5%E8%87%AA%E5%AE%9A%E4%B9%89%E4%BB%A3%E7%A0%81/)
```c
#pragma section(".CRT$XCU", read)
__declspec(allocate(".CRT$XCU")) void(*p)() = modefied;
```




# 手动加载第二层exe

## 第二层exe代码

copy之前crt初始化写的

```c
#include <iostream>
#include <Windows.h>
#include <stdint.h>
#include <stdlib.h>
using namespace std;

char welcome[] = "this is a challenge";
char key = 0x4d;

void modefied() {
	if (IsDebuggerPresent()) {
		return;
	}
	char* key_addr = (char* )welcome + 0x14;
	*key_addr = 0x5d;
	return;
}

#pragma section(".CRT$XCU", read)
__declspec(allocate(".CRT$XCU")) void(*p)() = modefied;

int main() {
	char buffer[256];
	char entext[] = { 59, 49, 60, 58, 38, 41, 53, 52, 46, 2, 60, 59, 49, 60, 58, 32 };
	cout << welcome << endl;
	cout << "please input your flag" << endl;
	cin >> buffer;
	int length = strlen(buffer);
	if (length != 16) {
		cout << "wrong length" << endl;
		return 0;
	}
	for (int i = 0; i < 16; i++) {
		buffer[i] = buffer[i] ^ key;
	}
	for (int i = 0; i < 16; i++) {
		if (buffer[i] != entext[i]) {
			cout << "wrong flag" << endl;
			return 0;
		}
	}
	cout << "correct flag" << endl;
	return 0;
}
```

## 加密第二层exe代码

编译生成exe后用010editor打开提取十六进制数据，然后对十六进制数据异或0x5a
```python
# 复制为 Python - 从 010 Editor - 字节计数：90624 (0x16200)
buffer = b''.join(['你的十六进制数据'])

encrypted_data = 0
with open("encrypt_dump.txt", "a") as f:
    for i in range(len(buffer)):
        encrypted_data = buffer[i] ^ 0x5a
        f.write(hex(encrypted_data))
        f.write(",")
        if (i % 16 == 0):
            f.write("\n")
print("ok")
```
把加密后的十六进制数据存下来后面以全局变量的形式写在第一层PE中。


## 手动加载第二层PE

### 搭建框架

```c
void modefied()
{
	if (IsDebuggerPresent())
	{
		return;
	}
	else
	{
		DecryptData(encrypted_data, sizeof(encrypted_data));
	}
	if (!VerifyData(encrypted_data))
	{
		return;
	}

	PBYTE lpImageBuffer = MyLoadLibrary(encrypted_data);
	if (!lpImageBuffer)
	{
		return;
	}


	MyExe(lpImageBuffer);


	return;

}
```
modefied是自己之前在crt初始化注册的函数。
检查是否被调试，如果是则返回执行第一层PE，否则对加密数据进行解密( ^ 0x5a) 
```c
void DecryptData(unsigned char* data, size_t len)
{
	for (int i = 0; i < len; i++)
	{
		data[i] ^= 0x5a;
	}
}
```

解密完数据后检查是否为有效的PE文件格式
```c
BOOL VerifyData(unsigned char* data)
{
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)data;
	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return FALSE;
	}

	PIMAGE_NT_HEADERS pNtHeader = (PIMAGE_NT_HEADERS)(data + pDosHeader->e_lfanew);
	if (pNtHeader->Signature != IMAGE_NT_SIGNATURE)
	{
		return FALSE;
	}

	return TRUE;
}
```
主要检查dos头的signature(MZ)和nt头的signature(PE)
然后实现自己的LoadLibraryA(将PE文件拉伸到内存并进行导入表修复，重定位修复)

以上操作失败都会返回执行第一层PE，如果成功则执行MyExe执行第二层PE入口点（执行完第二层PE后不会再执行第一层PE，因为第二层PE执行完后会把当前进程关闭）。


### 实现LoadLibrary手动加载PE文件


#### 整体框架：

```c
PBYTE MyLoadLibrary(unsigned char* data)
{
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)data;
	PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(data + pDosHeader->e_lfanew);
	PIMAGE_SECTION_HEADER pSectionHeader = IMAGE_FIRST_SECTION(pNtHeaders);


	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return NULL;
	}
	if (pNtHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		return NULL;
	}


	DWORD dwImageSize = pNtHeaders->OptionalHeader.SizeOfImage;
	PBYTE lpImageBuffer = (PBYTE)VirtualAlloc(NULL, dwImageSize, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
	if (!lpImageBuffer)
	{
		return NULL;
	}


	memcpy(lpImageBuffer, data, pNtHeaders->OptionalHeader.SizeOfHeaders);

	for (size_t i = 0; i < pNtHeaders->FileHeader.NumberOfSections; i++)
	{
		if (pSectionHeader[i].SizeOfRawData == 0)
		{
			continue;
		}
		memcpy(
			lpImageBuffer + pSectionHeader[i].VirtualAddress,
			data + pSectionHeader[i].PointerToRawData,
			min(pSectionHeader[i].SizeOfRawData, pSectionHeader[i].Misc.VirtualSize)
		);
	}

	LONGLONG llDelta = (LONGLONG)(lpImageBuffer - pNtHeaders->OptionalHeader.ImageBase);

	if (!FixImportTable(lpImageBuffer))
	{
		return NULL;
	}


	if (!FixRelocationTable(lpImageBuffer, llDelta))
	{
		return NULL;
	}



	return lpImageBuffer;

}
```




#### PE拉伸：
```c
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)data;
	PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(data + pDosHeader->e_lfanew);
	PIMAGE_SECTION_HEADER pSectionHeader = IMAGE_FIRST_SECTION(pNtHeaders);


	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return NULL;
	}
	if (pNtHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		return NULL;
	}


	DWORD dwImageSize = pNtHeaders->OptionalHeader.SizeOfImage;
	PBYTE lpImageBuffer = (PBYTE)VirtualAlloc(NULL, dwImageSize, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
	if (!lpImageBuffer)
	{
		return NULL;
	}


	memcpy(lpImageBuffer, data, pNtHeaders->OptionalHeader.SizeOfHeaders);

	for (size_t i = 0; i < pNtHeaders->FileHeader.NumberOfSections; i++)
	{
		if (pSectionHeader[i].SizeOfRawData == 0)
		{
			continue;
		}
		memcpy(
			lpImageBuffer + pSectionHeader[i].VirtualAddress,
			data + pSectionHeader[i].PointerToRawData,
			min(pSectionHeader[i].SizeOfRawData, pSectionHeader[i].Misc.VirtualSize)
		);
	}
```



#### 导入表修复:
```c
BOOL FixImportTable(PBYTE lpImageBuffer)
{
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)lpImageBuffer;
	PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(lpImageBuffer + pDosHeader->e_lfanew);
	PIMAGE_SECTION_HEADER pSectionHeader = IMAGE_FIRST_SECTION(pNtHeaders);

	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return FALSE;
	}
	if (pNtHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		return FALSE;
	}

	PIMAGE_DATA_DIRECTORY pImportDir = &pNtHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT];
	if (pImportDir->Size == 0 || pImportDir->VirtualAddress == 0)
	{
		return FALSE;
	}

	PIMAGE_IMPORT_DESCRIPTOR pImportDesc = (PIMAGE_IMPORT_DESCRIPTOR)(pImportDir->VirtualAddress + lpImageBuffer);

	for (size_t i = 0; pImportDesc[i].Name != 0; i++)
	{
		PCHAR DllName = (PCHAR)(pImportDesc[i].Name + lpImageBuffer);
		HMODULE hModule = LoadLibraryA(DllName);

		if (!hModule)
		{
			continue;
		}

		PIMAGE_THUNK_DATA pINT = NULL;
		PIMAGE_THUNK_DATA pIAT = (PIMAGE_THUNK_DATA)(pImportDesc[i].FirstThunk + lpImageBuffer);

		if (pImportDesc[i].OriginalFirstThunk != 0)
		{
			pINT = (PIMAGE_THUNK_DATA)(pImportDesc[i].OriginalFirstThunk + lpImageBuffer);
		}
		else
		{
			pINT = pIAT;
		}


		for (size_t i = 0; pINT[i].u1.AddressOfData != 0; i++)
		{
			FARPROC procAddr = NULL;

			if (IMAGE_SNAP_BY_ORDINAL(pINT[i].u1.Ordinal))
			{
				DWORD ordinal = IMAGE_ORDINAL(pINT[i].u1.Ordinal);
				procAddr = GetProcAddress(hModule, (LPCSTR)(ULONG_PTR)ordinal);
			}

			else
			{
				PIMAGE_IMPORT_BY_NAME pImportByName = (PIMAGE_IMPORT_BY_NAME)(lpImageBuffer + pINT[i].u1.AddressOfData);
				procAddr = GetProcAddress(hModule, (LPCSTR)pImportByName->Name);
			}

			if (procAddr)
			{
				pIAT[i].u1.Function = (ULONG_PTR)procAddr;
			}

		}

	}

	return TRUE;

}
```



#### 重定位修复：
```c
BOOL FixRelocationTable(PBYTE lpImageBuffer, LONGLONG Delta)
{
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)lpImageBuffer;
	PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(lpImageBuffer + pDosHeader->e_lfanew);
	PIMAGE_SECTION_HEADER pSectionHeader = IMAGE_FIRST_SECTION(pNtHeaders);


	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return FALSE;
	}
	if (pNtHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		return FALSE;
	}

	PIMAGE_DATA_DIRECTORY pRelocDir = &pNtHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC];
	if (pRelocDir->VirtualAddress == 0 || pRelocDir->Size == 0)
	{
		return FALSE;
	}

	PIMAGE_BASE_RELOCATION pRelocBlock = (PIMAGE_BASE_RELOCATION)(lpImageBuffer + pRelocDir->VirtualAddress);
	if (!pRelocBlock)
	{
		return FALSE;
	}


	while (pRelocBlock->VirtualAddress != 0 && pRelocBlock->SizeOfBlock != 0)
	{

		DWORD entryCount = (pRelocBlock->SizeOfBlock - sizeof(IMAGE_BASE_RELOCATION)) / sizeof(WORD);
		PWORD pEntry = (PWORD)(pRelocBlock + 1);

		for (size_t i = 0; i < entryCount; i++)
		{
			WORD entry = pEntry[i];
			BYTE type = (entry >> 12) & 0xf;
			WORD offset = entry & 0xfff;

			DWORD FixRva = pRelocBlock->VirtualAddress + offset;
			ULONG_PTR pFixAddress = (ULONG_PTR)(lpImageBuffer + FixRva);

			switch (type)
			{
			case IMAGE_REL_BASED_ABSOLUTE:
				continue;

			case IMAGE_REL_BASED_HIGHLOW:
				*(PDWORD)pFixAddress += (DWORD)Delta;
				break;

			case IMAGE_REL_BASED_DIR64:
				*(PLONGLONG)pFixAddress += Delta;
				break;


			default:
				break;

			}
		}

		pRelocBlock = (PIMAGE_BASE_RELOCATION)((BYTE*)pRelocBlock + pRelocBlock->SizeOfBlock);
	}



	return TRUE;
}
```



## 执行第二层PE


```c
void MyExe(PBYTE lpImageBuffer)
{
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)lpImageBuffer;
	PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(lpImageBuffer + pDosHeader->e_lfanew);
	PIMAGE_SECTION_HEADER pSectionHeader = IMAGE_FIRST_SECTION(pNtHeaders);


	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return;
	}
	if (pNtHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		return;
	}

	ULONG_PTR AddressOfEntryPoint = (ULONG_PTR)(lpImageBuffer + pNtHeaders->OptionalHeader.AddressOfEntryPoint);

	typedef int (WINAPI* PENTRYPOINT)(void);
	PENTRYPOINT EntryPoint = (PENTRYPOINT)AddressOfEntryPoint;

	int result = EntryPoint();

}
```



# 完整代码

```c
#include <Windows.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

unsigned char encrypted_data[] = { 第二层加密代码 };



void DecryptData(unsigned char* data, size_t len)
{
	for (int i = 0; i < len; i++)
	{
		data[i] ^= 0x5a;
	}
}

BOOL VerifyData(unsigned char* data)
{
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)data;
	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return FALSE;
	}

	PIMAGE_NT_HEADERS pNtHeader = (PIMAGE_NT_HEADERS)(data + pDosHeader->e_lfanew);
	if (pNtHeader->Signature != IMAGE_NT_SIGNATURE)
	{
		return FALSE;
	}

	return TRUE;
}

BOOL FixImportTable(PBYTE lpImageBuffer)
{
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)lpImageBuffer;
	PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(lpImageBuffer + pDosHeader->e_lfanew);
	PIMAGE_SECTION_HEADER pSectionHeader = IMAGE_FIRST_SECTION(pNtHeaders);

	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return FALSE;
	}
	if (pNtHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		return FALSE;
	}

	PIMAGE_DATA_DIRECTORY pImportDir = &pNtHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT];
	if (pImportDir->Size == 0 || pImportDir->VirtualAddress == 0)
	{
		return FALSE;
	}

	PIMAGE_IMPORT_DESCRIPTOR pImportDesc = (PIMAGE_IMPORT_DESCRIPTOR)(pImportDir->VirtualAddress + lpImageBuffer);

	for (size_t i = 0; pImportDesc[i].Name != 0; i++)
	{
		PCHAR DllName = (PCHAR)(pImportDesc[i].Name + lpImageBuffer);
		HMODULE hModule = LoadLibraryA(DllName);

		if (!hModule)
		{
			continue;
		}

		PIMAGE_THUNK_DATA pINT = NULL;
		PIMAGE_THUNK_DATA pIAT = (PIMAGE_THUNK_DATA)(pImportDesc[i].FirstThunk + lpImageBuffer);

		if (pImportDesc[i].OriginalFirstThunk != 0)
		{
			pINT = (PIMAGE_THUNK_DATA)(pImportDesc[i].OriginalFirstThunk + lpImageBuffer);
		}
		else
		{
			pINT = pIAT;
		}


		for (size_t i = 0; pINT[i].u1.AddressOfData != 0; i++)
		{
			FARPROC procAddr = NULL;

			if (IMAGE_SNAP_BY_ORDINAL(pINT[i].u1.Ordinal))
			{
				DWORD ordinal = IMAGE_ORDINAL(pINT[i].u1.Ordinal);
				procAddr = GetProcAddress(hModule, (LPCSTR)(ULONG_PTR)ordinal);
			}

			else
			{
				PIMAGE_IMPORT_BY_NAME pImportByName = (PIMAGE_IMPORT_BY_NAME)(lpImageBuffer + pINT[i].u1.AddressOfData);
				procAddr = GetProcAddress(hModule, (LPCSTR)pImportByName->Name);
			}

			if (procAddr)
			{
				pIAT[i].u1.Function = (ULONG_PTR)procAddr;
			}

		}

	}

	return TRUE;

}

BOOL FixRelocationTable(PBYTE lpImageBuffer, LONGLONG Delta)
{
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)lpImageBuffer;
	PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(lpImageBuffer + pDosHeader->e_lfanew);
	PIMAGE_SECTION_HEADER pSectionHeader = IMAGE_FIRST_SECTION(pNtHeaders);


	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return FALSE;
	}
	if (pNtHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		return FALSE;
	}

	PIMAGE_DATA_DIRECTORY pRelocDir = &pNtHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC];
	if (pRelocDir->VirtualAddress == 0 || pRelocDir->Size == 0)
	{
		return FALSE;
	}

	PIMAGE_BASE_RELOCATION pRelocBlock = (PIMAGE_BASE_RELOCATION)(lpImageBuffer + pRelocDir->VirtualAddress);
	if (!pRelocBlock)
	{
		return FALSE;
	}


	while (pRelocBlock->VirtualAddress != 0 && pRelocBlock->SizeOfBlock != 0)
	{

		DWORD entryCount = (pRelocBlock->SizeOfBlock - sizeof(IMAGE_BASE_RELOCATION)) / sizeof(WORD);
		PWORD pEntry = (PWORD)(pRelocBlock + 1);

		for (size_t i = 0; i < entryCount; i++)
		{
			WORD entry = pEntry[i];
			BYTE type = (entry >> 12) & 0xf;
			WORD offset = entry & 0xfff;

			DWORD FixRva = pRelocBlock->VirtualAddress + offset;
			ULONG_PTR pFixAddress = (ULONG_PTR)(lpImageBuffer + FixRva);

			switch (type)
			{
			case IMAGE_REL_BASED_ABSOLUTE:
				continue;

			case IMAGE_REL_BASED_HIGHLOW:
				*(PDWORD)pFixAddress += (DWORD)Delta;
				break;

			case IMAGE_REL_BASED_DIR64:
				*(PLONGLONG)pFixAddress += Delta;
				break;


			default:
				break;

			}
		}

		pRelocBlock = (PIMAGE_BASE_RELOCATION)((BYTE*)pRelocBlock + pRelocBlock->SizeOfBlock);
	}



	return TRUE;
}

PBYTE MyLoadLibrary(unsigned char* data)
{
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)data;
	PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(data + pDosHeader->e_lfanew);
	PIMAGE_SECTION_HEADER pSectionHeader = IMAGE_FIRST_SECTION(pNtHeaders);


	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return NULL;
	}
	if (pNtHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		return NULL;
	}


	DWORD dwImageSize = pNtHeaders->OptionalHeader.SizeOfImage;
	PBYTE lpImageBuffer = (PBYTE)VirtualAlloc(NULL, dwImageSize, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
	if (!lpImageBuffer)
	{
		return NULL;
	}


	memcpy(lpImageBuffer, data, pNtHeaders->OptionalHeader.SizeOfHeaders);

	for (size_t i = 0; i < pNtHeaders->FileHeader.NumberOfSections; i++)
	{
		if (pSectionHeader[i].SizeOfRawData == 0)
		{
			continue;
		}
		memcpy(
			lpImageBuffer + pSectionHeader[i].VirtualAddress,
			data + pSectionHeader[i].PointerToRawData,
			min(pSectionHeader[i].SizeOfRawData, pSectionHeader[i].Misc.VirtualSize)
		);
	}

	LONGLONG llDelta = (LONGLONG)(lpImageBuffer - pNtHeaders->OptionalHeader.ImageBase);

	if (!FixImportTable(lpImageBuffer))
	{
		return NULL;
	}


	if (!FixRelocationTable(lpImageBuffer, llDelta))
	{
		return NULL;
	}



	return lpImageBuffer;

}

void MyExe(PBYTE lpImageBuffer)
{
	PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)lpImageBuffer;
	PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(lpImageBuffer + pDosHeader->e_lfanew);
	PIMAGE_SECTION_HEADER pSectionHeader = IMAGE_FIRST_SECTION(pNtHeaders);


	if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return;
	}
	if (pNtHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		return;
	}

	ULONG_PTR AddressOfEntryPoint = (ULONG_PTR)(lpImageBuffer + pNtHeaders->OptionalHeader.AddressOfEntryPoint);

	typedef int (WINAPI* PENTRYPOINT)(void);
	PENTRYPOINT EntryPoint = (PENTRYPOINT)AddressOfEntryPoint;

	int result = EntryPoint();

}

void modefied()
{
	if (IsDebuggerPresent())
	{
		return;
	}
	else
	{
		DecryptData(encrypted_data, sizeof(encrypted_data));
	}
	if (!VerifyData(encrypted_data))
	{
		return;
	}

	PBYTE lpImageBuffer = MyLoadLibrary(encrypted_data);
	if (!lpImageBuffer)
	{
		return;
	}


	MyExe(lpImageBuffer);


	return;

}



#pragma section(".CRT$XCU", read)
__declspec(allocate(".CRT$XCU")) void(*p)() = modefied;


char welcome[] = "this is a challenge";
char key = 0x5a;


int main() {
	char buffer[256];
	char entext[] = { 0x2E,0x32,0x33,0x29,0x05,0x33,0x29,0x05,0x3B,0x05,0x3C,0x3B,0x31,0x3F,0x3F,0x3F };
	printf("%s\n", welcome);
	printf("please input your flag\n");
	scanf_s("%255s", buffer, (unsigned)_countof(buffer));
	size_t length = strlen(buffer);
	if (length != 16) {
		printf("wrong length\n");
		return 0;
	}
	for (int i = 0; i < 16; i++) {
		buffer[i] = buffer[i] ^ key;
	}
	for (int i = 0; i < 16; i++) {
		if (buffer[i] != entext[i]) {
			printf("wrong\n");
			return 0;
		}
	}
	printf("this is not right place\n");
	return 0;
}
```


# 效果

ida无法查找到第二层exe的输入成功或者失败的字符串提示。动态调试时如果没有过掉反调试那么不会执行到第二层PE当中。


# 痕迹

## 1.
在导入表中可以看到恨到敏感的api函数，通过这些api函数可以定位到加载第二层PE代码的位置。

![](assets/Pasted%20image%2020260510145315.png)

比如可以通过我们在源代码中使用的api函数isdebuggerpresent，loadlibrary，getprocaddress定位到第二层pe代码。

## 2.
直接查找crt函数指针表可以找到我们写的modefied函数。



