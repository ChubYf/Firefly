---
title: tea系列
published: 2026-07-15
description: tea系列加解密算法实现
image: ./cover.jpg
tags: [逆向]
category: Crypto
draft: false
---

## 加密脚本
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

// xxtea所需核心加密宏
#define MX (((z >> 5) ^ (y << 2)) + ((y >> 3) ^ (z << 4))) ^ ((sum ^ y) + (key[(p & 3) ^ e] ^ z))

void tea_encrypt(uint32_t* v, uint32_t* k)
{
	uint32_t v0 = v[0], v1 = v[1];
	uint32_t sum = 0;
	uint32_t delta = 0x9e3779b9;

	for (int i = 0; i < 32; i++)
	{
		sum += delta;
		v0 += ((v1 << 4) + k[0]) ^ (v1 + sum) ^ ((v1 >> 5) + k[1]);
		v1 += ((v0 << 4) + k[2]) ^ (v0 + sum) ^ ((v0 >> 5) + k[3]);
	}

	v[0] = v0;
	v[1] = v1;

}

void xtea_encrypt(uint32_t* v, uint32_t* k)
{
	uint32_t v0 = v[0], v1 = v[1];
	uint32_t sum = 0;
	uint32_t delta = 0x9e3779b9;

	for (int i = 0; i < 32; i++)
	{
		v0 += (((v1 << 4) ^ (v1 >> 5)) + v1) ^ (sum + k[sum & 3]);
		sum += delta;
		v1 += (((v0 << 4) ^ (v0 >> 5)) + v0) ^ (sum + k[(sum >> 11) & 3]);
	}

	v[0] = v0;
	v[1] = v1;

}

void xxtea_encrypt(uint32_t* v, int n, uint32_t* key)
{
	// z: 最后一个数据, y: 第一个数据, sum: 累加和, e: 密钥索引, n 数据长度(输入的4字节数据的个数)
	uint32_t z = v[n - 1], y = v[0], sum = 0, e;
	uint32_t delta = 0x9e3779b9;
	uint32_t p, q;

	q = 6 + 52 / n; // 计算轮数

	// 核心加密循环
	while (q > 0)
	{
		sum += delta;
		e = (sum >> 2) & 3; // e用来随机化key索引的选取

		/* 对每个数据字进行加密: p为当前加密数据索引，z为前一个加密数据索引，y为下一个加密数据索引 */
		for (p = 0; p < n - 1; p++)
		{
			y = v[p + 1];
			v[p] += MX;
			z = v[p];
		}

		y = v[0];
		v[n - 1] += MX;
		z = v[n - 1];

		q--;
	}
}

int main()
{
	uint8_t inputBuffer[0xff] = { 0 };
	uint8_t key[16] = { 0x7c, 0x29, 0x16, 0x58, 0x70, 0xd2, 0x81, 0x45, 0x00, 0xe6, 0xc5, 0x0e, 0x03, 0x4e, 0x94, 0x86 };
	uint8_t ciphertext[] = { 250,72,161,115,73,155,6,130,244,25,135,201,15,133,129,41,172,207,200,0,19,182,123,162 };

	printf("Please input your flag: ");
	scanf("%s", inputBuffer);

	int input_len = strlen((char*)inputBuffer);
	if (input_len != 24)
	{
		printf("Wrong Length\r\n");
		system("pause");
		return 0;
	}

	/* tea, xtea加密
	int blockCount = input_len / 8;
	for (int i = 0; i < blockCount; i++)
	{
		//tea_encrypt((uint32_t*)&inputBuffer[i * 8], (uint32_t*)key);
		xtea_encrypt((uint32_t*)&inputBuffer[i * 8], (uint32_t*)key);
	}
	*/

	
	// xxtea加密
	int n = input_len / 4;
	xxtea_encrypt((uint32_t*)inputBuffer, n, (uint32_t*)key);
	
	/* 获取密文数据
	for (int i = 0; i < input_len; i++)
	{
		printf("%d,",inputBuffer[i]);
	}
	*/

	for (int i = 0; i < input_len; i++)
	{
		if (ciphertext[i] != inputBuffer[i])
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

tea加密：blackmanba{1_L0v3_tea!!}
xtea加密：blackmanba{1_L0v3_xtea!}
xxtea加密：blackmanba{1_L0v3_xxtea}


# 解密脚本

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

// xxtea所需核心加密宏
#define MX (((z >> 5) ^ (y << 2)) + ((y >> 3) ^ (z << 4))) ^ ((sum ^ y) + (key[(p & 3) ^ e] ^ z))

void tea_decrypt(uint32_t* v, uint32_t* k)
{

	uint32_t v0 = v[0], v1 = v[1];
	uint32_t delta = 0x9e3779b9;
	uint32_t sum = delta * 32;

	for (int i = 0; i < 32; i++)
	{
		v1 -= ((v0 << 4) + k[2]) ^ (v0 + sum) ^ ((v0 >> 5) + k[3]);
		v0 -= ((v1 << 4) + k[0]) ^ (v1 + sum) ^ ((v1 >> 5) + k[1]);
		sum -= delta;
	}

	v[0] = v0;
	v[1] = v1;


}

void xtea_decrypt(uint32_t* v, uint32_t* k)
{
	uint32_t v0 = v[0], v1 = v[1];
	uint32_t delta = 0x9e3779b9;
	uint32_t sum = 32 * delta;

	for (int i = 0; i < 32; i++)
	{
		v1 -= (((v0 << 4) ^ (v0 >> 5)) + v0) ^ (sum + k[(sum >> 11) & 3]);
		sum -= delta;
		v0 -= (((v1 << 4) ^ (v1 >> 5)) + v1) ^ (sum + k[sum & 3]);
	}

	v[0] = v0;
	v[1] = v1;
}

void xxtea_decrypt(uint32_t* v, int n,uint32_t* key)
{
	// z: 最后一个数据, y: 第一个数据, sum: 累加和, e: 密钥索引, n 数据长度(输入的4字节数据的个数)
	uint32_t z = v[n - 1], y = v[0], sum = 0, e;
	uint32_t delta = 0x9e3779b9;
	uint32_t p, q;

	q = 6 + 52 / n; 
	sum = q * delta;

	while (q > 0)
	{
		e = (sum >> 2) & 3; 

		// p为当前解密数据  z指向前一个数据 y指向后一个数据
		for (p = n - 1; p > 0; p--)
		{
			z = v[p - 1]; 
			v[p] -= MX;   
			y = v[p];	 
		}
		
		
		z = v[n - 1];
		v[0] -= MX;
		y = v[0];

		sum -= delta;
		q--;
	}
}

int main()
{
	uint8_t key[16] = { 0x7c, 0x29, 0x16, 0x58, 0x70, 0xd2, 0x81, 0x45, 0x00, 0xe6, 0xc5, 0x0e, 0x03, 0x4e, 0x94, 0x86 };
	uint8_t ciphertext[] = { 250,72,161,115,73,155,6,130,244,25,135,201,15,133,129,41,172,207,200,0,19,182,123,162 };

	int cipher_len = sizeof(ciphertext) / sizeof(ciphertext[0]);
	if (cipher_len % 8 != 0)
	{
		printf("理论上密文长度满足8的整数倍\r\n");
		return 0;
	}

	/* tea,xtea解密:
	int blockCount = cipher_len / 8;
	for (int i = 0; i < blockCount; i++)
	{
		//tea_decrypt((uint32_t*)&ciphertext[i * 8],(uint32_t*)key);
		xtea_decrypt((uint32_t*)&ciphertext[i * 8], (uint32_t*)key);
	}
	
	*/
	
	// xxtea
	int n = cipher_len / 4;
	xxtea_decrypt((uint32_t*)ciphertext, n, (uint32_t*)key);


	printf("%s\r\n", ciphertext);



	return 0;
}
```