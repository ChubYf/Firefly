---
title: base64
published: 2026-06-21
description: base64加解密算法实现
image: ./cover.jpg
tags: [逆向]
category: Crypto
draft: false
---

# base64
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

const char base64_table[] = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";

void Base64_encode(char* input, char* output, int length)
{
	int pad_num = 0; //需要填充的'='个数
	int in_len = 0;  //重新计算输入数据的长度(加上'=')
	if (length % 3 != 0)
	{
		pad_num = 3 - length % 3; //计算需要填充的'='个数
		in_len = length + pad_num; //重新计算输入数据的长度(加上'=')
	}
	else
	{
		in_len = length; //刚好为3的倍数，无需填充'='
	}

	//output存储base64编码后的数据, 用i遍历input数组，j遍历output数组
	for (int i = 0, j = 0; i < in_len; i += 3, j += 4)
	{
		// 处理填充（最后一块不足3字节时）
		if (i == in_len - 3 && pad_num != 0)
		{
			// 填充'='个数为1
			if (pad_num == 1)
			{
				output[j] = base64_table[(input[i] >> 2) & 0x3f];
				output[j + 1] = base64_table[((input[i] << 4) | (input[i + 1] >> 4)) & 0x3f];
				output[j + 2] = base64_table[(input[i + 1] << 2) & 0x3f];
				output[j + 3] = '=';
			}
			// 填充'='个数为2
			else if (pad_num == 2)
			{
				output[j] = base64_table[(input[i] >> 2) & 0x3f];
				output[j + 1] = base64_table[(input[i] << 4) & 0x3f];
				output[j + 2] = '=';
				output[j + 3] = '=';
			}
		}
		//正常情况
		else
		{
			output[j] = base64_table[(input[i] >> 2) & 0x3f];
			output[j + 1] = base64_table[((input[i] << 4) | (input[i + 1] >> 4)) & 0x3f];
			output[j + 2] = base64_table[((input[i + 1] << 2) | (input[i + 2] >> 6)) & 0x3f];
			output[j + 3] = base64_table[input[i + 2] & 0x3f];
		}
	}

}

int main()
{
	char outputBuffer[0xff] = { 0 };
	char inputBuffer[0xff] = { 0 };
	char flag[] = "YmxhY2ttYW5iYXtCYXNlNjRfZW5jb2RlX2lzX2V6IX0=";

	printf("Please input your flag: ");
	scanf("%s", inputBuffer);

	int length = strlen(inputBuffer);


	if (length != 32)
	{
		printf("flag length wrong\r\n");
		system("pause");
		return 0;
	}

	
	

	Base64_encode(inputBuffer, outputBuffer, length);

	if (strcmp(outputBuffer, flag) == 0)
	{
		printf("Correct\r\n");
	}
	else
	{
		printf("Wrong\r\n");
	}
	system("pause");
	return 0;
}
```
flag: blackmanba{Base64_encode_is_ez!}


# base64变表

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

const char base64_table[] = "RSTUVWLbcdefghiMNOPrstuvQXYZajCklmnEFGHIJKwxyzABDopq0123456789+/";

void Base64_encode(char* input, char* output, int length)
{
	int pad_num = 0;
	int in_len = 0;
	if (length % 3 != 0)
	{
		pad_num = 3 - length % 3;
		in_len = length + pad_num;
	}
	else
	{
		in_len = length;
	}

	
	for (int i = 0, j = 0; i < in_len; i += 3, j += 4)
	{
		if (i == in_len - 3 && pad_num != 0)
		{
			if (pad_num == 1)
			{
				output[j] = base64_table[(input[i] >> 2) & 0x3f];
				output[j + 1] = base64_table[((input[i] << 4) | (input[i + 1] >> 4)) & 0x3f];
				output[j + 2] = base64_table[(input[i + 1] << 2) & 0x3f];
				output[j + 3] = '=';
			}
			else if (pad_num == 2)
			{
				output[j] = base64_table[(input[i] >> 2) & 0x3f];
				output[j + 1] = base64_table[(input[i] << 4) & 0x3f];
				output[j + 2] = '=';
				output[j + 3] = '=';
			}
		}
		else
		{
			output[j] = base64_table[(input[i] >> 2) & 0x3f];
			output[j + 1] = base64_table[((input[i] << 4) | (input[i + 1] >> 4)) & 0x3f];
			output[j + 2] = base64_table[((input[i + 1] << 2) | (input[i + 2] >> 6)) & 0x3f];
			output[j + 3] = base64_table[input[i + 2] & 0x3f];
		}
	}

}

int main()
{
	char outputBuffer[0xff] = { 0 };
	char inputBuffer[0xff] = { 0 };
	char flag[] = "QHomQ2zzQu5nQvzTQvhGhEOkru9FYuXKXuOsQudyXv0=";

	printf("Please input your flag: ");
	scanf("%s", inputBuffer);

	int length = strlen(inputBuffer);


	if (length != 32)
	{
		printf("flag length wrong\r\n");
		system("pause");
		return 0;
	}

	
	

	Base64_encode(inputBuffer, outputBuffer, length);

	if (strcmp(outputBuffer, flag) == 0)
	{
		printf("Correct\r\n");
	}
	else
	{
		printf("Wrong\r\n");
	}
	system("pause");
	return 0;
}
```
flag: blackmanba{Base64_ModifiedTable}


# 解密脚本

python实现:
## 1.不依赖python库
适用于标准base64和魔改base64。使用时将base64表和密文更改即可
```python
base64_table = b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"

  
  

def base64_decode(encrypt_data):

  

    # 移除填充, 记录填充数

    padding = encrypt_data.count(b'=')

    encrypt_data = encrypt_data.rstrip(b'=')

  

    decrypted_data = "" #存储解密后的字符串

    bin_data = "" #存储decrypted_data的二进制字符串

  

    for data in encrypt_data:

        index_value = base64_table.index(data) #计算密文每个字母在base64表中的索引值

        bin_data += f"{index_value:0>6b}" #将索引值转化为二进制并以6bits形式存储

  

    # 根据填充数, 删除多余的位

    if padding:

        bin_data = bin_data[: -padding * 2]

  

    #将二进制数据转化为字符串

    for i in range(0, len(bin_data), 8):

        decrypted_data += chr(int(bin_data[i:i+8], 2))

  

    return decrypted_data

  
  

def main() :

    encrypt_data = b"YmxhY2ttYW5iYXtCYXNlNjRfZW5jb2RlX2lzX2V6IX0="

    decrypted_data = base64_decode(encrypt_data)

  

    print(decrypted_data)

    return

  

if __name__ == "__main__":

    main()
```


## 2.依赖python库

### 标准base64情况
非常简单，只需导入base64并使用base64提供的b64decode解码

```python
import base64

  

encoded_str = "YmxhY2ttYW5iYXtCYXNlNjRfZW5jb2RlX2lzX2V6IX0="

decoded_bytes = base64.b64decode(encoded_str)

decoded_str = decoded_bytes.decode('utf-8')

  

print(decoded_str)
```

### 魔改base64表

使用字符映射转化
```python
import base64

  

standard_table = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/" # 标准base64表

modified_table = "RSTUVWLbcdefghiMNOPrstuvQXYZajCklmnEFGHIJKwxyzABDopq0123456789+/" # 魔改base64表

encoded_str = "QHomQ2zzQu5nQvzTQvhGhEOkru9FYuXKXuOsQudyXv0=" # 密文

  

map = str.maketrans(modified_table, standard_table) # 用str类中的maketrans建立映射，注意第一个参数是需要映射的字符串，第二个参数是映射的目标

map_text = encoded_str.translate(map) # 映射实现替换密文，替换前是base64换表加密，替换后则是base64标准表加密

  

decoded_bytes = base64.b64decode(map_text)

decoded_str = decoded_bytes.decode('utf-8')

  

print(decoded_str)
```

