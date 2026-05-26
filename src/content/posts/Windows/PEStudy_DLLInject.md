---
title: PEStudy2_dll反射注入
published: 2026-05-26
description: dll注入目标进程
image: ./cover.jpg
tags: [逆向]
category: Windows逆向
draft: false
---

# 前言

出于学习PE结构目的，写一个程序把dll注入到目标进程中并弹出消息框。只适用于32位程序


# 脚本编写

## 思路

写一个Dll动态链接库，在dllmain入口点弹窗。
写一个DllInject程序，实现通过进程id或者进程名字获取到目标进程Pid, 获取用户输入的dll，将dll反射注入到目标进程中（pe拉伸，导入表修复，重定位修复）, 然后再目标进程写一个shellcode，创建线程执行dllmain入口。

## 获取目标进程PID

通过输入的PID获取或者通过输入的进程名称查找PID

```c
//获取用户输入目标进程id,名字 得到目标进程id
	DWORD targetPid = 0;
	CHAR targetProcessName[MAX_PATH] = { 0 };
	CHAR input[255] = { 0 };

	printf("Input Target ID/Name\n");
	printf("1. Process ID\n");
	printf("2. Process Name\n");
	printf("Enter choice(1 / 2): ");

	fgets(input, sizeof(input), stdin);
	int choice = atoi(input);

	if (choice == 1)
	{
		printf("Please input target process ID: ");
		fgets(input, sizeof(input), stdin);
		targetPid = atoi(input);
		if (targetPid == 0)
		{
			printf("Invalid process ID\n");
			return 0;
		}
	}
	else if (choice == 2)
	{
		printf("Please input target process name: ");
		fgets(input, sizeof(input), stdin);
		size_t len = strlen(input);
		if (len > 0 && input[len - 1] == '\n')
		{
			input[len - 1] = '\0';
		}
		strcpy(targetProcessName, input);

		targetPid = FindProcessIdByName((LPCTSTR)targetProcessName);

		if (targetPid == 0)
		{
			printf("find process id by name: %s wrong\n", targetProcessName);
			return 0;
		}

	}
	else
	{
		printf("wrong choice\n");
		system("pause");
		return 0;
	}
```


通过进程名字获取PID。使用CreateToolhelp32Snapshot api函数获取当前系统所有进程的快照，使用Process32First, Process32Next遍历每一个进程。pe32存储每一个进程的信息
```c
DWORD FindProcessIdByName(LPCTSTR targetProcessName)
{
	HANDLE hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
	if (hSnapShot == INVALID_HANDLE_VALUE)
	{
		printf("Failed to create process snapshot\n");
		return 0;
	}
	PROCESSENTRY32 pe32 = { 0 };
	pe32.dwSize = sizeof(PROCESSENTRY32);

	if (!Process32First(hSnapShot, &pe32))
	{
		printf("Failed to get first process\n");
		CloseHandle(hSnapShot);
		return 0;
	}

	DWORD processId = 0;
	do
	{
		if (_stricmp(pe32.szExeFile, targetProcessName) == 0)
		{
			processId = pe32.th32ProcessID;
			break;
		}
	} while (Process32Next(hSnapShot, &pe32));

	CloseHandle(hSnapShot);
	return processId;
}
```

## 获取dll路径

```c
	char dllPath[MAX_PATH] = { 0 };
	printf("Please input dll file path: ");
	fgets(input, sizeof(input), stdin);
	size_t len = strlen(input);
	if (len > 0 && input[len - 1] == '\n')
	{
		input[len - 1] = '\0';
	}

	strcpy(dllPath, input);

	if (dllPath[0] == '\0')
	{
		printf("Invalid dll path\n");
		return 0;
	}

	system("cls");
	printf("target PID: %d \n", targetPid);
	printf("Dll Path: %s\n", dllPath);
```


## 分析dll信息

打开dll文件，映射到当前进程内存中(fileBuffer)，然后进行初步PE拉伸(imageBuffer)，同时记录dos,nt,section头信息
```c
typedef struct
{
	HANDLE hFile;
	DWORD fileSize;
	PBYTE fileBuffer;
	PBYTE imageBuffer;
	DWORD imageSize;

	PIMAGE_DOS_HEADER dosHeader;
	PIMAGE_NT_HEADERS ntHeaders;
	PIMAGE_SECTION_HEADER sectionHeader;

	LPVOID remoteBaseAddress;

}PE_CONTEXT;

BOOL LoadPeFile(char* dllPath, PE_CONTEXT* peContext)
{
	if (!dllPath || !peContext) return FALSE;
	//初始化peContext内容
	memset(peContext, 0, sizeof(peContext));


	//打开文件
	peContext->hFile = CreateFileA(
		dllPath,
		GENERIC_READ,
		FILE_SHARE_READ,
		NULL,
		OPEN_EXISTING,
		FILE_ATTRIBUTE_NORMAL,
		NULL);

	if (peContext->hFile == INVALID_HANDLE_VALUE)
	{
		printf("Failed to open dll file: %s\n", dllPath);
		return FALSE;
	}

	//获取文件大小
	peContext->fileSize = GetFileSize(peContext->hFile, NULL);

	if (peContext->fileSize == INVALID_FILE_SIZE)
	{
		printf("Failed to get file size: %s", dllPath);
		CloseHandle(peContext->hFile);
		peContext->hFile = INVALID_HANDLE_VALUE;
		return FALSE;
	}

	//将文件映射到内存中
	
	peContext->fileBuffer = (PBYTE)VirtualAlloc(
		NULL,
		peContext->fileSize,
		MEM_COMMIT | MEM_RESERVE,
		PAGE_READWRITE);
	
	if (!peContext->fileBuffer)
	{
		printf("Failed to allocate memory for file: %s\n", dllPath);
		CloseHandle(peContext->hFile);
		peContext->hFile = INVALID_HANDLE_VALUE;
		return FALSE;
	}

	DWORD byteRead = 0;
	if (!ReadFile(
		peContext->hFile,
		peContext->fileBuffer,
		peContext->fileSize,
		&byteRead,
		NULL) || byteRead != peContext->fileSize)
	{
		printf("Failed to read file: %s\n", dllPath);
		VirtualFree(peContext->fileBuffer, 0, MEM_RELEASE);
		peContext->fileBuffer = NULL;
		CloseHandle(peContext->hFile);
		peContext->hFile = INVALID_HANDLE_VALUE;
		return FALSE;
	}

	CloseHandle(peContext->hFile);
	peContext->hFile = INVALID_HANDLE_VALUE;

	//解析dos,nt,section头
	peContext->dosHeader = (PIMAGE_DOS_HEADER)(peContext->fileBuffer);
	if (peContext->dosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		printf("Invalid dos signature\n");
		UnLoadPeFile(peContext);
		return FALSE;
	}

	peContext->ntHeaders = (PIMAGE_NT_HEADERS)(peContext->fileBuffer + peContext->dosHeader->e_lfanew);
	if (peContext->ntHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		printf("Invalid nt signature\n");
		UnLoadPeFile(peContext);
		return FALSE;
	}
	//检测是否为dll文件
	if (!(peContext->ntHeaders->FileHeader.Characteristics & IMAGE_FILE_DLL))
	{
		printf("Input file is not a dll file\n");
		UnLoadPeFile(peContext);
		return FALSE;
	}

	peContext->sectionHeader = IMAGE_FIRST_SECTION(peContext->ntHeaders);

	//将dll文件在内存中拉伸
	peContext->imageBuffer = (PBYTE)VirtualAlloc(
		NULL,
		peContext->ntHeaders->OptionalHeader.SizeOfImage,
		MEM_COMMIT | MEM_RESERVE,
		PAGE_READWRITE);


	if (!peContext->imageBuffer)
	{
		printf("Failed to allocate image buffer\n");
		UnLoadPeFile(peContext);
		return FALSE;
	}
	memset(peContext->imageBuffer, 0, peContext->ntHeaders->OptionalHeader.SizeOfImage);

	//copy 头
	memcpy(peContext->imageBuffer, peContext->fileBuffer, peContext->ntHeaders->OptionalHeader.SizeOfHeaders);
	
	//copy 节区
	for (size_t i = 0; i < peContext->ntHeaders->FileHeader.NumberOfSections; i++)
	{
		if (peContext->sectionHeader[i].SizeOfRawData == 0)
		{
			continue;
		}

		memcpy(
			peContext->imageBuffer + peContext->sectionHeader[i].VirtualAddress,
			peContext->fileBuffer + peContext->sectionHeader[i].PointerToRawData,
			min(peContext->sectionHeader[i].SizeOfRawData, peContext->sectionHeader[i].Misc.VirtualSize)
			);
		
	}

	peContext->imageSize = peContext->ntHeaders->OptionalHeader.SizeOfImage;

	return TRUE;
}

VOID UnLoadPeFile(PE_CONTEXT* context)
{
	if (!context) return;

	if (context->fileBuffer)
	{
		VirtualFree(context->fileBuffer, 0, MEM_RELEASE);
		context->fileBuffer = NULL;
	}

	if (context->hFile && context->hFile != INVALID_HANDLE_VALUE)
	{
		CloseHandle(context->hFile);
		context->hFile = INVALID_HANDLE_VALUE;
	}

	if (context->imageBuffer)
	{
		VirtualFree(context->imageBuffer, 0, MEM_RELEASE);
		context->imageBuffer = NULL;
	}

	return;
}

```

## 向目标进程注入dll

通过processid用OpenProcess打开目标进程获得句柄, VirtualAllocEx和WriteProcessMemory将dll写入目标进程内存中。然后在目标进程内存中修复导入表和导出表

```c
BOOL InjectDllToProcess(DWORD processID, PE_CONTEXT* context)
{
	if (!processID || !context) return FALSE;
	
	//向目标进程写入dll
	HANDLE hProcess = OpenProcess(
		PROCESS_ALL_ACCESS,
		FALSE,
		processID);
	if (hProcess == NULL)
	{
		printf("Failed to open process: %d", processID);
		return FALSE;
	}

	LPVOID remoteBaseAddress = VirtualAllocEx(
		hProcess,
		NULL,
		context->imageSize,
		MEM_COMMIT | MEM_RESERVE,
		PAGE_EXECUTE_READWRITE);
	
	if (remoteBaseAddress == NULL)
	{
		printf("Failed to allocate memory in target process\n");
		CloseHandle(hProcess);
		return FALSE;
	}

	context->remoteBaseAddress = remoteBaseAddress;
	printf("Memory allocated in target process: %p\n", remoteBaseAddress);

	if (!WriteProcessMemory(
		hProcess,
		remoteBaseAddress,
		context->imageBuffer,
		context->imageSize,
		NULL))
	{
		printf("Failed to write memory to target process\n");
		VirtualFreeEx(hProcess, remoteBaseAddress, 0, MEM_RELEASE);
		CloseHandle(hProcess);
		return FALSE;
	}
	
	printf("Image written to target process successfully\n");

	//在目标进程内存中修复dll导入表
	if (!FixImportTable(hProcess, context))
	{
		printf("Failed to fix import table\n");
		VirtualFreeEx(hProcess, remoteBaseAddress, 0, MEM_RELEASE);
		CloseHandle(hProcess);
		return FALSE;
	}
	printf("\nFix import table successfully\n");

	//在目标进程内存中修复dll重定位表
	if (!FixRelocationTable(hProcess, context))
	{
		printf("Failed to fix relocation table\n");
		VirtualFreeEx(hProcess, remoteBaseAddress, 0, MEM_RELEASE);
		CloseHandle(hProcess);
		return FALSE;
	}
	printf("\nfix relocation table successfully\n");

	CloseHandle(hProcess);

	return TRUE;
}
```

### 修复导入表

修复导入表过程中要用到LoadLibraryA--加载导入模块和GetProcAddress--获取导入模块的每一个函数的地址，将得到的地址修复导入表。而LoadLibraryA和GetProcAddress需要在目标进程中执行，所以需要编写一个shellcode。

shellcode用VirtualAllocEx放在目标进程内存中，WriteProcessMemory写入对应内存中，然后再用CreateRemoteThread创建线程执行shellcode，用WaitForSingleObject等待线程执行完毕，GetExitCodeThread获取返回值（LoadLibraryA返回的模块句柄和GetProcAddress返回的函数地址）。

又因为CreateRemoteThread往线程传入参数（即LoadLibraryA和GetProcAddress所需参数）的类型可能是指针，不能在将自身内存地址作为参数传入，所以又需要在目标进程中VirtualAllocEx和WriteProcessMemory写入所需参数（如dll名字,导入函数名字是char\*类型的）。

因为CreateRemoteThread只能传入一个参数，而GetProcAddress需要两个参数（模块句柄和函数名称/序号），所以将模块句柄和函数名称/序号封装在一个结构体中，用VirtualAllocEx和WriteProcessMemory写入目标进程内存中，将该结构体地址作为参数传入CreateRemoteThread.

shellcode的编写：具体情况具体分析，这里大概说个思路。CreateRemoteThread创建线程执行shellcode相当于调用shellcode这个函数，因为CreateRemoteThread是winapi，采用stdcall函数调用约定，所以一开始先push ebp将旧的ebp压栈，然后把传入参数压栈push \[ebp + 8],  push  \[ebp + 0xC],把要执行的函数放在eax寄存器 mov eax, .... ,中并call eax,最后pop ebp ; ret 4(ret 8)清理栈。

最后接收GetProcAddress返回的函数地址写入IAT表当中


```c
BOOL FixImportTable(HANDLE hProcess, PE_CONTEXT* context)
{
	if (!hProcess || !context) return FALSE;

	//定位导入表数据目录项
	IMAGE_DATA_DIRECTORY importDir = context->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT];
	if (importDir.Size == 0 || importDir.VirtualAddress == 0)
	{
		printf("Not found import table\n");
		return FALSE;
	}

	PIMAGE_IMPORT_DESCRIPTOR importDesc = (PIMAGE_IMPORT_DESCRIPTOR)(context->imageBuffer + importDir.VirtualAddress);
	
	//向目标进程分配shellcode内存
	LPVOID remoteFunc = VirtualAllocEx(
		hProcess,
		NULL,
		0x1000,
		MEM_COMMIT | MEM_RESERVE,
		PAGE_EXECUTE_READWRITE);
	if (!remoteFunc)
	{
		printf("Failed to allocate remotefunc in target process\n");
		return FALSE;
	}



	for (size_t i = 0; importDesc[i].Name; i++)
	{
		//获取当前导入dll名称
		CHAR* dllName = (CHAR*)(context->imageBuffer + importDesc[i].Name);
		printf("========  Import Dll: %s ========\n", dllName);

		//将dll名称写入目标进程中
		LPVOID remoteDllName = VirtualAllocEx(
			hProcess,
			NULL,
			strlen(dllName) + 1,
			MEM_COMMIT | MEM_RESERVE,
			PAGE_READWRITE);
		if (!remoteDllName)
		{
			printf("Failed to allocate memory for dll name in target process\n");
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}
		if (!WriteProcessMemory(
			hProcess,
			remoteDllName,
			dllName,
			strlen(dllName) + 1,
			NULL))
		{
			printf("Failed to write memory for dll name in target process\n");
			VirtualFreeEx(hProcess, remoteDllName, 0, MEM_RELEASE);
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}

		//编写shellcode在目标进程执行LoadLibrary(dll)
		CHAR LoadLibraryCode[] =
		{
			0x55,						//| push ebp |
			0x89,0xE5,					//| mov ebp,esp |
			0xFF,0x75, 0x08,			//| push dword ptr ss : [ebp + 8] |
			0xB8,0x00,0x00,0x00,0x00,	//| mov eax,0 |
			0xFF,0xD0,					//| call eax |
			0x5D,						//| pop ebp|
			0xC2,0x04,0x00				//| ret 4 |
		};

		HMODULE kernel32 = GetModuleHandleA("kernel32.dll");
		if (!kernel32)
		{
			printf("Failed to get module handle kernel32.dll\n");
			VirtualFreeEx(hProcess, remoteDllName, 0, MEM_RELEASE);
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}
		FARPROC loadLibraryAddr = GetProcAddress(kernel32, "LoadLibraryA");
		*(FARPROC*)(LoadLibraryCode + 7) = loadLibraryAddr;

		//将shellcode写入remoteFunc中
		if (!WriteProcessMemory(hProcess, remoteFunc, LoadLibraryCode, sizeof(LoadLibraryCode), NULL))
		{
			printf("Failed to write loadlibrary shellcode to target process\n");
			VirtualFreeEx(hProcess, remoteDllName, 0, MEM_RELEASE);
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}

		//创建线程执行LoadLibraryA
		HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)remoteFunc, remoteDllName, 0, NULL);
		if (!hThread)
		{
			printf("Failed to create remote thread to target process\n");
			VirtualFreeEx(hProcess, remoteDllName, 0, MEM_RELEASE);
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}

		//等待线程执行结束
		WaitForSingleObject(hThread, INFINITE);

		//接收LoadLibraryA返回地址
		DWORD hRemoteModule = 0;
		GetExitCodeThread(hThread, &hRemoteModule);
		CloseHandle(hThread);
		VirtualFreeEx(hProcess, remoteDllName, 0, MEM_RELEASE);

		if (!hRemoteModule)
		{
			printf("remote LoadLibraryA not return value\n");
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}
		
		//定位导入dll函数表
		DWORD intRva = importDesc[i].OriginalFirstThunk;
		DWORD iatRva = importDesc[i].FirstThunk;

		if (intRva == 0) intRva = iatRva;

		PIMAGE_THUNK_DATA thunkINT = (PIMAGE_THUNK_DATA)(context->imageBuffer + intRva);
		PIMAGE_THUNK_DATA thunkIAT = (PIMAGE_THUNK_DATA)(context->imageBuffer + iatRva);

		for (size_t j = 0; thunkINT[j].u1.AddressOfData; j++)
		{
			LPVOID remoteFuncName = NULL;
			DWORD ordinal = 0;

			//序号导入
			if (IMAGE_SNAP_BY_ORDINAL(thunkINT[j].u1.Ordinal))
			{
				ordinal = IMAGE_ORDINAL(thunkINT[j].u1.Ordinal);
				printf("ordinal import : -> %x", ordinal);
			}
			//名称导入
			else
			{
				PIMAGE_IMPORT_BY_NAME importByName = (PIMAGE_IMPORT_BY_NAME)(context->imageBuffer + thunkINT[j].u1.AddressOfData);
				remoteFuncName = VirtualAllocEx(hProcess, NULL, strlen(importByName->Name) + 1, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
				if (!remoteFuncName)
				{
					printf("Failed to allocate dll funcname to target process\n");
					VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
					return FALSE;
				}
				if (!WriteProcessMemory(hProcess, remoteFuncName, importByName->Name, strlen(importByName->Name) + 1, NULL))
				{
					printf("Failed to write dll funcname to target process\n");
					VirtualFreeEx(hProcess, remoteFuncName, 0, MEM_RELEASE);
					VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
					return FALSE;
				}
				printf("Name import : -> %s", importByName->Name);

			}

			//编写GetProcAddress shellcode
			CHAR getProcCode[] =
			{
				0x55,                   // push ebp
				0x8B, 0xEC,             // mov ebp, esp
				0x8B, 0x45, 0x08,       // mov eax, [ebp+8]       ; eax = params
				0xFF, 0x70, 0x04,       // push dword ptr [eax+4] ; proc name / ordinal
				0xFF, 0x30,             // push dword ptr [eax]   ; hModule
				0xB8, 0, 0, 0, 0,       // mov eax, GetProcAddress
				0xFF, 0xD0,             // call eax
				0x5D,                   // pop ebp
				0xC2, 0x04, 0x00        // ret 4
			};
			FARPROC getProcAddr = GetProcAddress(kernel32, "GetProcAddress");
			*(FARPROC*)(getProcCode + 12) = getProcAddr;

			if (!WriteProcessMemory(hProcess, remoteFunc, getProcCode, sizeof(getProcCode), NULL))
			{
				printf("Failed to write getprocess shell code to target process\n");
				VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
				return FALSE;
			}

			//定义传参结构体, 包含两个参数：模块句柄 和 函数名称\序号
			struct {
				HMODULE hModule;
				LPCSTR funcName;
			}params = {
					(HMODULE)hRemoteModule,
					IMAGE_SNAP_BY_ORDINAL(thunkINT[j].u1.Ordinal) ? (LPCSTR)(ULONG_PTR)ordinal : (LPCSTR)remoteFuncName};

			//将参数传入目标进程内存中
			LPVOID remoteParams = VirtualAllocEx(hProcess, NULL, sizeof(params), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
			if (!remoteParams)
			{
				printf("Failed to allocate remoteParams to target process\n");
				VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
				return FALSE;
			}
			if (!WriteProcessMemory(hProcess, remoteParams, (LPCVOID)&params, sizeof(params), NULL))
			{
				printf("Failed to allocate remoteParams to target process\n");
				VirtualFreeEx(hProcess, remoteParams, 0, MEM_RELEASE);
				VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
				return FALSE;
			}

			//CreateRemoteThread 执行shellcode
			HANDLE hProcThread = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)remoteFunc, remoteParams, 0, NULL);
			if (!hProcThread)
			{
				printf("Failed to CreatRemoteThread(GetProcAddress) in target process\n");
				VirtualFreeEx(hProcess, remoteParams, 0, MEM_RELEASE);
				VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
				return FALSE;
			}
			//等待线程执行完毕
			WaitForSingleObject(hProcThread, INFINITE);
			
			DWORD funcAddr = 0;//GetProcAddress返回地址
			GetExitCodeThread(hProcThread, &funcAddr);
			CloseHandle(hProcThread);
			VirtualFreeEx(hProcess, remoteParams, 0, MEM_RELEASE);
			VirtualFreeEx(hProcess, remoteFuncName, 0, MEM_RELEASE);

			if (!funcAddr)
			{
				printf("!!!!!!!!!! Failed to get GetProcAddr return function: addr: !!!!!!!!!\n");
				continue;
			}
			printf("Import Addr: -> %x\n", funcAddr);

			if (!WriteProcessMemory(hProcess,
				(BYTE*)context->remoteBaseAddress + iatRva + (j * sizeof(IMAGE_THUNK_DATA)),
				&funcAddr,
				sizeof(DWORD),
				NULL))
			{
				printf("!!!!!!!!!!! Failed to write fixed import addr to target process !!!!!!!!!!!\n");
			}
		}



	}

	VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);

	return TRUE;
}
```


### 修复重定位表

关于delta: 由于重定位表中要修复的数据原本的值data = imagebase + rva ，而我们修复的过程就是把其变为 remoteBaseAddress + rva, 其中的imagebase是写死的值(存储在ntheaders->optionalheader中), remoteBaseAddress是pe文件拉伸到内存后在内存中实际的基址。所以delta = remoteBaseAddress - imagebase，后面只需要计算data + delta = remoteBaseAddress + rva写入对应要修复的位置就行了。

```c
BOOL FixRelocationTable(HANDLE hProcess, PE_CONTEXT* context)
{
	if (!hProcess || !context) return FALSE;

	//计算实际基址和理想基址差值
	DWORD64 delta = (DWORD64)((PBYTE)context->remoteBaseAddress - context->ntHeaders->OptionalHeader.ImageBase);

	if (delta == 0)
	{
		printf("No Relacation need\n");
		return TRUE;
	}
	
	//定位重定位表
	PIMAGE_DATA_DIRECTORY relocDir = (PIMAGE_DATA_DIRECTORY)&context->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC];
	if (relocDir->Size == 0 || relocDir->VirtualAddress == 0)
	{
		printf("No Relocation Information\n");
		return TRUE;
	}
	
	PIMAGE_BASE_RELOCATION relocTable = (PIMAGE_BASE_RELOCATION)(context->imageBuffer + relocDir->VirtualAddress);
	//遍历每一个重定位表
	while (relocTable->SizeOfBlock)
	{
		DWORD blockSize = (relocTable->SizeOfBlock - sizeof(IMAGE_BASE_RELOCATION)) / sizeof(WORD);
		PWORD relocBlock = (PWORD)((PBYTE)relocTable + sizeof(IMAGE_BASE_RELOCATION));

		//遍历重定位表的每一个块
		for (size_t i = 0; i < blockSize; i++)
		{
			WORD entry = relocBlock[i];
			WORD offset = entry & 0xFFF;
			WORD type = entry >> 12;

			//匹配64位还是32位（实际本程序只适合32位）
			switch (type)
			{
			case IMAGE_REL_BASED_HIGHLOW:
			{
				//获取要修复的数据的地址，并将数据修复
				PDWORD address = (PDWORD)(context->imageBuffer + relocTable->VirtualAddress + offset);
				DWORD newValue = *address + delta;

				//将修复的数据写入目标进程中
				if (!WriteProcessMemory(
					hProcess,
					(LPVOID)((PBYTE)context->remoteBaseAddress + relocTable->VirtualAddress + offset),
					&newValue,
					sizeof(DWORD),
					NULL))
				{
					printf("Failed to write fixed relocation address to target process\n");
					return FALSE;
				}
				break;
			}
			case IMAGE_REL_BASED_DIR64:
			{
				PDWORD64 address = (PDWORD64)(context->imageBuffer + relocTable->VirtualAddress + offset);
				DWORD64 newValue = *address + delta;

				if (!WriteProcessMemory(
					hProcess,
					(LPVOID)((PBYTE)context->remoteBaseAddress + relocTable->VirtualAddress + offset),
					&newValue,
					sizeof(DWORD64),
					NULL))
				{
					printf("Failed to write fixed relocation address to target process\n");
					return FALSE;
				}
				break;
			}


			}


		}
		//定位下一个重定位表
		relocTable = (PIMAGE_BASE_RELOCATION)((PBYTE)relocTable + relocTable->SizeOfBlock);
	}


	return TRUE;
}
```


## 执行dll入口点

关于shellcode编写：shellcode的编写依赖于我们写的dll。我的dll是通过visual studio创建的动态链接库.
DllMain:
```c
BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        MessageBoxA(NULL, "Dll Inject", "Load", 0);
        break;
    case DLL_THREAD_ATTACH:
        break;
    case DLL_THREAD_DETACH:
        break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```
其中DllMain需要传入三个参数hModule,ul_reason_for_call,lpReserved。第一个是我们的基址,第二个对应switch case的值(DLL_PROCESS_ATTACH = 1), 第三个保留0。所以我们的shellcode一次push 0; push 1;push remoteBaseAddress; 然后mov eax, dllMain入口点地址, call eax。最后ret 0xc清理传入参数

```c
BOOL CallDllMain(DWORD processID, PE_CONTEXT* context)
{

	//获取函数入口点偏移
	DWORD entryPointRva = context->ntHeaders->OptionalHeader.AddressOfEntryPoint;
	if (entryPointRva == 0)
	{
		printf("Dll has no entry point\n");
		return FALSE;
	}

	//编写执行dllmain的shellcode
	CHAR callDllCode[] =
	{
		0x68, 0x00, 0x00, 0x00, 0x00,		//push
		0x68, 0x00, 0x00, 0x00, 0x00,		//push
		0x68, 0x00, 0x00, 0x00, 0x00,		//push
		0xB8, 0x00, 0x00, 0x00, 0x00,		//mov eax, dllentrypoint
		0xFF, 0xD0,							//call eax
		0xC2, 0x0C, 0x00                    //retn 12
	};
	*(DWORD*)(callDllCode + 6) = 1;
	*(DWORD*)(callDllCode + 11) = (DWORD)context->remoteBaseAddress;
	*(DWORD*)(callDllCode + 16) = (DWORD)((PBYTE)context->remoteBaseAddress + entryPointRva);

	//获取进程句柄
	HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, processID);
	if (hProcess == NULL)
	{
		printf("Faile to open process in CallDllMain\n");
		return FALSE;
	}

	//为shellcode在目标进程中分配内存并写入
	LPVOID remoteFunc = VirtualAllocEx(hProcess, NULL, sizeof(callDllCode), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
	if (!remoteFunc)
	{
		printf("Failed to allocate callDllCode memory in target process\n");
		CloseHandle(hProcess);
		return FALSE;
	}

	if (!WriteProcessMemory(hProcess, remoteFunc, callDllCode, sizeof(callDllCode), NULL))
	{
		printf("Failed to write callDllCode to target process memory\n");
		VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
		CloseHandle(hProcess);
		return FALSE;
	}

	//CreateRemoteThread执行dllmain
	HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)remoteFunc, NULL, 0, NULL);
	if (!hThread)
	{
		printf("Failed to create remote thread to call dll in target process\n");
		VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
		CloseHandle(hProcess);
		return FALSE;
	}

	WaitForSingleObject(hThread, INFINITE);
	
	DWORD exitCode = 0;
	GetExitCodeThread(hThread, &exitCode);
	CloseHandle(hThread);

	VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
	CloseHandle(hProcess);

	printf("Call Dll return: %d\n", exitCode);


	return TRUE;
}
```


## dll编写

visual studio新建解决方案选择动态链接库

```c
#include "pch.h"
#include <Windows.h>

BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        MessageBoxA(NULL, "Dll Inject", "Load", 0);
        break;
    case DLL_THREAD_ATTACH:
        break;
    case DLL_THREAD_DETACH:
        break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

# 完整脚本

## exe

```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <Windows.h>
#include <stdlib.h>
#include <string.h>
#include <TlHelp32.h>
#include <tchar.h>


//====================== 结构体定义 =========================
typedef struct
{
	HANDLE hFile;
	DWORD fileSize;
	PBYTE fileBuffer;
	PBYTE imageBuffer;
	DWORD imageSize;

	PIMAGE_DOS_HEADER dosHeader;
	PIMAGE_NT_HEADERS ntHeaders;
	PIMAGE_SECTION_HEADER sectionHeader;

	LPVOID remoteBaseAddress;

}PE_CONTEXT;

//====================== 结构体定义 =========================





//====================== 函数声明 ==========================
DWORD FindProcessIdByName(LPCTSTR targetProcessName);
BOOL LoadPeFile(char* dllPath, PE_CONTEXT* peContext);
VOID UnLoadPeFile(PE_CONTEXT* context);
BOOL InjectDllToProcess(DWORD processID, PE_CONTEXT* context);
BOOL FixImportTable(HANDLE hProcess, PE_CONTEXT* context);
BOOL FixRelocationTable(HANDLE hProcess, PE_CONTEXT* context);
BOOL CallDllMain(DWORD processID, PE_CONTEXT* context);
//====================== 函数声明 ==========================






//====================== 主函数功能实现 ========================
int main()
{
	printf("========= DLL Inject ==========\n");

	//获取用户输入目标进程id,名字 得到目标进程id。
	DWORD targetPid = 0;
	CHAR targetProcessName[MAX_PATH] = { 0 };
	CHAR input[255] = { 0 };

	printf("Input Target ID/Name\n");
	printf("1. Process ID\n");
	printf("2. Process Name\n");
	printf("Enter choice(1 / 2): ");

	fgets(input, sizeof(input), stdin);
	int choice = atoi(input);

	if (choice == 1)
	{
		printf("Please input target process ID: ");
		fgets(input, sizeof(input), stdin);
		targetPid = atoi(input);
		if (targetPid == 0)
		{
			printf("Invalid process ID\n");
			return 0;
		}
	}
	else if (choice == 2)
	{
		printf("Please input target process name: ");
		fgets(input, sizeof(input), stdin);
		size_t len = strlen(input);
		if (len > 0 && input[len - 1] == '\n')
		{
			input[len - 1] = '\0';
		}
		strcpy(targetProcessName, input);

		targetPid = FindProcessIdByName((LPCTSTR)targetProcessName);//参数转化为LPCTSTR类型,适配项目的字符集设置

		if (targetPid == 0)
		{
			printf("find process id by name: %s wrong\n", targetProcessName);
			return 0;
		}

	}
	else
	{
		printf("wrong choice\n");
		system("pause");
		return 0;
	}


	//获取dll文件
	char dllPath[MAX_PATH] = { 0 };
	printf("Please input dll file path: ");
	fgets(input, sizeof(input), stdin);
	size_t len = strlen(input);
	if (len > 0 && input[len - 1] == '\n')
	{
		input[len - 1] = '\0';
	}

	strcpy(dllPath, input);

	if (dllPath[0] == '\0')
	{
		printf("Invalid dll path\n");
		return 0;
	}

	system("cls");
	printf("target PID: %d \n", targetPid);
	printf("Dll Path: %s\n", dllPath);



	PE_CONTEXT peContext;

	//加载dll文件
	if (!LoadPeFile(dllPath, &peContext))
	{
		return 0;
	}


	//向目标进程注入dll
	if (!InjectDllToProcess(targetPid, &peContext))
	{
		UnLoadPeFile(&peContext);
		return 0;
	}


	//执行dllMain
	if (!CallDllMain(targetPid, &peContext))
	{
		printf("Failed to call DllMain\n");
		return 0;
	}

	printf("Call DllMain successfully\n");


	return 0;
}


//====================== 主函数功能实现 ========================
DWORD FindProcessIdByName(LPCTSTR targetProcessName)
{
	HANDLE hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
	if (hSnapShot == INVALID_HANDLE_VALUE)
	{
		printf("Failed to create process snapshot\n");
		return 0;
	}
	PROCESSENTRY32 pe32 = { 0 };
	pe32.dwSize = sizeof(PROCESSENTRY32);

	if (!Process32First(hSnapShot, &pe32))
	{
		printf("Failed to get first process\n");
		CloseHandle(hSnapShot);
		return 0;
	}

	DWORD processId = 0;
	do
	{
		if (_stricmp(pe32.szExeFile, targetProcessName) == 0)
		{
			processId = pe32.th32ProcessID;
			break;
		}
	} while (Process32Next(hSnapShot, &pe32));

	CloseHandle(hSnapShot);
	return processId;
}

BOOL LoadPeFile(char* dllPath, PE_CONTEXT* peContext)
{
	if (!dllPath || !peContext) return FALSE;
	//初始化peContext内容
	memset(peContext, 0, sizeof(peContext));


	//打开文件
	peContext->hFile = CreateFileA(
		dllPath,
		GENERIC_READ,
		FILE_SHARE_READ,
		NULL,
		OPEN_EXISTING,
		FILE_ATTRIBUTE_NORMAL,
		NULL);

	if (peContext->hFile == INVALID_HANDLE_VALUE)
	{
		printf("Failed to open dll file: %s\n", dllPath);
		return FALSE;
	}

	//获取文件大小
	peContext->fileSize = GetFileSize(peContext->hFile, NULL);

	if (peContext->fileSize == INVALID_FILE_SIZE)
	{
		printf("Failed to get file size: %s", dllPath);
		CloseHandle(peContext->hFile);
		peContext->hFile = INVALID_HANDLE_VALUE;
		return FALSE;
	}

	//将文件映射到内存中
	
	peContext->fileBuffer = (PBYTE)VirtualAlloc(
		NULL,
		peContext->fileSize,
		MEM_COMMIT | MEM_RESERVE,
		PAGE_READWRITE);
	
	if (!peContext->fileBuffer)
	{
		printf("Failed to allocate memory for file: %s\n", dllPath);
		CloseHandle(peContext->hFile);
		peContext->hFile = INVALID_HANDLE_VALUE;
		return FALSE;
	}

	DWORD byteRead = 0;
	if (!ReadFile(
		peContext->hFile,
		peContext->fileBuffer,
		peContext->fileSize,
		&byteRead,
		NULL) || byteRead != peContext->fileSize)
	{
		printf("Failed to read file: %s\n", dllPath);
		VirtualFree(peContext->fileBuffer, 0, MEM_RELEASE);
		peContext->fileBuffer = NULL;
		CloseHandle(peContext->hFile);
		peContext->hFile = INVALID_HANDLE_VALUE;
		return FALSE;
	}

	CloseHandle(peContext->hFile);
	peContext->hFile = INVALID_HANDLE_VALUE;

	//解析dos,nt,section头
	peContext->dosHeader = (PIMAGE_DOS_HEADER)(peContext->fileBuffer);
	if (peContext->dosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		printf("Invalid dos signature\n");
		UnLoadPeFile(peContext);
		return FALSE;
	}

	peContext->ntHeaders = (PIMAGE_NT_HEADERS)(peContext->fileBuffer + peContext->dosHeader->e_lfanew);
	if (peContext->ntHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		printf("Invalid nt signature\n");
		UnLoadPeFile(peContext);
		return FALSE;
	}
	//检测是否为dll文件
	if (!(peContext->ntHeaders->FileHeader.Characteristics & IMAGE_FILE_DLL))
	{
		printf("Input file is not a dll file\n");
		UnLoadPeFile(peContext);
		return FALSE;
	}

	peContext->sectionHeader = IMAGE_FIRST_SECTION(peContext->ntHeaders);

	//将dll文件在内存中拉伸
	peContext->imageBuffer = (PBYTE)VirtualAlloc(
		NULL,
		peContext->ntHeaders->OptionalHeader.SizeOfImage,
		MEM_COMMIT | MEM_RESERVE,
		PAGE_READWRITE);


	if (!peContext->imageBuffer)
	{
		printf("Failed to allocate image buffer\n");
		UnLoadPeFile(peContext);
		return FALSE;
	}
	memset(peContext->imageBuffer, 0, peContext->ntHeaders->OptionalHeader.SizeOfImage);

	//copy 头
	memcpy(peContext->imageBuffer, peContext->fileBuffer, peContext->ntHeaders->OptionalHeader.SizeOfHeaders);
	
	//copy 节区
	for (size_t i = 0; i < peContext->ntHeaders->FileHeader.NumberOfSections; i++)
	{
		if (peContext->sectionHeader[i].SizeOfRawData == 0)
		{
			continue;
		}

		memcpy(
			peContext->imageBuffer + peContext->sectionHeader[i].VirtualAddress,
			peContext->fileBuffer + peContext->sectionHeader[i].PointerToRawData,
			min(peContext->sectionHeader[i].SizeOfRawData, peContext->sectionHeader[i].Misc.VirtualSize)
			);
		
	}

	peContext->imageSize = peContext->ntHeaders->OptionalHeader.SizeOfImage;

	return TRUE;
}

VOID UnLoadPeFile(PE_CONTEXT* context)
{
	if (!context) return;

	if (context->fileBuffer)
	{
		VirtualFree(context->fileBuffer, 0, MEM_RELEASE);
		context->fileBuffer = NULL;
	}

	if (context->hFile && context->hFile != INVALID_HANDLE_VALUE)
	{
		CloseHandle(context->hFile);
		context->hFile = INVALID_HANDLE_VALUE;
	}

	if (context->imageBuffer)
	{
		VirtualFree(context->imageBuffer, 0, MEM_RELEASE);
		context->imageBuffer = NULL;
	}

	return;
}

BOOL InjectDllToProcess(DWORD processID, PE_CONTEXT* context)
{
	if (!processID || !context) return FALSE;
	
	//向目标进程写入dll
	HANDLE hProcess = OpenProcess(
		PROCESS_ALL_ACCESS,
		FALSE,
		processID);
	if (hProcess == NULL)
	{
		printf("Failed to open process: %d", processID);
		return FALSE;
	}

	LPVOID remoteBaseAddress = VirtualAllocEx(
		hProcess,
		NULL,
		context->imageSize,
		MEM_COMMIT | MEM_RESERVE,
		PAGE_EXECUTE_READWRITE);
	
	if (remoteBaseAddress == NULL)
	{
		printf("Failed to allocate memory in target process\n");
		CloseHandle(hProcess);
		return FALSE;
	}

	context->remoteBaseAddress = remoteBaseAddress;
	printf("Memory allocated in target process: %p\n", remoteBaseAddress);

	if (!WriteProcessMemory(
		hProcess,
		remoteBaseAddress,
		context->imageBuffer,
		context->imageSize,
		NULL))
	{
		printf("Failed to write memory to target process\n");
		VirtualFreeEx(hProcess, remoteBaseAddress, 0, MEM_RELEASE);
		CloseHandle(hProcess);
		return FALSE;
	}
	
	printf("Image written to target process successfully\n");

	//在目标进程内存中修复dll导入表
	if (!FixImportTable(hProcess, context))
	{
		printf("Failed to fix import table\n");
		VirtualFreeEx(hProcess, remoteBaseAddress, 0, MEM_RELEASE);
		CloseHandle(hProcess);
		return FALSE;
	}
	printf("\nFix import table successfully\n");

	//在目标进程内存中修复dll重定位表
	if (!FixRelocationTable(hProcess, context))
	{
		printf("Failed to fix relocation table\n");
		VirtualFreeEx(hProcess, remoteBaseAddress, 0, MEM_RELEASE);
		CloseHandle(hProcess);
		return FALSE;
	}
	printf("\nfix relocation table successfully\n");

	CloseHandle(hProcess);

	return TRUE;
}

BOOL FixImportTable(HANDLE hProcess, PE_CONTEXT* context)
{
	if (!hProcess || !context) return FALSE;

	//定位导入表数据目录项
	IMAGE_DATA_DIRECTORY importDir = context->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT];
	if (importDir.Size == 0 || importDir.VirtualAddress == 0)
	{
		printf("Not found import table\n");
		return FALSE;
	}

	PIMAGE_IMPORT_DESCRIPTOR importDesc = (PIMAGE_IMPORT_DESCRIPTOR)(context->imageBuffer + importDir.VirtualAddress);
	
	//向目标进程分配shellcode内存
	LPVOID remoteFunc = VirtualAllocEx(
		hProcess,
		NULL,
		0x1000,
		MEM_COMMIT | MEM_RESERVE,
		PAGE_EXECUTE_READWRITE);
	if (!remoteFunc)
	{
		printf("Failed to allocate remotefunc in target process\n");
		return FALSE;
	}



	for (size_t i = 0; importDesc[i].Name; i++)
	{
		//获取当前导入dll名称
		CHAR* dllName = (CHAR*)(context->imageBuffer + importDesc[i].Name);
		printf("========  Import Dll: %s ========\n", dllName);

		//将dll名称写入目标进程中
		LPVOID remoteDllName = VirtualAllocEx(
			hProcess,
			NULL,
			strlen(dllName) + 1,
			MEM_COMMIT | MEM_RESERVE,
			PAGE_READWRITE);
		if (!remoteDllName)
		{
			printf("Failed to allocate memory for dll name in target process\n");
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}
		if (!WriteProcessMemory(
			hProcess,
			remoteDllName,
			dllName,
			strlen(dllName) + 1,
			NULL))
		{
			printf("Failed to write memory for dll name in target process\n");
			VirtualFreeEx(hProcess, remoteDllName, 0, MEM_RELEASE);
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}

		//编写shellcode在目标进程执行LoadLibrary(dll)
		CHAR LoadLibraryCode[] =
		{
			0x55,						//| push ebp |
			0x89,0xE5,					//| mov ebp,esp |
			0xFF,0x75, 0x08,			//| push dword ptr ss : [ebp + 8] |
			0xB8,0x00,0x00,0x00,0x00,	//| mov eax,0 |
			0xFF,0xD0,					//| call eax |
			0x5D,						//| pop ebp|
			0xC2,0x04,0x00				//| ret 4 |
		};

		HMODULE kernel32 = GetModuleHandleA("kernel32.dll");
		if (!kernel32)
		{
			printf("Failed to get module handle kernel32.dll\n");
			VirtualFreeEx(hProcess, remoteDllName, 0, MEM_RELEASE);
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}
		FARPROC loadLibraryAddr = GetProcAddress(kernel32, "LoadLibraryA");
		*(FARPROC*)(LoadLibraryCode + 7) = loadLibraryAddr;

		//将shellcode写入remoteFunc中
		if (!WriteProcessMemory(hProcess, remoteFunc, LoadLibraryCode, sizeof(LoadLibraryCode), NULL))
		{
			printf("Failed to write loadlibrary shellcode to target process\n");
			VirtualFreeEx(hProcess, remoteDllName, 0, MEM_RELEASE);
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}

		//创建线程执行LoadLibraryA
		HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)remoteFunc, remoteDllName, 0, NULL);
		if (!hThread)
		{
			printf("Failed to create remote thread to target process\n");
			VirtualFreeEx(hProcess, remoteDllName, 0, MEM_RELEASE);
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}

		//等待线程执行结束
		WaitForSingleObject(hThread, INFINITE);

		//接收LoadLibraryA返回地址
		DWORD hRemoteModule = 0;
		GetExitCodeThread(hThread, &hRemoteModule);
		CloseHandle(hThread);
		VirtualFreeEx(hProcess, remoteDllName, 0, MEM_RELEASE);

		if (!hRemoteModule)
		{
			printf("remote LoadLibraryA not return value\n");
			VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
			return FALSE;
		}
		
		//定位导入dll函数表
		DWORD intRva = importDesc[i].OriginalFirstThunk;
		DWORD iatRva = importDesc[i].FirstThunk;

		if (intRva == 0) intRva = iatRva;

		PIMAGE_THUNK_DATA thunkINT = (PIMAGE_THUNK_DATA)(context->imageBuffer + intRva);
		PIMAGE_THUNK_DATA thunkIAT = (PIMAGE_THUNK_DATA)(context->imageBuffer + iatRva);

		for (size_t j = 0; thunkINT[j].u1.AddressOfData; j++)
		{
			LPVOID remoteFuncName = NULL;
			DWORD ordinal = 0;

			//序号导入
			if (IMAGE_SNAP_BY_ORDINAL(thunkINT[j].u1.Ordinal))
			{
				ordinal = IMAGE_ORDINAL(thunkINT[j].u1.Ordinal);
				printf("ordinal import : -> %x", ordinal);
			}
			//名称导入
			else
			{
				PIMAGE_IMPORT_BY_NAME importByName = (PIMAGE_IMPORT_BY_NAME)(context->imageBuffer + thunkINT[j].u1.AddressOfData);
				remoteFuncName = VirtualAllocEx(hProcess, NULL, strlen(importByName->Name) + 1, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
				if (!remoteFuncName)
				{
					printf("Failed to allocate dll funcname to target process\n");
					VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
					return FALSE;
				}
				if (!WriteProcessMemory(hProcess, remoteFuncName, importByName->Name, strlen(importByName->Name) + 1, NULL))
				{
					printf("Failed to write dll funcname to target process\n");
					VirtualFreeEx(hProcess, remoteFuncName, 0, MEM_RELEASE);
					VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
					return FALSE;
				}
				printf("Name import : -> %s", importByName->Name);

			}

			//编写GetProcAddress shellcode
			CHAR getProcCode[] =
			{
				0x55,                   // push ebp
				0x8B, 0xEC,             // mov ebp, esp
				0x8B, 0x45, 0x08,       // mov eax, [ebp+8]       ; eax = params
				0xFF, 0x70, 0x04,       // push dword ptr [eax+4] ; proc name / ordinal
				0xFF, 0x30,             // push dword ptr [eax]   ; hModule
				0xB8, 0, 0, 0, 0,       // mov eax, GetProcAddress
				0xFF, 0xD0,             // call eax
				0x5D,                   // pop ebp
				0xC2, 0x04, 0x00        // ret 4
			};
			FARPROC getProcAddr = GetProcAddress(kernel32, "GetProcAddress");
			*(FARPROC*)(getProcCode + 12) = getProcAddr;

			if (!WriteProcessMemory(hProcess, remoteFunc, getProcCode, sizeof(getProcCode), NULL))
			{
				printf("Failed to write getprocess shell code to target process\n");
				VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
				return FALSE;
			}

			//定义传参结构体, 包含两个参数：模块句柄 和 函数名称\序号
			struct {
				HMODULE hModule;
				LPCSTR funcName;
			}params = {
					(HMODULE)hRemoteModule,
					IMAGE_SNAP_BY_ORDINAL(thunkINT[j].u1.Ordinal) ? (LPCSTR)(ULONG_PTR)ordinal : (LPCSTR)remoteFuncName};

			//将参数传入目标进程内存中
			LPVOID remoteParams = VirtualAllocEx(hProcess, NULL, sizeof(params), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
			if (!remoteParams)
			{
				printf("Failed to allocate remoteParams to target process\n");
				VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
				return FALSE;
			}
			if (!WriteProcessMemory(hProcess, remoteParams, (LPCVOID)&params, sizeof(params), NULL))
			{
				printf("Failed to allocate remoteParams to target process\n");
				VirtualFreeEx(hProcess, remoteParams, 0, MEM_RELEASE);
				VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
				return FALSE;
			}

			//CreateRemoteThread 执行shellcode
			HANDLE hProcThread = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)remoteFunc, remoteParams, 0, NULL);
			if (!hProcThread)
			{
				printf("Failed to CreatRemoteThread(GetProcAddress) in target process\n");
				VirtualFreeEx(hProcess, remoteParams, 0, MEM_RELEASE);
				VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
				return FALSE;
			}
			//等待线程执行完毕
			WaitForSingleObject(hProcThread, INFINITE);
			
			DWORD funcAddr = 0;//GetProcAddress返回地址
			GetExitCodeThread(hProcThread, &funcAddr);
			CloseHandle(hProcThread);
			VirtualFreeEx(hProcess, remoteParams, 0, MEM_RELEASE);
			VirtualFreeEx(hProcess, remoteFuncName, 0, MEM_RELEASE);

			if (!funcAddr)
			{
				printf("!!!!!!!!!! Failed to get GetProcAddr return function: addr: !!!!!!!!!\n");
				continue;
			}
			printf("Import Addr: -> %x\n", funcAddr);

			if (!WriteProcessMemory(hProcess,
				(BYTE*)context->remoteBaseAddress + iatRva + (j * sizeof(IMAGE_THUNK_DATA)),
				&funcAddr,
				sizeof(DWORD),
				NULL))
			{
				printf("!!!!!!!!!!! Failed to write fixed import addr to target process !!!!!!!!!!!\n");
			}
		}



	}

	VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);

	return TRUE;
}

BOOL FixRelocationTable(HANDLE hProcess, PE_CONTEXT* context)
{
	if (!hProcess || !context) return FALSE;

	//计算实际基址和理想基址差值
	DWORD64 delta = (DWORD64)((PBYTE)context->remoteBaseAddress - context->ntHeaders->OptionalHeader.ImageBase);

	if (delta == 0)
	{
		printf("No Relacation need\n");
		return TRUE;
	}
	
	//定位重定位表
	PIMAGE_DATA_DIRECTORY relocDir = (PIMAGE_DATA_DIRECTORY)&context->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC];
	if (relocDir->Size == 0 || relocDir->VirtualAddress == 0)
	{
		printf("No Relocation Information\n");
		return TRUE;
	}
	
	PIMAGE_BASE_RELOCATION relocTable = (PIMAGE_BASE_RELOCATION)(context->imageBuffer + relocDir->VirtualAddress);
	//遍历每一个重定位表
	while (relocTable->SizeOfBlock)
	{
		DWORD blockSize = (relocTable->SizeOfBlock - sizeof(IMAGE_BASE_RELOCATION)) / sizeof(WORD);
		PWORD relocBlock = (PWORD)((PBYTE)relocTable + sizeof(IMAGE_BASE_RELOCATION));

		//遍历重定位表的每一个块
		for (size_t i = 0; i < blockSize; i++)
		{
			WORD entry = relocBlock[i];
			WORD offset = entry & 0xFFF;
			WORD type = entry >> 12;

			//匹配64位还是32位（实际本程序只适合32位）
			switch (type)
			{
			case IMAGE_REL_BASED_HIGHLOW:
			{
				//获取要修复的数据的地址，并将数据修复
				PDWORD address = (PDWORD)(context->imageBuffer + relocTable->VirtualAddress + offset);
				DWORD newValue = *address + delta;

				//将修复的数据写入目标进程中
				if (!WriteProcessMemory(
					hProcess,
					(LPVOID)((PBYTE)context->remoteBaseAddress + relocTable->VirtualAddress + offset),
					&newValue,
					sizeof(DWORD),
					NULL))
				{
					printf("Failed to write fixed relocation address to target process\n");
					return FALSE;
				}
				break;
			}
			case IMAGE_REL_BASED_DIR64:
			{
				PDWORD64 address = (PDWORD64)(context->imageBuffer + relocTable->VirtualAddress + offset);
				DWORD64 newValue = *address + delta;

				if (!WriteProcessMemory(
					hProcess,
					(LPVOID)((PBYTE)context->remoteBaseAddress + relocTable->VirtualAddress + offset),
					&newValue,
					sizeof(DWORD64),
					NULL))
				{
					printf("Failed to write fixed relocation address to target process\n");
					return FALSE;
				}
				break;
			}


			}


		}
		//定位下一个重定位表
		relocTable = (PIMAGE_BASE_RELOCATION)((PBYTE)relocTable + relocTable->SizeOfBlock);
	}


	return TRUE;
}

BOOL CallDllMain(DWORD processID, PE_CONTEXT* context)
{

	//获取函数入口点偏移
	DWORD entryPointRva = context->ntHeaders->OptionalHeader.AddressOfEntryPoint;
	if (entryPointRva == 0)
	{
		printf("Dll has no entry point\n");
		return FALSE;
	}

	//编写执行dllmain的shellcode
	CHAR callDllCode[] =
	{
		0x68, 0x00, 0x00, 0x00, 0x00,		//push
		0x68, 0x00, 0x00, 0x00, 0x00,		//push
		0x68, 0x00, 0x00, 0x00, 0x00,		//push
		0xB8, 0x00, 0x00, 0x00, 0x00,		//mov eax, dllentrypoint
		0xFF, 0xD0,							//call eax
		0xC2, 0x0C, 0x00
	};
	*(DWORD*)(callDllCode + 6) = 1;
	*(DWORD*)(callDllCode + 11) = (DWORD)context->remoteBaseAddress;
	*(DWORD*)(callDllCode + 16) = (DWORD)((PBYTE)context->remoteBaseAddress + entryPointRva);

	//获取进程句柄
	HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, processID);
	if (hProcess == NULL)
	{
		printf("Faile to open process in CallDllMain\n");
		return FALSE;
	}

	//为shellcode在目标进程中分配内存并写入
	LPVOID remoteFunc = VirtualAllocEx(hProcess, NULL, sizeof(callDllCode), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
	if (!remoteFunc)
	{
		printf("Failed to allocate callDllCode memory in target process\n");
		CloseHandle(hProcess);
		return FALSE;
	}

	if (!WriteProcessMemory(hProcess, remoteFunc, callDllCode, sizeof(callDllCode), NULL))
	{
		printf("Failed to write callDllCode to target process memory\n");
		VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
		CloseHandle(hProcess);
		return FALSE;
	}

	//CreateRemoteThread执行dllmain
	HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)remoteFunc, NULL, 0, NULL);
	if (!hThread)
	{
		printf("Failed to create remote thread to call dll in target process\n");
		VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
		CloseHandle(hProcess);
		return FALSE;
	}

	WaitForSingleObject(hThread, INFINITE);
	
	DWORD exitCode = 0;
	GetExitCodeThread(hThread, &exitCode);
	CloseHandle(hThread);

	VirtualFreeEx(hProcess, remoteFunc, 0, MEM_RELEASE);
	CloseHandle(hProcess);

	printf("Call Dll return: %d\n", exitCode);


	return TRUE;
}

```


## dll
```c
// dllmain.cpp : 定义 DLL 应用程序的入口点。
#include "pch.h"
#include <Windows.h>

BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        MessageBoxA(NULL, "Dll Inject", "Load", 0);
        break;
    case DLL_THREAD_ATTACH:
        break;
    case DLL_THREAD_DETACH:
        break;
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```



