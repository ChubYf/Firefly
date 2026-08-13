---
title: rc4
published: 2026-07-04
description: rc4加解密算法实现
image: ./cover.jpg
tags: [逆向]
category: Crypto
draft: false
---


# rc4加密
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void rc4_init(unsigned char* s, unsigned char* key, int key_len)
{
	unsigned char k[256] = { 0 };
	int j = 0;
	int temp = 0;

	// 初始化s盒和k盒
	for (int i = 0; i < 256; i++)
	{
		s[i] = i; // s盒: 0 - 256
		k[i] = key[i % key_len]; // k盒: 循环填充密钥
	}

	// 将s盒数据顺序打乱
	for (int i = 0; i < 256; i++)
	{
		j = (j + k[i] + s[i]) % 256;
		temp = s[i];
		s[i] = s[j];
		s[j] = temp;
	}

}

void rc4_crypto(unsigned char* s, unsigned char* input, int input_len)
{
	int i = 0, j = 0, t = 0;
	unsigned char temp = 0;

	// 明文循环异或s盒数据
	for (int k = 0; k < input_len; k++)
	{
		// 进一步混淆s盒数据顺序
		i = (i + 1) % 256;
		j = (j + s[i]) % 256;

		temp = s[i];
		s[i] = s[j];
		s[j] = temp;

		t = (s[i] + s[j]) % 256;

		// 关键加密
		input[k] ^= s[t];
	}
}

int main()
{
	unsigned char sBox[256] = { 0 }; // s盒
	unsigned char key[] = { 98,108,97,99,107,109,97,110,98,97 }; // 密钥
	unsigned char inputBuffer[0xff] = { 0 }; // 输入缓冲区
	unsigned char entext[] = { 0xF2, 0x6A, 0xD1, 0xFE, 0xF2, 0x67, 0x3C, 0xDF, 0xA0, 0x36, 0x44, 0x5E, 0x24, 0x78, 0x34, 0x7E, 0xF6, 0xB3, 0x20, 0x29, 0x8A, 0x78, 0xBB, 0xE4 };// 密文

	printf("Please input your flag: ");
	scanf("%s", inputBuffer);

	int input_length = strlen((char*)inputBuffer);


	if (input_length != 24)
	{
		printf("wrong length\r\n");
		system("pause");
		return 0;
	}

	int key_length = sizeof(key) / sizeof(key[0]); // 计算密钥长度
	rc4_init(sBox, key, key_length); // rc4初始化
	rc4_crypto(sBox, inputBuffer, input_length); // rc4加密

	for (int i = 0; i < input_length; i++)
	{
		if (inputBuffer[i] != entext[i])
		{
			printf("Wrong\r\n");
			system("pause");
			return 0;
		}
	}

	printf("Correct\r\n");
	system("pause");
	return 0;
}
```



# 解密脚本
rc4加解密使用同一套算法, 可以用上面的加密算法再加密一次获得明文。
这里使用python解密
```python

def rc4_init(key):

    # 初始化s盒

    s = []

    for i in range(256):

        s.append(i)

    # 初始化k盒

    k = []

    key_len = len(key)

    for i in range(256):

        k.append(key[i % key_len])

    # 混淆s盒数据顺序

    j = 0

    for i in range(256):

        j = (j + s[i] + k[i]) % 256

        s[i], s[j] = s[j], s[i]

  

    return s #返回s盒

  
  

def rc4_decrypt(ciphertext, key):

    #rc4初始化

    s = rc4_init(key)

  

    plaintext = [] # 存储明文

    i = j = 0

    ciphertext_len = len(ciphertext) # 计算密文长度

    # 解密

    for k in range(ciphertext_len):

        i = (i + 1) % 256

        j = (j + s[i]) % 256

  

        s[i], s[j] = s[j], s[i]

        t = (s[i] + s[j]) % 256

  

        plaintext.append(ciphertext[k] ^ s[t])

  

    return bytes(plaintext)

  

def main():

    key = bytes([98, 108, 97, 99, 107, 109, 97, 110, 98, 97])

    ciphertext = bytes.fromhex("F26AD1FEF2673CDFA036445E2478347EF6B320298A78BBE4")

  

    plaintext = rc4_decrypt(ciphertext,key)

  

    print(plaintext)

    return

  
  

if __name__ == "__main__":

    main()

```

