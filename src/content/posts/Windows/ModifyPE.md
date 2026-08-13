---
title: PEStudy_修改PE文件
published: 2026-08-03
description: 用代码修改PE文件内容
image: ./cover.jpg
tags: [逆向]
category: Windows逆向
draft: false
---

修改PE文件的内容:
1. 删除DOSStub
2. 增加节区
3. 合并所有节区
4. 扩大节区
5. 移动导出表(完全移动)
6. 移动导入表(不完全移动)
7. 移动重定位表(不完全移动)





#  全局操作前置须知（所有操作共享）
1. **校验和（CheckSum）**：任何修改后，必须将 `NT_HEADERS->OptionalHeader.CheckSum` 置为 `0`（除非你是签名工具）。
2. **数字签名（Security Directory）**：如果文件有签名（Overlay 末尾），修改文件大小或移动节区时，必须修正 `Security Directory` 的偏移（它是文件偏移 FOA，不是 RVA）或直接移除该目录项。
3. **指针重建**：使用 `realloc` 或重新分配 `malloc` 后，必须立即重置 `pe->dosHeader`、`pe->ntHeaders`、`pe->sectionHeader` 指针。



# 1️. 删除 DOS Stub
**目标**：消除 DOS Stub 占用的空间，将 NT 头和节表整体上移，减小文件体积。

**思路**：
1. **计算新偏移**：新的 `e_lfanew` 通常设定为 `sizeof(IMAGE_DOS_HEADER)`（即 `0x40`），因为 DOS Header 之后就是 Stub 区域。
2. **内存搬运**：将 `pe->buffer + 旧 e_lfanew` 处的整个 NT 头（含节表）**整体拷贝**到 `pe->buffer + 新 e_lfanew` 处。
3. **修正 `e_lfanew`**：`pe->dosHeader->e_lfanew = 新偏移`。
4. **保留标志性字符串（可选）**：若担心某些 Loader 校验 `"This program cannot be run in DOS mode"`，则**不要覆盖该字符串**。操作方法：将 NT 头移动到该字符串结尾之后（如 `0x80` 处），并设置 `e_lfanew = 0x80`，保留中间的原始字节。

---


# 2️.增加新节区
**目标**：在文件末尾追加一个新节区（如 `.newsec`），用于存放自定义代码或数据。

**思路**：
1. **计算对齐大小**：
   - 文件大小（Raw）：`rawSize = ALIGN_UP(自定义大小, FileAlignment)`
   - 内存大小（Virtual）：`virtualSize = 自定义大小`（注意：`VirtualSize` **不需要**按 `SectionAlignment` 对齐，但 `SizeOfImage` 需要）
2. **计算偏移**：
   - **内存 RVA**：`新RVA = ALIGN_UP( 旧SizeOfImage, SectionAlignment )`（利用映像末尾的空洞）
   - **文件 FOA**：`新FOA = ALIGN_UP( 原文件大小, FileAlignment )`（直接接在文件尾）
3. **写入节表**：在 `sectionHeader[旧数量]` 处填充名称、RVA、FOA、属性。
4. **更新字段**：`NumberOfSections += 1`，`SizeOfImage = 新RVA + ALIGN_UP(virtualSize, SectionAlignment)`。
5. **物理扩展**：使用 `realloc` 扩展 `pe->buffer` 至 `新FOA + rawSize`，并将新节区的数据（如自定义 Shellcode）写入该偏移处。

---

# 3️. 合并所有节区
**目标**：将所有散落的 `.text`、`.data`、`.rsrc` 等物理数据和内存映射，强行塞进第一个节区，使 `NumberOfSections = 1`。

**思路**：
1. **锚定起点**：以 `sectionHeader[0]` 的 `VirtualAddress` 和 `PointerToRawData` 为基准。
2. **计算总跨度**：
   - 内存终点：`maxRvaEnd = MAX( sec[i].VirtualAddress + MAX(VirtualSize, SizeOfRawData) )`
   - 文件终点：`maxRawEnd = baseRaw + (maxRvaEnd - baseRva)`（紧凑排列）
3. **物理重排**：申请新缓冲区，先拷贝 PE 头，然后**循环搬运**每个节区的 `SizeOfRawData` 到新缓冲区中的 `baseRaw + (sec[i].VirtualAddress - baseRva)` 位置。
4. **Overlay 保留**：记录原文件末尾 `oldRawEnd` 之后的附加数据，搬运到新文件末尾。
5. **收尾**：
   - `sectionHeader[0].VirtualSize = maxRvaEnd - baseRva`
   - `sectionHeader[0].SizeOfRawData = ALIGN_UP( maxRawEnd - baseRaw, FileAlignment )`
   - `SizeOfImage = ALIGN_UP( maxRvaEnd, SectionAlignment )`
   - 将后续所有节表项清零，`NumberOfSections = 1`。
   - **合并属性**：新节区属性 = 所有原节区属性的按位或，并强制加上 `READ | WRITE | EXECUTE`。

---

# 4️. 扩大最后一个节区（内存扩展 vs 文件扩展）
**目标**：为最后一个节区增加 `additionalSize` 空间。

**思路**：
- **情况 A（仅扩内存）**：当 `新VirtualSize <= 旧SizeOfRawData` 时（即文件对齐留出的冗余空间足够），**只需修改** `VirtualSize` 和 `SizeOfImage`，无需改动文件物理大小。
- **情况 B（扩内存+扩文件）**：当 `新VirtualSize > 旧SizeOfRawData` 时：
  1. 计算新 RawSize：`ALIGN_UP(新VirtualSize, FileAlignment)`，算出 `extraBytes = 新RawSize - 旧RawSize`。
  2. **处理 Overlay**（关键！）：如果文件末尾有附加数据（Overlay），不能直接 `realloc` 覆盖它。必须先将 Overlay 数据备份，`realloc` 扩展文件后，再将 Overlay 拷贝到**新文件末尾**（紧接在新 RawSize 之后），并更新 `Security Directory` 的偏移（若存在）。
  3. 更新 `SizeOfImage = ALIGN_UP( 新VirtualSize, SectionAlignment )`。

---

# 5️. 移动导出表（完全移动）
**目标**：彻底删除旧的导出表目录，将导出表重建到一个全新的节区中，实现“逻辑与物理的完全搬迁”。

**思路**：
1. **解析旧表**：读取旧 `IMAGE_EXPORT_DIRECTORY`，获取 `NumberOfFunctions`、`NumberOfNames` 等。
2. **计算新节区大小**：
   - `总大小 = sizeof(IMAGE_EXPORT_DIRECTORY)`（头结构）
   - `+ 1`（DLL 名字符串长度，对齐）
   - `+ NumberOfFunctions * 4`（函数地址数组 RVA）
   - `+ NumberOfNames * 4`（函数名称数组 RVA）
   - `+ NumberOfNames * 2`（序号数组 WORD）
   - `+ 所有函数名称字符串长度之和`
3. **写入新数据**：
   - 将导出目录头写入新节区起始。
   - 将 DLL 名称字符串紧接其后。
   - 紧接着放置 **函数地址数组**、**函数名称 RVA 数组**、**序号数组**，以及名称字符串本体。
4. **关键修正——RVA 重定位**：
   - 旧表中的 `AddressOfFunctions`、`AddressOfNames`、`AddressOfNameOrdinals` 以及 `Name` 字段存储的都是 **RVA**。
   - 新表中，这些字段的 RVA 必须更新为：`新节区基址(RVA) + 相对于新节区起始的偏移`。
   - 同时，数组中的每一项（函数地址、函数名称 RVA）也必须加上新节区基址的偏移值（因为它们是相对于模块基址的 RVA）。
5. **更新目录项**：`DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress = 新节区RVA`，Size 更新为总大小。

---

# 6️.移动导入表 & 重定位表（不完全移动 / 批量复制）
**目标**：不解析内部复杂的链表结构，仅将原始的 `.idata` 和 `.reloc` 节区二进制数据，原封不动地“剪贴”到一个新节区中。

**思路**：
1. **定位与备份**：
   - 获取 `DataDirectory[IMPORT]` 指向的旧 RVA，通过 RVA->FOA 转换算出旧文件偏移，记录其 `Size`。
   - 对于重定位表同理。
2. **分配新空间**：新增一个节区（或复用同一个新增节区），大小为 `导入表大小 + 重定位表大小`（按 FileAlignment 对齐）。
3. **直接复制字节**：
   - `memcpy(新缓冲区 + 新FOA, 旧缓冲区 + 旧FOA, 旧Size)`。
   - **为什么这样可行？** 因为导入表（`IMAGE_IMPORT_DESCRIPTOR`）和重定位表（`IMAGE_BASE_RELOCATION`）内部存储的 **RVA 指针** 是基于模块基址的绝对偏移。如果我们把整个表（包括其指向的字符串和 IAT 数组）整体搬到另一个地址，**只要内部指针指向的地址还在这个新节区的范围内，且相对新节区起始的偏移不变**，Windows 加载器就能正确解析。（因为 `RVA - 新节区基址` 与 `旧RVA - 旧节区基址` 的差值保持不变）。
4. **更新目录项**：将导入表和重定位表的 `VirtualAddress` 改为新节区的 RVA，`Size` 填入复制的大小。
5. **注意**：如果导入表引用了**其他节区**（如 `.text` 中的 IAT 占位符），这种“不完全移动”会丢失引用，因此必须确保移动前，导入表数据和代码段中的 IAT 占位符在同一个节区内（通常确实如此）。若考虑绝对健壮，完全移动需要遍历 Thunk 并修正，但这属于“完全移动”范畴，你定义的“不完全移动”用直接复制即可。

---

# else

###  通用工具函数（必须实现）
```c
// 宏：按指定粒度向上对齐
#define ALIGN_UP(value, alignment) (((value) + (alignment) - 1) & ~((alignment) - 1))

// 转换函数：RVA 转文件偏移（遍历节区）
DWORD RvaToFileOffset(PE_FILE* pe, DWORD rva);

// 转换函数：文件偏移转 RVA（反向）
DWORD FileOffsetToRva(PE_FILE* pe, DWORD foa);
```

###  额外提醒
1. **`.textbss` 节区**：如果节区 `SizeOfRawData == 0` 但 `VirtualSize > 0`（如增量链接表），在合并节区时，`mappedSize` 必须取最大值 `VirtualSize`，否则内存空间会漏算。
2. **重定位表移动的边界**：重定位表的数据结构是连续的 `Block`，直接复制字节流完全安全，因为其中的 `VirtualAddress` 是绝对 RVA，不依赖节区基址。





# 完整代码

```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <Windows.h>
#include <stdlib.h>
#include <string.h>
#include <conio.h>


typedef struct
{
	PBYTE buffer;
	DWORD size;
	char inputPath[MAX_PATH];
	char outputPath[MAX_PATH];
	PIMAGE_DOS_HEADER dosHeader;
	PIMAGE_NT_HEADERS ntHeaders;
	PIMAGE_SECTION_HEADER sectionHeader;
	BOOL isModified;
}PE_FILE;

// =================
VOID PrintMenu();

DWORD AlignmentUp(DWORD value, DWORD alignment);

BOOL GetFilePath(char* buffer, int maxsize);

BOOL LoadPeFile(PE_FILE* pe, char* filepath);

BOOL IsPEFile(const BYTE* buffer);

void GenerateDefaultOutputName(const char* inputPath, char* outputPath);

void AnalyzePEFile(PE_FILE* pe);

void ClearScreen();

BOOL SavePEFile(PE_FILE* pe, const char* filePath);
// =================




void DisplayPEInfo(PE_FILE* pe);

BOOL MoveNtSectionToDosStub(PE_FILE* pe);

BOOL AddSection(PE_FILE* pe, const char* sectionName, DWORD sectionSize, DWORD characteristics);

BOOL MergeAllSection(PE_FILE* pe);

BOOL ExpandLastSection(PE_FILE* pe, DWORD additionalSize);

BOOL MoveExportTableToNewSection(PE_FILE* pe);

BOOL MoveImportTableToNewSection(PE_FILE* pe);

BOOL MoveRelocationTableToNewSection(PE_FILE* pe);

BOOL MoveRelocationTableToNewSection(PE_FILE* pe);



int main()
{
	PE_FILE pe = { 0 };
	int choice = 0;
	char tempBuffer[MAX_PATH] = { 0 };
	BOOL operationSuccess = FALSE;

	printf("%-30s\n", "============== PE修改 =============");

	while (1)
	{
		if (pe.buffer == NULL)
		{
			printf("%-30s", "请输入PE文件路径: ");
			if (!GetFilePath(tempBuffer, MAX_PATH))
			{
				continue;
			}

			if (!LoadPeFile(&pe, tempBuffer))
			{
				printf("%-30s\n", "无法LoadPEFile");
				break;
			}

			GenerateDefaultOutputName(pe.inputPath, pe.outputPath);
			printf("%-30s%s\n", "成功加载文件:", pe.inputPath);
			printf("%-30s%s\n", "默认输出文件:", pe.outputPath);

			AnalyzePEFile(&pe);

			printf("\n%-30s", "按任意键继续...");
			_getch();
			ClearScreen();
		}

		PrintMenu();

		printf("\n%-30s%s\n", "当前文件:", pe.inputPath);
		if (pe.isModified)
		{
			printf("%-30s\n", "状态: 已修改 (尚未保存) ");
		}
		else
		{
			printf("%-30s\n", "状态: 未修改");
		}

		printf("\n%-30s ", "请选择操作 [0-9]");
		scanf("%d", &choice);
		getchar();

		ClearScreen();
		operationSuccess = FALSE;

		switch (choice)
		{
		case 0:
		{
			printf("%-30s%s\n", "当前输出文件:", pe.outputPath);
			operationSuccess = SavePEFile(&pe, pe.outputPath);
			if (operationSuccess)
			{
				printf("%-30s%s\n", "文件成功保存到:", pe.outputPath);
				pe.isModified = FALSE;
			}
			else
			{
				printf("%-30s\n", "保存文件失败");
			}
			break;
		}
		case 1:
		{
			DisplayPEInfo(&pe);
			operationSuccess = TRUE;
			break;
		}
		case 2:
		{
			operationSuccess = MoveNtSectionToDosStub(&pe);
			if (operationSuccess)
			{
				printf("%-30s\n", "成功删除DOS_STUB");
				pe.isModified = TRUE;
			}
			else
			{
				printf("%-30s\n", "删除DOS_STUB失败");
			}

			break;
		}
		case 3:
		{
			char sectionName[9] = { 0 };
			DWORD sectionSize = 0;
			DWORD characteristics = 0;

			printf("%-30s: ", "请输入节区名(最多8个字符)");
			if (!GetFilePath(sectionName, 9))
			{
				printf("%-30s\n", "输入节区名称失败");
				break;
			}

			printf("%-30s", "请输入节区大小(十进制): ");
			scanf("%d", &sectionSize);
			getchar();

			printf("%-30s", "请输入节区特性(十六进制,如:0xF0000020):  ");
			scanf("%x", &characteristics);
			getchar();

			operationSuccess = AddSection(&pe, sectionName, sectionSize, characteristics);
			if (operationSuccess)
			{
				printf("%-30s\n", "成功增加节区");
				pe.isModified = TRUE;
			}
			else
			{
				printf("%-30s\n", "添加节区失败");
			}
			break;
		}
		case 4:
		{
			operationSuccess = MergeAllSection(&pe);
			if (operationSuccess)
			{
				printf("%-30s\n", "成功合并所有节区");
				pe.isModified = TRUE;
			}
			else
			{
				printf("%-30s\n", "合并节区失败");
			}
			break;
		}
		case 5:
		{
			DWORD additionalSize = 0;

			printf("%-30s", "请输入要扩大的大小(十进制, 单位: 字节");
			scanf("%d", &additionalSize);
			getchar();

			if (additionalSize <= 0)
			{
				printf("%-30s\n", "扩展大小必须大于n");
				break;
			}
			operationSuccess = ExpandLastSection(&pe, additionalSize);
			if (operationSuccess)
			{
				printf("%-30s\n", "成功扩大节区");
				break;
			}
			else
			{
				printf("%-30s\n", "扩大节区失败");
			}
			break;
		}
		case 6:
		{
			operationSuccess = MoveExportTableToNewSection(&pe);
			if (operationSuccess)
			{
				printf("%-30s\n", "导出表移动成功");
			}
			else
			{
				printf("%-30s\n", "导出表移动失败");
			}
			break;
		}
		case 7:
		{
			operationSuccess = MoveImportTableToNewSection(&pe);
			if (operationSuccess)
			{
				printf("%-30s\n", "导入表移动成功");
			}
			else
			{
				printf("%-30s\n", "导入表移动失败");
			}
			break;
		}
		case 8:
		{
			operationSuccess = MoveRelocationTableToNewSection(&pe);
			if (operationSuccess)
			{
				printf("%-30s\n", "重定位表移动成功");
			}
			else
			{
				printf("%-30s\n", "重定位表移动失败");
			}
			break;
		}

		default:
		{
			printf("%-30s\n", "choice错误");
			break;
		}
		}

	}

	return 0;
}
void PrintMenu()
{
	printf("[0] 保存文件\n");
	printf("[1] 查看简略PE文件信息\n");
	printf("[2] 删除DosStub\n");
	printf("[3] 添加新节区\n");
	printf("[4] 合并所有节区\n");
	printf("[5] 扩大最后一个节区\n");
	printf("[6] 移动导出表到新节区\n");
	printf("[7] 移动导入表到新节区\n");
	printf("[8] 移动重定位表到新节区\n");

}

DWORD AlignmentUp(DWORD value, DWORD alignment)
{
	return (value + alignment - 1) & ~(alignment - 1);
}


BOOL GetFilePath(char* buffer, int maxsize)
{
	if (!buffer || maxsize <= 0)
	{
		return FALSE;
	}

	if (!fgets(buffer, maxsize, stdin))
	{
		printf("%-30s\n", "读取输入失败");
		return FALSE;
	}

	int length = strlen(buffer);
	if (length == 0 || length >= maxsize)
	{
		printf("%-30s\n", "输入有误, 请重新输入");
		return FALSE;
	}

	buffer[length - 1] = '\0';
	return TRUE;
}

BOOL LoadPeFile(PE_FILE * pe, char* filepath)
{

	FILE* file = fopen(filepath, "rb");
	if (!file)
	{
		printf("%-30s%s\n", "无法打开文件", filepath);
		return FALSE;
	}

	fseek(file, 0, SEEK_END);
	pe->size = ftell(file);
	fseek(file, 0, SEEK_SET);

	if (pe->size <= 0)
	{
		printf("%-30s\n", "文件大小无效或为空");
		fclose(file);
		return FALSE;
	}

	pe->buffer = (PBYTE)malloc(pe->size);
	if (!pe->buffer)
	{
		printf("%-30s\n", "内存分配失败");
		fclose(file);
		return FALSE;
	}

	memset(pe->buffer, 0, pe->size);

	if (fread(pe->buffer, 1, pe->size, file) != pe->size)
	{
		printf("%-30s\n", "读取文件失败");
		free(pe->buffer);
		pe->buffer = NULL;
		fclose(file);
		return FALSE;
	}

	fclose(file);

	if (!IsPEFile(pe->buffer))
	{
		printf("%-30s\n", "不是有效的PE文件");
		free(pe->buffer);
		pe->buffer = NULL;
		fclose(file);
		return FALSE;
	}

	strcpy(pe->inputPath, filepath);

	return TRUE;
}

BOOL IsPEFile(const BYTE * buffer)
{
	PIMAGE_DOS_HEADER DosHeader = (PIMAGE_DOS_HEADER)buffer;

	if (DosHeader->e_magic != IMAGE_DOS_SIGNATURE)
	{
		return FALSE;
	}

	PIMAGE_NT_HEADERS NtHeaders = (PIMAGE_NT_HEADERS)(buffer + DosHeader->e_lfanew);
	if (NtHeaders->Signature != IMAGE_NT_SIGNATURE)
	{
		return FALSE;
	}

	return TRUE;
}

void GenerateDefaultOutputName(const char* inputPath, char* outputPath)
{
	char drive[_MAX_DRIVE];
	char dir[_MAX_DIR];
	char fname[_MAX_FNAME];
	char ext[_MAX_EXT];

	_splitpath(inputPath, drive, dir, fname, ext); // 将输入路径进行分割

	// 添加"_modified"后缀
	char newFname[_MAX_FNAME + 10];
	sprintf(newFname, "%s_modified", fname); // 增加_modified后缀名

	_makepath(outputPath, drive, dir, newFname, ext); // 生成修改后的路径名

}

void AnalyzePEFile(PE_FILE * pe)
{
	pe->dosHeader = (PIMAGE_DOS_HEADER)pe->buffer;
	pe->ntHeaders = (PIMAGE_NT_HEADERS)(pe->buffer + pe->dosHeader->e_lfanew);
	pe->sectionHeader = (PIMAGE_SECTION_HEADER)(IMAGE_FIRST_SECTION(pe->ntHeaders));

	printf("%-30s\n", "PE文件分析完成");
}

void ClearScreen()
{
	system("cls");
}

BOOL SavePEFile(PE_FILE* pe, const char* filePath)
{
	FILE* file = fopen(filePath, "wb");

	if (!file)
	{
		printf("%-30s%s\n", "无法创建文件:", filePath);
		return FALSE;
	}

	if (fwrite(pe->buffer, 1, pe->size, file) != pe->size)
	{
		printf("%-30s\n", "写入文件失败");
		fclose(file);
		return FALSE;
	}


	fclose(file);
	return TRUE;
}

void DisplayPEInfo(PE_FILE* pe)
{
	if (!pe || !pe->buffer || !pe->dosHeader || !pe->ntHeaders)
	{
		printf("%-30s\n", "PE文件尚未正确分析");
		return;
	}

	printf("========== DOS Header ==========\n");
	printf("e_magic:        0x%X\n", pe->dosHeader->e_magic);
	printf("e_lfanew:       0x%X\n", pe->dosHeader->e_lfanew);

	printf("\n========== NT Header ==========\n");
	printf("Signature:      0x%X\n", pe->ntHeaders->Signature);
	printf("Machine:        0x%X\n", pe->ntHeaders->FileHeader.Machine);
	printf("NumberOfSections: %d\n", pe->ntHeaders->FileHeader.NumberOfSections);
	printf("SizeOfOptionalHeader: 0x%X\n", pe->ntHeaders->FileHeader.SizeOfOptionalHeader);
	printf("Characteristics: 0x%X\n", pe->ntHeaders->FileHeader.Characteristics);

	printf("\n========== Optional Header ==========\n");
	printf("AddressOfEntryPoint: 0x%X\n", pe->ntHeaders->OptionalHeader.AddressOfEntryPoint);
	printf("ImageBase:           0x%p\n", (void*)pe->ntHeaders->OptionalHeader.ImageBase);
	printf("SectionAlignment:    0x%X\n", pe->ntHeaders->OptionalHeader.SectionAlignment);
	printf("FileAlignment:       0x%X\n", pe->ntHeaders->OptionalHeader.FileAlignment);
	printf("SizeOfImage:         0x%X\n", pe->ntHeaders->OptionalHeader.SizeOfImage);
	printf("SizeOfHeaders:       0x%X\n", pe->ntHeaders->OptionalHeader.SizeOfHeaders);

	printf("\n========== Section Headers ==========\n");

	for (int i = 0; i < pe->ntHeaders->FileHeader.NumberOfSections; i++)
	{
		IMAGE_SECTION_HEADER* sec = &pe->sectionHeader[i];

		char name[9] = { 0 };
		memcpy(name, sec->Name, 8);

		printf("\n[%d] %s\n", i + 1, name);
		printf("VirtualAddress:      0x%X\n", sec->VirtualAddress);
		printf("Misc.VirtualSize:    0x%X\n", sec->Misc.VirtualSize);
		printf("PointerToRawData:    0x%X\n", sec->PointerToRawData);
		printf("SizeOfRawData:       0x%X\n", sec->SizeOfRawData);
		printf("Characteristics:     0x%X\n", sec->Characteristics);
	}
}




BOOL MoveNtSectionToDosStub(PE_FILE* pe)
{
	// 计算DosStub大小
	DWORD dosStubSize = pe->dosHeader->e_lfanew - sizeof(IMAGE_DOS_HEADER);
	if (dosStubSize == 0)
	{
		printf("%-30s\n", "该PE文件没有DOS存根");
		return FALSE;
	}

	// 计算移动NtHeader, SectionHeader大小
	DWORD ntHeaderSize = sizeof(IMAGE_NT_HEADERS);
	DWORD sectionHeaderSize = sizeof(IMAGE_SECTION_HEADER) * pe->ntHeaders->FileHeader.NumberOfSections;
	DWORD allSize = ntHeaderSize + sectionHeaderSize;

	// 移动
	memmove(pe->buffer + sizeof(IMAGE_DOS_HEADER), pe->ntHeaders, allSize);

	// 修正偏移，ntheader,sectionheader指向
	pe->dosHeader->e_lfanew = sizeof(IMAGE_DOS_HEADER);
	pe->ntHeaders = (PIMAGE_NT_HEADERS)(pe->buffer + pe->dosHeader->e_lfanew);
	pe->sectionHeader = (PIMAGE_SECTION_HEADER)IMAGE_FIRST_SECTION(pe->ntHeaders);


	printf("%-30s\n", "DOS存根已成功删除 (NT头和节表以移动)");
	return TRUE;

}

BOOL AddSection(PE_FILE* pe, const char* sectionName, DWORD sectionSize, DWORD characteristics)
{

	if (!pe || !pe->buffer || !sectionName || sectionSize == 0)
	{
		printf("%-30s\n", "无效的参数");
		return FALSE;
	}

	if (strlen(sectionName) > 8)
	{
		printf("%-30s\n", "节区名称不能超过8个字符");
		return FALSE;
	}

	// 计算PE头末尾是否有足够空间存储新section头
	// dos + nt + section头大小
	DWORD allHeaderSize = pe->dosHeader->e_lfanew + sizeof(IMAGE_NT_HEADERS) + pe->ntHeaders->FileHeader.NumberOfSections * sizeof(IMAGE_SECTION_HEADER);
	
	// 所有头对齐后的大小
	DWORD headerAlignmentSize = pe->ntHeaders->OptionalHeader.SizeOfHeaders;

	DWORD remainSize = headerAlignmentSize - allHeaderSize;

	if (remainSize < 40) // 节区头大小为40字节
	{
		printf("%-30s\n", "没有足够的剩余空间来添加新节区头");
		return FALSE;
	}
	
	// sectionSize文件对齐后的大小
	DWORD newFileAligmentSize = (sectionSize + pe->ntHeaders->OptionalHeader.FileAlignment - 1) &
		~(pe->ntHeaders->OptionalHeader.FileAlignment - 1); 

	// 新节区在文件中的偏移 (最后一个节区偏移加上其大小)
	DWORD newSectionFoa = pe->sectionHeader[pe->ntHeaders->FileHeader.NumberOfSections - 1].PointerToRawData
		+ pe->sectionHeader[pe->ntHeaders->FileHeader.NumberOfSections - 1].SizeOfRawData;
	newSectionFoa = (newSectionFoa + pe->ntHeaders->OptionalHeader.FileAlignment - 1) &
		~(pe->ntHeaders->OptionalHeader.FileAlignment - 1);// 文件对齐

	// 新节区在内存中的偏移
	DWORD newSectionRva = pe->sectionHeader[pe->ntHeaders->FileHeader.NumberOfSections - 1].VirtualAddress
		+ pe->sectionHeader[pe->ntHeaders->FileHeader.NumberOfSections - 1].Misc.VirtualSize;
	newSectionRva = (newSectionRva + pe->ntHeaders->OptionalHeader.SectionAlignment - 1) &
		~(pe->ntHeaders->OptionalHeader.SectionAlignment - 1);// 内存对齐

	// 增加新节区后的总大小(对齐后)
	DWORD newTotalSize = newSectionFoa + newFileAligmentSize;

	// 重新申请新增内存大小
	pe->buffer = (PBYTE)realloc(pe->buffer, newTotalSize);
	if (pe->buffer == NULL)
	{
		printf("%-30s\n", "新申请内存失败");
		return FALSE;
	}
	memset(pe->buffer + newSectionFoa, 0, newFileAligmentSize);
	
	// 重新定位dos,nt,section头
	pe->dosHeader = (PIMAGE_DOS_HEADER)pe->buffer;
	pe->ntHeaders = (PIMAGE_NT_HEADERS)(pe->buffer + pe->dosHeader->e_lfanew);
	pe->sectionHeader = (PIMAGE_SECTION_HEADER)IMAGE_FIRST_SECTION(pe->ntHeaders);


	// 定位新节区
	PIMAGE_SECTION_HEADER newSection = (PIMAGE_SECTION_HEADER)(&pe->sectionHeader[pe->ntHeaders->FileHeader.NumberOfSections]);
	
	// 更新新节区值
	memset(newSection, 0, sizeof(IMAGE_SECTION_HEADER)); // 初始化
	newSection->Characteristics = characteristics; // 节区属性
	newSection->Misc.VirtualSize = sectionSize;    // 节区在内存中的大小(不用内存对齐,直接等于节区实际大小)
	strcpy((char*)newSection->Name, sectionName);	// 复制节区名称
	newSection->PointerToRawData = newSectionFoa;	// 节区在文件中的偏移
	newSection->SizeOfRawData = newFileAligmentSize;// 节区在文件中的大小(文件对齐后)
	newSection->VirtualAddress = newSectionRva;		// 节区在内存中的偏移

	// 更新nt头数据
	pe->ntHeaders->FileHeader.NumberOfSections++; // 节区数量加1
	pe->ntHeaders->OptionalHeader.SizeOfImage = (newSection->VirtualAddress + newSection->Misc.VirtualSize + pe->ntHeaders->OptionalHeader.SectionAlignment - 1) &
		~(pe->ntHeaders->OptionalHeader.SectionAlignment - 1); // 内存大小，加上新节区大小在内存对齐后的大小
	pe->size = newTotalSize;
	pe->isModified = TRUE;

	printf("%-30s%s\n", "成功添加新节区", sectionName);
	printf("%-30s0x%x\n", "虚拟地址:", newSectionRva);
	printf("%-30s0x%x\n", "虚拟大小", sectionSize);
	printf("%-30s0x%x\n", "原始数据偏移", newSectionFoa);
	printf("%-30s0x%x\n", "原始数据大小", newFileAligmentSize);

	return TRUE;
}

BOOL MergeAllSection(PE_FILE* pe)
{
	// ======================== 函数入口：合并 PE 所有节区 ========================
	// 前提：pe 结构已加载有效 PE 文件，所有指针已初始化

	// 检查 PE 结构是否完整：buffer、DOS头、NT头、节表都不能为空
	if (!pe || !pe->buffer || !pe->dosHeader || !pe->ntHeaders || !pe->sectionHeader)
	{
		printf("%-30s\n", "无效PE文件");
		return FALSE;
	}

	// 获取当前节区数量（WORD 类型，最大 65535，但 PE 通常不超过 96）
	WORD numberOfSections = pe->ntHeaders->FileHeader.NumberOfSections;

	// 如果只有 1 个或 0 个节区，合并没有意义，直接返回
	if (numberOfSections <= 1)
	{
		printf("%-30s\n", "没有足够的节区数量");
		return FALSE;
	}

	// 读取文件对齐粒度（硬盘上节区数据的对齐单位）和内存对齐粒度（内存中节区起始地址的对齐单位）
	DWORD fileAlignment = pe->ntHeaders->OptionalHeader.FileAlignment;
	DWORD sectionAlignment = pe->ntHeaders->OptionalHeader.SectionAlignment;

	// 对齐值不能为 0，否则后续除零或无限循环
	if (fileAlignment == 0 || sectionAlignment == 0)
	{
		printf("%-30s\n", "无效的对齐值");
		return FALSE;
	}

	// 指向原始节表数组（第一个节区的指针）
	PIMAGE_SECTION_HEADER oldSection = pe->sectionHeader;

	// 记录原始文件总大小（用于后续 Overlay 计算）
	DWORD oldFileSize = pe->size;

	// 以第一个节区为“锚点”：它的 RVA 和文件偏移作为合并后的基准
	DWORD baseRva = oldSection[0].VirtualAddress;
	DWORD baseRaw = oldSection[0].PointerToRawData;

	// 校验锚点合法性：
	// 1. baseRaw 不能为 0（通常在文件头后面）
	// 2. 不能超出文件大小
	// 3. 不能小于头部大小（否则会覆盖 PE 头）
	if (baseRaw == 0 || baseRaw > oldFileSize ||
		baseRaw < pe->ntHeaders->OptionalHeader.SizeOfHeaders)
	{
		printf("%-30s\n", "第一个节区Raw偏移异常");
		printf("%-30s\n", "尝试强制处理: 修改baseraw为头部之后的起始位置");
		// 容错处理：如果第一个节区偏移异常，强行设为头部结束位置
		// 注意：这可能导致数据覆盖，但总比直接失败好
		baseRaw = pe->ntHeaders->OptionalHeader.SizeOfHeaders;
	}

	// ---------- 遍历节区，收集最大边界和属性 ----------
	DWORD maxRvaEnd = baseRva;          // 内存中所有节区的最大结束地址（RVA终点）
	DWORD maxNewRawEnd = baseRaw;       // 重新排列后，所有节区在文件中的最大结束偏移
	DWORD oldRawEnd = baseRaw;          // 原始文件中，最后一个有数据的节区结束偏移（用于Overlay）
	DWORD mergedCharacteristics = 0;    // 合并后节区的属性（OR 所有原节区属性）

	// 遍历所有节区
	for (WORD i = 0; i < numberOfSections; i++)
	{
		PIMAGE_SECTION_HEADER sec = &oldSection[i];

		// 确保第一个节区是 RVA 最小的（否则不能简单以第一个为锚点）
		if (sec->VirtualAddress < baseRva)
		{
			printf("%-30s\n", "第一个节不是RVA最小的节，不能直接合并");
			return FALSE;
		}

		// 计算该节区在内存中的“实际占用大小”：取 VirtualSize 和 SizeOfRawData 的最大值
		// 原因：.bss 节 VirtualSize 大但文件大小为0；.text 节文件大小因对齐可能大于 VirtualSize
		DWORD mappedSize = sec->Misc.VirtualSize > sec->SizeOfRawData
			? sec->Misc.VirtualSize
			: sec->SizeOfRawData;

		// 该节区在内存中的结束地址（RVA终点）
		DWORD rvaEnd = sec->VirtualAddress + mappedSize;

		// 更新全局最大内存终点
		if (rvaEnd > maxRvaEnd)
			maxRvaEnd = rvaEnd;

		// 如果该节区在文件中有实际数据（SizeOfRawData != 0），才处理文件偏移
		if (sec->SizeOfRawData != 0)
		{
			// 原始文件中的结束偏移
			DWORD rawEnd = sec->PointerToRawData + sec->SizeOfRawData;

			// 校验原始偏移是否在文件范围内
			if (sec->PointerToRawData >= oldFileSize || rawEnd > oldFileSize)
			{
				printf("%-30s\n", "节区Raw范围超出文件大小");
				return FALSE;
			}

			// 计算该节区在“合并后新文件”中的起始偏移（按 RVA 顺序紧凑排列）
			DWORD newRaw = baseRaw + (sec->VirtualAddress - baseRva);
			// 该节区在新文件中的结束偏移
			DWORD newRawEnd = newRaw + sec->SizeOfRawData;

			// 更新全局最大新文件偏移
			if (newRawEnd > maxNewRawEnd)
				maxNewRawEnd = newRawEnd;

			// 更新原始文件的最后一个有效数据结尾（用于 Overlay 的起始位置）
			if (rawEnd > oldRawEnd)
				oldRawEnd = rawEnd;
		}

		// 合并所有节区的属性标志（按位或）
		mergedCharacteristics |= sec->Characteristics;
	}

	// ---------- 计算合并后的各种大小 ----------
	// 内存中所有节区的总跨度（从第一个节区起始到最后一个节区终点）
	DWORD newVirtualSize = maxRvaEnd - baseRva;

	// 硬盘上新节区的原始数据大小（按 fileAlignment 向上对齐）
	DWORD newRawSize = AlignmentUp(maxNewRawEnd - baseRaw, fileAlignment);

	// 硬盘上新节区数据的结束偏移（baseRaw + 对齐后的大小）
	DWORD newRawEnd = baseRaw + newRawSize;

	// 整个 PE 映像在内存中的总大小（必须按 sectionAlignment 对齐）
	DWORD newSizeOfImage = AlignmentUp(maxRvaEnd, sectionAlignment);

	// 检查所有大小是否在 32 位 DWORD 范围内（防止溢出，但代码中用了 DWORD 存储，所以要确保不超）
	if (newVirtualSize > 0xFFFFFFFFULL ||
		newRawSize > 0xFFFFFFFFULL ||
		newRawEnd > 0xFFFFFFFFULL ||
		newSizeOfImage > 0xFFFFFFFFULL)
	{
		printf("%-30s\n", "合并后大小溢出");
		return FALSE;
	}

	// ---------- 处理 Overlay（附加数据，如数字签名） ----------
	// Overlay 起始位置：原始文件中最后一个节区结束的位置
	DWORD oldOverlayStart = (DWORD)oldRawEnd;
	DWORD overlaySize = 0;

	// 如果文件总大小大于所有节区数据结束位置，说明存在 Overlay
	if (oldFileSize > oldOverlayStart)
		overlaySize = oldFileSize - oldOverlayStart;

	// 新文件总大小 = 新节区数据结束偏移 + Overlay 大小
	DWORD newFileSize = newRawEnd + overlaySize;

	// 再次检查是否溢出
	if (newFileSize > 0xFFFFFFFFULL)
	{
		printf("%-30s\n", "新文件大小溢出");
		return FALSE;
	}

	// ---------- 分配新内存并构建新文件 ----------
	PBYTE newBuffer = (PBYTE)malloc((size_t)newFileSize);
	if (!newBuffer)
	{
		printf("%-30s\n", "内存分配失败");
		return FALSE;
	}

	// 将新缓冲区全部清零（确保未使用的部分为 0）
	memset(newBuffer, 0, (size_t)newFileSize);

	// 复制 PE 头（从文件开头到第一个节区之前的所有数据，即 DOS头、NT头、节表）
	// 因为第一个节区的数据将从 baseRaw 开始，所以复制 [0, baseRaw) 部分
	memcpy(newBuffer, pe->buffer, baseRaw);

	// ---------- 按新偏移将每个节区的原始数据搬运到新缓冲区 ----------
	for (WORD i = 0; i < numberOfSections; i++)
	{
		PIMAGE_SECTION_HEADER sec = &oldSection[i];

		// 如果该节区在文件中没有数据（SizeOfRawData == 0），跳过（如 .bss 节）
		if (sec->SizeOfRawData == 0)
			continue;

		// 计算该节区在新文件中的偏移
		DWORD newRaw = baseRaw + (sec->VirtualAddress - baseRva);

		// 安全检查：防止越界写入
		if (newRaw + sec->SizeOfRawData > newRawEnd)
		{
			free(newBuffer);
			printf("%-30s\n", "新Raw范围计算错误");
			return FALSE;
		}

		// 从旧缓冲区复制该节区的原始数据到新位置
		memcpy(
			newBuffer + newRaw,
			pe->buffer + sec->PointerToRawData,
			sec->SizeOfRawData
		);
	}

	// ---------- 保留 Overlay 数据 ----------
	if (overlaySize != 0)
	{
		// 将 Overlay 数据拷贝到新文件末尾（紧接在合并后的节区数据之后）
		memcpy(
			newBuffer + (DWORD)newRawEnd,
			pe->buffer + oldOverlayStart,
			overlaySize
		);
	}

	// 释放旧缓冲区
	free(pe->buffer);

	// 更新 PE 结构中的缓冲区指针和大小
	pe->buffer = newBuffer;
	pe->size = (DWORD)newFileSize;

	// ---------- 重新定位 PE 头指针（因为新缓冲区地址变了） ----------
	pe->dosHeader = (PIMAGE_DOS_HEADER)pe->buffer;
	pe->ntHeaders = (PIMAGE_NT_HEADERS)(pe->buffer + pe->dosHeader->e_lfanew);
	pe->sectionHeader = (PIMAGE_SECTION_HEADER)IMAGE_FIRST_SECTION(pe->ntHeaders);

	// 获取第一个节区（现在也是唯一有效的节区）
	PIMAGE_SECTION_HEADER first = &pe->sectionHeader[0];

	// 更新第一个节区的信息，让它覆盖所有合并后的数据
	first->Misc.VirtualSize = (DWORD)newVirtualSize;    // 内存中总跨度
	first->SizeOfRawData = (DWORD)newRawSize;           // 硬盘文件大小（对齐后）
	first->PointerToRawData = baseRaw;                  // 文件偏移（保持原锚点）

	// 设置合并后的节区属性：合并所有原节区属性
	first->Characteristics = mergedCharacteristics;

	// 强制加上读、写、执行权限，确保合并后的节区可运行（许多程序依赖）
	first->Characteristics |= IMAGE_SCN_MEM_READ;
	first->Characteristics |= IMAGE_SCN_MEM_WRITE;
	first->Characteristics |= IMAGE_SCN_MEM_EXECUTE;

	// 清除可能引起问题的标志：不能是可丢弃的、不能是链接器临时节等
	first->Characteristics &= ~IMAGE_SCN_MEM_DISCARDABLE;
	first->Characteristics &= ~IMAGE_SCN_LNK_REMOVE;
	first->Characteristics &= ~IMAGE_SCN_LNK_INFO;

	// 将除第一个节区外的所有节区头清零（因为已经合并，它们不再有效）
	for (WORD i = 1; i < numberOfSections; i++)
	{
		memset(&pe->sectionHeader[i], 0, sizeof(IMAGE_SECTION_HEADER));
	}

	// 更新 NT 头中的节区数量为 1
	pe->ntHeaders->FileHeader.NumberOfSections = 1;

	// 更新映像总大小（必须按内存对齐）
	pe->ntHeaders->OptionalHeader.SizeOfImage = newSizeOfImage;

	// 清零校验和，因为文件已被修改，原校验和无效，避免系统验证失败
	pe->ntHeaders->OptionalHeader.CheckSum = 0;

	// ---------- 特殊处理：安全目录（数字证书） ----------
	// 注意：IMAGE_DIRECTORY_ENTRY_SECURITY 的 VirtualAddress 实际存储的是文件偏移（FOA），不是 RVA
	// 如果证书数据在 Overlay 中，且 Overlay 被移动了，需要更新该偏移
	if (pe->ntHeaders->OptionalHeader.NumberOfRvaAndSizes > IMAGE_DIRECTORY_ENTRY_SECURITY)
	{
		PIMAGE_DATA_DIRECTORY securityDir =
			&pe->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_SECURITY];

		// 判断原证书偏移是否在旧的 Overlay 区域内
		if (securityDir->VirtualAddress >= oldOverlayStart &&
			securityDir->VirtualAddress < oldFileSize)
		{
			// 更新为新文件中的偏移：新Overlay起始 + 原偏移相对于旧Overlay起始的偏移
			securityDir->VirtualAddress =
				newRawEnd + (securityDir->VirtualAddress - oldOverlayStart);
		}
	}

	// 标记 PE 已被修改
	pe->isModified = TRUE;

	// ---------- 打印合并结果信息 ----------
	char name[9] = { 0 };
	memcpy(name, first->Name, 8);  // 节区名称最多 8 字节，取第一个节区的名称

	printf("%-30s\n", "所有节区已成功合并为一个节区");
	printf("%-30s%s\n", "保留节区名称:", name);
	printf("%-30s0x%X\n", "虚拟地址:", first->VirtualAddress);
	printf("%-30s0x%X\n", "虚拟大小:", first->Misc.VirtualSize);
	printf("%-30s0x%X\n", "原始数据偏移:", first->PointerToRawData);
	printf("%-30s0x%X\n", "原始数据大小:", first->SizeOfRawData);

	return TRUE;
}

BOOL ExpandLastSection(PE_FILE* pe, DWORD additionalSize)
{
	// 检查 PE 结构是否有效（至少 buffer 不能为空）
	if (!pe || !pe->buffer)
	{
		printf("%-30s\n", "无效PE文件");
		return FALSE;
	}

	// 获取当前节区数量
	WORD numberOfSections = pe->ntHeaders->FileHeader.NumberOfSections;

	// 至少需要有一个节区才能操作
	if (numberOfSections < 1)
	{
		printf("%-30s\n", "没有节区");
		return FALSE;
	}

	// 获取最后一个节区的指针（节表数组的最后一个元素）
	PIMAGE_SECTION_HEADER lastSection = (PIMAGE_SECTION_HEADER)(&pe->sectionHeader[numberOfSections - 1]);

	// 读取文件对齐粒度和内存对齐粒度
	DWORD fileAlignment = pe->ntHeaders->OptionalHeader.FileAlignment;
	DWORD sectionAlignment = pe->ntHeaders->OptionalHeader.SectionAlignment;

	// 打印当前最后一个节区的信息（调试/用户反馈）
	printf("%-30s%s\n", "最后一个节区名称", (char*)lastSection->Name);
	printf("%-30s0x%x\n", "原始虚拟地址大小", lastSection->Misc.VirtualSize);
	printf("%-30s0x%x\n", "原始数据大小", lastSection->SizeOfRawData);

	// 计算新的虚拟大小（原虚拟大小 + 要增加的字节数）
	DWORD newVirtualSize = lastSection->Misc.VirtualSize + additionalSize;

	// 将新虚拟大小按文件对齐粒度向上对齐，得到新的文件原始数据大小
	DWORD newRawDataSize = AlignmentUp(newVirtualSize, fileAlignment);

	// 将旧的文件大小也按文件对齐（虽然 SizeOfRawData 本来就应该是对齐的，但以防万一）
	DWORD oldRawSize = AlignmentUp(lastSection->SizeOfRawData, fileAlignment);

	// 计算需要额外增加的文件字节数（新对齐大小 - 旧对齐大小）
	DWORD extraBytes = newRawDataSize - oldRawSize;

	// ---------- 情况 1：不需要额外文件空间 ----------
	// 条件：extraBytes == 0 且 新虚拟大小 <= 旧文件大小（说明旧文件大小已经足够容纳）
	// 这种情况只需要更新 VirtualSize 和 SizeOfImage 即可，无需扩展文件
	if (extraBytes == 0 && newVirtualSize <= lastSection->SizeOfRawData)
	{
		// 更新虚拟大小（磁盘上 SizeOfRawData 不变，因为空间足够）
		lastSection->Misc.VirtualSize = newVirtualSize;

		// 按内存对齐粒度对齐虚拟大小，用于更新 SizeOfImage
		DWORD alignedVirtualSize = (newVirtualSize + sectionAlignment - 1) & ~(sectionAlignment - 1);

		// 更新整个 PE 映像在内存中的总大小
		// 注意：最后一个节区在内存中的结束地址 = lastSection->VirtualAddress + alignedVirtualSize
		pe->ntHeaders->OptionalHeader.SizeOfImage = lastSection->VirtualAddress + alignedVirtualSize;

		// 打印成功信息
		printf("%-30s\n", "最后一个节区成功扩展");
		printf("%-30s0x%x", "新的虚拟大小", lastSection->Misc.VirtualSize);
		printf("%-30s0x%x", "新的原始数据大小", lastSection->SizeOfRawData);
		printf("%-30s0x%x", "新的映像大小:", pe->ntHeaders->OptionalHeader.SizeOfImage);

		// 标记 PE 已被修改
		pe->isModified = TRUE;
		return TRUE;
	}

	// ---------- 情况 2：需要扩展文件（extraBytes > 0） ----------
	// 说明新的大小超过了原来的文件分配空间，需要增加文件长度
	if (extraBytes > 0)
	{
		// 计算新文件总大小 = 原文件大小 + 额外字节
		DWORD newFileSize = pe->size + extraBytes;

		// 使用 realloc 重新分配缓冲区，保留原有数据，并增加 extraBytes 空间
		// realloc 可能会移动内存块，所以需要更新所有指针
		PBYTE newBuffer = (PBYTE)realloc(pe->buffer, newFileSize);
		if (!newBuffer)
		{
			printf("内存分配失败");
			return FALSE;
		}

		// 更新 PE 结构中的缓冲区指针和大小
		pe->buffer = newBuffer;
		pe->size = newFileSize;

		// ----- 重新定位所有 PE 头指针（因为 realloc 可能移动了内存地址） -----
		pe->dosHeader = (PIMAGE_DOS_HEADER)(pe->buffer);
		pe->ntHeaders = (PIMAGE_NT_HEADERS)(pe->buffer + pe->dosHeader->e_lfanew);
		pe->sectionHeader = (PIMAGE_SECTION_HEADER)IMAGE_FIRST_SECTION(pe->ntHeaders);

		// 重新获取最后一个节区的指针（因为节表地址可能变了）
		lastSection = &pe->sectionHeader[numberOfSections - 1];
	}

	// ---------- 更新节区头字段（无论是否需要扩展文件，都要执行） ----------
	// 更新虚拟大小
	lastSection->Misc.VirtualSize = newVirtualSize;
	// 更新文件原始数据大小（已经按文件对齐）
	lastSection->SizeOfRawData = newRawDataSize;

	// 按内存对齐粒度对齐虚拟大小
	DWORD alignedVirtualSize = AlignmentUp(newVirtualSize, sectionAlignment);

	// 更新整个映像的内存总大小
	pe->ntHeaders->OptionalHeader.SizeOfImage = lastSection->VirtualAddress + alignedVirtualSize;

	// 打印成功信息
	printf("%-30s\n", "最后一个节区成功扩展");
	printf("%-30s0x%x", "新的虚拟大小", lastSection->Misc.VirtualSize);
	printf("%-30s0x%x", "新的原始数据大小", lastSection->SizeOfRawData);
	printf("%-30s0x%x", "新的映像大小:", pe->ntHeaders->OptionalHeader.SizeOfImage);

	// 标记 PE 已被修改
	pe->isModified = TRUE;
	return TRUE;
}

DWORD RvaToFoa(PE_FILE* pe, DWORD rva)
{
	PIMAGE_SECTION_HEADER pSection = &pe->sectionHeader[0];
	DWORD numberOfSections = pe->ntHeaders->FileHeader.NumberOfSections;

	for (int i = 0; i < numberOfSections; i++)
	{
		if (pSection->VirtualAddress <= rva && rva < pSection->VirtualAddress + pSection->Misc.VirtualSize)
		{
			DWORD offset = rva - pSection->VirtualAddress;
			DWORD foa = pSection->PointerToRawData + offset;
			return foa;
		}

		pSection++;
	}

	return 0;
}

DWORD FoaToRva(PE_FILE* pe, DWORD foa)
{
	PIMAGE_SECTION_HEADER pSection = &pe->sectionHeader[0];
	DWORD numberOfSections = pe->ntHeaders->FileHeader.NumberOfSections;

	for (int i = 0; i < numberOfSections; i++)
	{
		if (pSection->PointerToRawData <= foa && foa < pSection->PointerToRawData + pSection->SizeOfRawData)
		{
			DWORD offset = foa - pSection->PointerToRawData;
			DWORD rva = pSection->VirtualAddress + offset;
			return rva;
		}

		pSection++;
	}

	return 0;


}

BOOL MoveExportTableToNewSection(PE_FILE* pe)
{
	PIMAGE_DATA_DIRECTORY pExportDir = &pe->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT];
	if (pExportDir->VirtualAddress == 0 || pExportDir->Size == 0)
	{
		printf("%-30s\n", "不存在导出表");
		return FALSE;
	}

	DWORD exportDirFoa = RvaToFoa(pe, pExportDir->VirtualAddress);
	if (exportDirFoa == 0)
	{
		printf("%-30s\n", "RvaToFoa转化失败");
		return FALSE;
	}

	PIMAGE_EXPORT_DIRECTORY pExportTable = (PIMAGE_EXPORT_DIRECTORY)(pe->buffer + exportDirFoa);

	printf("%-30s\n", "原始导出表信息");
	printf("%-30s0x%x\n", "导出模块名称", pExportTable->Name);
	printf("%-30s0x%x\n", "导出函数数量", pExportTable->NumberOfFunctions);
	printf("%-30s0x%x\n", "导出名称数量", pExportTable->NumberOfNames);
	printf("%-30s0x%x\n", "导出函数RVA", pExportTable->AddressOfFunctions);
	printf("%-30s0x%x\n", "导出名称RVA", pExportTable->AddressOfNames);
	printf("%-30s0x%x\n", "导出名称序号RVA", pExportTable->AddressOfNameOrdinals);

	DWORD nameRva = pExportTable->Name;
	DWORD addressTableRva = pExportTable->AddressOfFunctions;
	DWORD namePointerTableRva = pExportTable->AddressOfNames;
	DWORD ordinalTableRva = pExportTable->AddressOfNameOrdinals;

	DWORD nameFoa = RvaToFoa(pe, nameRva);
	DWORD addressTableFoa = RvaToFoa(pe, addressTableRva);
	DWORD namePointerTableFoa = RvaToFoa(pe, namePointerTableRva);
	DWORD ordinalTableFoa = RvaToFoa(pe, ordinalTableRva);

	// 计算导出表数据大小
	DWORD exportDataSize = 0;

	// 1. 计算DLL名称字符串大小
	DWORD dllNameSize = (DWORD)strlen((char*)(pe->buffer + nameFoa)) + 1;

	// 2. 计算导出地址表大小
	DWORD addressTableSize = pExportTable->NumberOfFunctions * sizeof(DWORD);

	// 3. 计算名称指针表大小
	DWORD namePointerTableSize = pExportTable->NumberOfNames * sizeof(DWORD);

	// 4. 计算序号表大小
	DWORD ordinalTableSize = pExportTable->NumberOfNames * sizeof(WORD);

	// 5. 计算名称字符串表大小
	DWORD nameStringTableSize = 0;
	DWORD* namePointers = (DWORD*)(pe->buffer + namePointerTableFoa);
	for (DWORD i = 0; i < pExportTable->NumberOfNames; i++)
	{
		DWORD nameFoa = RvaToFoa(pe, namePointers[i]);
		nameStringTableSize += (DWORD)strlen((char*)pe->buffer + nameFoa) + 1;
	}

	// 6. 计算总大小
	exportDataSize = sizeof(IMAGE_EXPORT_DIRECTORY) + dllNameSize +
		addressTableSize + namePointerTableSize +
		ordinalTableSize + nameStringTableSize;

	// 7. 添加一些额外空间
	exportDataSize = (exportDataSize + 1023) & ~1023; // 向上对齐1kb


	// 创建新节区存储导出表
	char sectionName[] = ".exData";
	DWORD sectionCharacteristics = IMAGE_SCN_MEM_READ | IMAGE_SCN_CNT_INITIALIZED_DATA;

	printf("%-30s\n", "正在创建新节区用于存放导出表数据");
	if (!AddSection(pe, sectionName, exportDataSize, sectionCharacteristics))
	{
		printf("%-30s\n", "创建新节区失败");
		return FALSE;
	}

	// 获取新创建的节区
	PIMAGE_SECTION_HEADER newSection = &pe->sectionHeader[pe->ntHeaders->FileHeader.NumberOfSections - 1];

	// 新导出表的起始位置
	DWORD newExportDirRva = newSection->VirtualAddress;
	DWORD newExportDirFoa = newSection->PointerToRawData;

	// 复制导出目录表到新节区
	PIMAGE_EXPORT_DIRECTORY newExportDir = (PIMAGE_EXPORT_DIRECTORY)(pe->buffer + newExportDirFoa);
	memcpy(newExportDir, pExportTable, sizeof(IMAGE_EXPORT_DIRECTORY));

	// 获取当前节区偏移
	DWORD currentOffset = sizeof(IMAGE_EXPORT_DIRECTORY);

	// 1.复制DLL名称字符串
	char* dllName = (char*)(pe->buffer + nameFoa);
	char* newDllName = (char*)(pe->buffer + newExportDirFoa + currentOffset);
	strcpy(newDllName, dllName);

	// 1.更新导出目录中的DLL名称地址
	newExportDir->Name = newExportDirRva + currentOffset;
	currentOffset += dllNameSize;

	// 2.复制并更新导出地址表
	DWORD* addressTable = (DWORD*)(pe->buffer + addressTableFoa);
	DWORD* newAddressTable = (DWORD*)(pe->buffer + newExportDirFoa + currentOffset);
	memcpy(newAddressTable, addressTable, addressTableSize);

	// 2.更新导出目录中的地址表地址
	newExportDir->AddressOfFunctions = newExportDirRva + currentOffset;
	currentOffset += addressTableSize;

	// 3.复制并更新名称指针表
	DWORD* nameTable = (DWORD*)(pe->buffer + newExportDirFoa + currentOffset);

	// 3.暂存旧的名称指针位置，后面更新
	DWORD namePointerOffset = currentOffset;
	currentOffset += namePointerTableSize;

	// 3.更新导出目录中的名称指针表地址
	newExportDir->AddressOfNames = newExportDirRva + namePointerOffset;

	// 4.复制并更新序号表
	WORD* ordinalTable = (WORD*)(pe->buffer + ordinalTableFoa);
	WORD* newOrdinalTable = (WORD*)(pe->buffer + newExportDirFoa + currentOffset);
	memcpy(newOrdinalTable, ordinalTable, ordinalTableSize);

	// 4.更新导出目录中的序号表地址
	newExportDir->AddressOfNameOrdinals = newExportDirRva + currentOffset;
	currentOffset += ordinalTableSize;

	// 5.复制名称字符串表并更新名称指针
	DWORD* newNamePointers = (DWORD*)(pe->buffer + newExportDirFoa + namePointerOffset);
	for (int i = 0; i < pExportTable->NumberOfNames; i++)
	{
		// 获取原始名称字符串
		DWORD oldNameRva = namePointers[i];
		DWORD oldNameFoa = RvaToFoa(pe, oldNameRva);
		char* functionName = (char*)(pe->buffer + oldNameFoa);

		// 复制到新位置
		char* newFunctionName = (char*)(pe->buffer + newExportDirFoa + currentOffset);
		strcpy(newFunctionName, functionName);

		// 更新名称指针表
		newNamePointers[i] = newExportDirRva + currentOffset;

		// 更新偏移量
		currentOffset += (DWORD)strlen(functionName) + 1;

	}

	// 更新PE数据目录中的导出表条目
	pe->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress = newExportDirRva;
	pe->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].Size = exportDataSize;

	return TRUE;
}

BOOL MoveImportTableToNewSection(PE_FILE* pe)
{
	if (!pe || !pe->buffer || !pe->dosHeader || !pe->ntHeaders || !pe->sectionHeader)
	{
		printf("%-30s\n", "无效PE文件");
		return FALSE;
	}

	if (pe->ntHeaders->OptionalHeader.NumberOfRvaAndSizes <= IMAGE_DIRECTORY_ENTRY_IMPORT)
	{
		printf("%-30s\n", "PE没有导入表目录项");
		return FALSE;
	}

	PIMAGE_DATA_DIRECTORY pImportDir =
		&pe->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT];

	if (pImportDir->VirtualAddress == 0 || pImportDir->Size == 0)
	{
		printf("%-30s\n", "不存在导入表");
		return FALSE;
	}

	DWORD oldImportRva = pImportDir->VirtualAddress;
	DWORD oldImportFoa = RvaToFoa(pe, oldImportRva);

	if (oldImportFoa == 0 || oldImportFoa >= pe->size)
	{
		printf("%-30s\n", "导入表RVA转FOA失败");
		return FALSE;
	}

	PIMAGE_IMPORT_DESCRIPTOR oldImportDesc =
		(PIMAGE_IMPORT_DESCRIPTOR)(pe->buffer + oldImportFoa);

	// 统计 IMAGE_IMPORT_DESCRIPTOR 数量
	DWORD descCount = 0;
	DWORD maxDescCount = (pe->size - oldImportFoa) / sizeof(IMAGE_IMPORT_DESCRIPTOR);

	while (descCount < maxDescCount)
	{
		if (oldImportDesc[descCount].Name == 0 && oldImportDesc[descCount].FirstThunk == 0)
		{
			break;
		}
		descCount++;
	}

	if (descCount == maxDescCount)
	{
		printf("%-30s\n", "导入描述符没有找到结束项");
		return FALSE;
	}

	// 加上最后的全0结束描述符
	DWORD importDataSize = (descCount + 1) * sizeof(IMAGE_IMPORT_DESCRIPTOR);

	DWORD sectionCharacteristics =
		IMAGE_SCN_MEM_READ |
		IMAGE_SCN_MEM_WRITE |
		IMAGE_SCN_CNT_INITIALIZED_DATA;

	printf("%-30s\n", "正在创建新节区用于存放导入描述符");

	if (!AddSection(pe, ".ImpDa", importDataSize, sectionCharacteristics))
	{
		printf("%-30s\n", "添加新节区失败");
		return FALSE;
	}

	// AddSection 内部可能 realloc，所以这里必须重新定位所有指针
	pe->dosHeader = (PIMAGE_DOS_HEADER)pe->buffer;
	pe->ntHeaders = (PIMAGE_NT_HEADERS)(pe->buffer + pe->dosHeader->e_lfanew);
	pe->sectionHeader = (PIMAGE_SECTION_HEADER)IMAGE_FIRST_SECTION(pe->ntHeaders);

	pImportDir =
		&pe->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT];

	// 旧导入表 FOA 没有变，但 buffer 地址可能变了，所以重新生成指针
	oldImportDesc = (PIMAGE_IMPORT_DESCRIPTOR)(pe->buffer + oldImportFoa);

	PIMAGE_SECTION_HEADER newSection =
		&pe->sectionHeader[pe->ntHeaders->FileHeader.NumberOfSections - 1];

	DWORD newImportRva = newSection->VirtualAddress;
	DWORD newImportFoa = newSection->PointerToRawData;

	PIMAGE_IMPORT_DESCRIPTOR newImportDesc =
		(PIMAGE_IMPORT_DESCRIPTOR)(pe->buffer + newImportFoa);

	// 复制导入描述符数组，包括最后的全0项
	memcpy(newImportDesc, oldImportDesc, importDataSize);

	// 关键：更新导入表数据目录
	pImportDir->VirtualAddress = newImportRva;
	pImportDir->Size = importDataSize;

	// 修改后原校验和无效
	pe->ntHeaders->OptionalHeader.CheckSum = 0;

	pe->isModified = TRUE;

	printf("%-30s\n", "导入描述符已移动到新节区");
	printf("%-30s0x%X\n", "原导入表RVA:", oldImportRva);
	printf("%-30s0x%X\n", "新导入表RVA:", newImportRva);
	printf("%-30s0x%X\n", "新导入表FOA:", newImportFoa);
	printf("%-30s%d\n", "导入描述符数量:", descCount);
	printf("%-30s0x%X\n", "复制大小:", importDataSize);

	return TRUE;
}

BOOL MoveRelocationTableToNewSection(PE_FILE* pe)
{
	if (!pe || !pe->buffer || !pe->dosHeader || !pe->ntHeaders || !pe->sectionHeader)
	{
		printf("%-30s\n", "无效PE文件");
		return FALSE;
	}

	if (pe->ntHeaders->OptionalHeader.NumberOfRvaAndSizes <= IMAGE_DIRECTORY_ENTRY_BASERELOC)
	{
		printf("%-30s\n", "PE没有重定位表目录项");
		return FALSE;
	}

	PIMAGE_DATA_DIRECTORY pRelocDir =
		&pe->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC];

	if (pRelocDir->VirtualAddress == 0 || pRelocDir->Size == 0)
	{
		printf("%-30s\n", "不存在重定位表");
		return FALSE;
	}

	DWORD oldRelocRva = pRelocDir->VirtualAddress;
	DWORD relocSize = pRelocDir->Size;
	DWORD oldRelocFoa = RvaToFoa(pe, oldRelocRva);

	if (oldRelocFoa == 0)
	{
		printf("%-30s\n", "重定位表RVA转FOA失败");
		return FALSE;
	}

	if ((ULONGLONG)oldRelocFoa + relocSize > pe->size)
	{
		printf("%-30s\n", "重定位表范围超出文件大小");
		return FALSE;
	}

	// 简单校验重定位块结构
	DWORD offset = 0;
	DWORD blockCount = 0;

	while (offset + sizeof(IMAGE_BASE_RELOCATION) <= relocSize)
	{
		PIMAGE_BASE_RELOCATION pBlock =
			(PIMAGE_BASE_RELOCATION)(pe->buffer + oldRelocFoa + offset);

		// 遇到全0块，一般是对齐填充，可以停止解析
		if (pBlock->VirtualAddress == 0 && pBlock->SizeOfBlock == 0)
		{
			break;
		}

		if (pBlock->SizeOfBlock < sizeof(IMAGE_BASE_RELOCATION))
		{
			printf("%-30s\n", "重定位块大小异常");
			return FALSE;
		}

		if (offset + pBlock->SizeOfBlock > relocSize)
		{
			printf("%-30s\n", "重定位块超出重定位表范围");
			return FALSE;
		}

		offset += pBlock->SizeOfBlock;
		blockCount++;
	}

	printf("%-30s\n", "原始重定位表信息");
	printf("%-30s0x%X\n", "原重定位表RVA:", oldRelocRva);
	printf("%-30s0x%X\n", "原重定位表FOA:", oldRelocFoa);
	printf("%-30s0x%X\n", "重定位表大小:", relocSize);
	printf("%-30s%d\n", "重定位块数量:", blockCount);

	// 创建新节区存放重定位表
	char sectionName[] = ".reloc2";

	DWORD sectionCharacteristics =
		IMAGE_SCN_CNT_INITIALIZED_DATA |
		IMAGE_SCN_MEM_READ |
		IMAGE_SCN_MEM_DISCARDABLE;

	printf("%-30s\n", "正在创建新节区用于存放重定位表");

	if (!AddSection(pe, sectionName, relocSize, sectionCharacteristics))
	{
		printf("%-30s\n", "创建新节区失败");
		return FALSE;
	}

	// AddSection 内部可能 realloc，所以这里必须重新定位指针
	pe->dosHeader = (PIMAGE_DOS_HEADER)pe->buffer;
	pe->ntHeaders = (PIMAGE_NT_HEADERS)(pe->buffer + pe->dosHeader->e_lfanew);
	pe->sectionHeader = (PIMAGE_SECTION_HEADER)IMAGE_FIRST_SECTION(pe->ntHeaders);

	pRelocDir =
		&pe->ntHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC];

	PIMAGE_SECTION_HEADER newSection =
		&pe->sectionHeader[pe->ntHeaders->FileHeader.NumberOfSections - 1];

	DWORD newRelocRva = newSection->VirtualAddress;
	DWORD newRelocFoa = newSection->PointerToRawData;

	if ((ULONGLONG)newRelocFoa + relocSize > pe->size)
	{
		printf("%-30s\n", "新节区空间不足");
		return FALSE;
	}

	// 重新生成旧重定位表指针
	// 注意：AddSection 可能 realloc，但 FOA 不变，所以用新的 pe->buffer + oldRelocFoa
	PBYTE oldRelocData = pe->buffer + oldRelocFoa;
	PBYTE newRelocData = pe->buffer + newRelocFoa;

	// 复制重定位表到新节区
	memcpy(newRelocData, oldRelocData, relocSize);

	// 更新数据目录，让加载器使用新的重定位表
	pRelocDir->VirtualAddress = newRelocRva;
	pRelocDir->Size = relocSize;

	// 如果原来错误地设置了“重定位已剥离”，这里清掉
	pe->ntHeaders->FileHeader.Characteristics &= ~IMAGE_FILE_RELOCS_STRIPPED;

	// 修改 PE 后原校验和无效
	pe->ntHeaders->OptionalHeader.CheckSum = 0;

	pe->isModified = TRUE;

	printf("%-30s\n", "重定位表已移动到新节区");
	printf("%-30s%s\n", "新节区名称:", sectionName);
	printf("%-30s0x%X\n", "新重定位表RVA:", newRelocRva);
	printf("%-30s0x%X\n", "新重定位表FOA:", newRelocFoa);
	printf("%-30s0x%X\n", "重定位表大小:", relocSize);

	return TRUE;
}


```
