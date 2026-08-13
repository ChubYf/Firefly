---
title: aes
published: 2026-07-11
description: aes加解密算法实现
image: ./cover.jpg
tags: [逆向]
category: Crypto
draft: false
---


# aes128ECB模式
## 加密脚本

```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>

// Subbytes所用S盒
uint8_t Sbox[256] = {
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,
    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,
    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,
    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,
    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,
    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,
    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,
    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,
    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,
    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,
    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,
    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,
    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,
    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,
    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,
    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16
};

// 轮常数 (前10个)
uint32_t Rcon[10] = { 
    0x01000000, 0x02000000,
    0x04000000, 0x08000000,
    0x10000000, 0x20000000,
    0x40000000, 0x80000000,
    0x1b000000, 0x36000000 };

// 列混合所用矩阵
uint8_t colM[4][4] =
{
    2, 3, 1, 1,
    1, 2, 3, 1,
    1, 1, 2, 3,
    3, 1, 1, 2
};

// 将4字节串转化为4字节整数(大端序)
uint32_t BytesToIntB(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) |
        ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) |
        ((uint32_t)bytes[3]);
}

// 列混合所用辅助函数
//GF(2⁸) 乘法实现:
uint8_t GFMul2(uint8_t s)
{
    uint8_t result = s << 1;
    if (s & 0x80)
    {
        result ^= 0x1b;
    }
    return result;
}

uint8_t GFMul3(uint8_t s)
{
    return GFMul2(s) ^ s;
}
uint8_t GFMul4(uint8_t s)
{
    return GFMul2(GFMul2(s));
}
uint8_t GFMul8(uint8_t s)
{
    return GFMul2(GFMul4(s));
}
uint8_t GFMul9(uint8_t s)
{
    return GFMul8(s) ^ s;
}
uint8_t GFMul11(uint8_t s)
{
    return GFMul9(s) ^ GFMul2(s);
}
uint8_t GFMul12(uint8_t s)
{
    return GFMul8(s) ^ GFMul4(s);
}
uint8_t GFMul13(uint8_t s)
{
    return GFMul12(s) ^ s;
}
uint8_t GFMul14(uint8_t s)
{
    return GFMul12(s) ^ GFMul2(s);
}

uint8_t GFMul(int n, uint8_t s)
{
    switch (n)
    {
    case 1:
        return s;
    case 2:
        return GFMul2(s);
    case 3:
        return GFMul3(s);
    case 4:
        return GFMul4(s);
    case 8:
        return GFMul8(s);
    case 9:
        return GFMul9(s);
    case 11:
        return GFMul11(s);
    case 12:
        return GFMul12(s);
    case 13:
        return GFMul13(s);
    case 14:
        return GFMul14(s);
    default:
        printf("not found target GFMul number\r\n");
        system("pause");
        exit(EXIT_FAILURE);
    }
}

// 将4字节整数数据转化为字节串
void IntToBytesB(uint8_t* bytes, uint32_t intData)
{
    bytes[0] = (intData >> 24) & 0xff;
    bytes[1] = (intData >> 16) & 0xff;
    bytes[2] = (intData >> 8) & 0xff;
    bytes[3] = intData & 0xff;
}

// 密钥扩展中的T函数
uint32_t T(uint32_t num, uint32_t round)
{

    // 子循环: 将1个字中的4个字节循环左移1个字节。
    uint8_t t1 = (num >> 16) & 0xff;
    uint8_t t2 = (num >> 8) & 0xff;
    uint8_t t3 = num & 0xff;
    uint8_t t4 = (num >> 24) & 0xff;

    // 字节代换: 用S盒进行字节代换
    t1 = Sbox[t1];
    t2 = Sbox[t2];
    t3 = Sbox[t3];
    t4 = Sbox[t4];

    // 转化为4字节大端序整数
    uint32_t result = 
        ((uint32_t)t1 << 24) |
        ((uint32_t)t2 << 16) |
        ((uint32_t)t3 << 8) |
        ((uint32_t)t4);

    // 轮常量异或: 与同轮常量Rcon[round / 4 - 1]异或
    result ^= Rcon[round / 4 - 1];

    return result;
}

// 密钥扩展
void key_expansion(uint8_t* key, uint32_t* rk)
{
    // 将16字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 4; i++)
    {
        rk[i] = BytesToIntB(&key[i*4]);
    }

    // 扩展10轮密钥
    for (int i = 4; i < 44; i++)
    {
        // 如果i为4的整数倍需要进行T函数处理
        if (i % 4 == 0)
        {
            rk[i] = rk[i - 4] ^ T(rk[i - 1], i);
        }
        else
        {
            rk[i] = rk[i - 4] ^ rk[i - 1];
        }
    }

}

// 字节代换
void SubBytes(uint8_t inputBuffer[4][4])
{
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] = Sbox[inputBuffer[i][j]];
        }
    }
}

// 行移位
void ShiftRows(uint8_t inputbuffer[4][4])
{
    uint8_t temp[4] = { 0 };
    // 第一行不移位

    // 第二行左移1位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputbuffer[1][i];
    }
    inputbuffer[1][0] = temp[1];
    inputbuffer[1][1] = temp[2];
    inputbuffer[1][2] = temp[3];
    inputbuffer[1][3] = temp[0];

    // 第三行左移2位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputbuffer[2][i];
    }
    inputbuffer[2][0] = temp[2];
    inputbuffer[2][1] = temp[3];
    inputbuffer[2][2] = temp[0];
    inputbuffer[2][3] = temp[1];

    // 第四行行左移3位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputbuffer[3][i];
    }
    inputbuffer[3][0] = temp[3];
    inputbuffer[3][1] = temp[0];
    inputbuffer[3][2] = temp[1];
    inputbuffer[3][3] = temp[2];

}

// 列混合
void MixColumns(uint8_t inputBuffer[4][4])
{
    uint8_t tempArray[4][4];
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            tempArray[i][j] = inputBuffer[i][j];
        }
    }

    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] =
                GFMul(colM[i][0], tempArray[0][j]) ^
                GFMul(colM[i][1], tempArray[1][j]) ^
                GFMul(colM[i][2], tempArray[2][j]) ^
                GFMul(colM[i][3], tempArray[3][j]);
        }
    }

}

// 轮密钥加
void AddRoundKey(uint8_t inputBuffer[4][4], uint32_t* rk , int round)
{
    // 存储密钥
    uint8_t tempArray[4];

    for (int i = 0; i < 4; i++)
    {
        // 将4字节整数转化为字节串
        IntToBytesB(tempArray, rk[round * 4 + i]);

        for (int j = 0; j < 4; j++)
        {
            inputBuffer[j][i] ^= tempArray[j];
        }
    }
}

void aes128Encrypt(uint8_t* inputBuffer, uint32_t* rk, uint8_t* encryptData)
{
    // 将输入转化为明文矩阵
    uint8_t inputArray[4][4];
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputArray[j][i] = inputBuffer[i * 4 + j];
        }
    }

    // 初始轮密钥加
    AddRoundKey(inputArray, rk, 0);

    // 9轮加密循环
    for (int i = 1; i < 10; i++)
    {
        SubBytes(inputArray);
        ShiftRows(inputArray);
        MixColumns(inputArray);
        AddRoundKey(inputArray, rk, i);
    }

    // 最后一轮加密，无MixColumns
    SubBytes(inputArray);
    ShiftRows(inputArray);
    AddRoundKey(inputArray, rk, 10);

    // 将密文矩阵转化为字节串
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            encryptData[i * 4 + j] = inputArray[j][i];
        }
    }

}

void aesEncrypt(uint8_t* inputBuffer, int input_len, uint8_t* key, uint8_t* encryptData)
{

    uint32_t rk[44] = { 0 };
    key_expansion(key, rk);


    int blockSize = input_len / 16;
    for (int i = 0; i < blockSize; i++)
    {
        aes128Encrypt(&inputBuffer[i * 16], rk, &encryptData[i * 16]);
    }

}

int main()
{
    // 固定16字节密钥
    uint8_t key[16] = { 0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A };
    uint8_t encryptData[0xff] = { 0 }; // 存放加密后的数据
    uint8_t ciphertext[] = { 0x25,  0x1C,  0xCC,  0xA0,  0xD8,  0x56,  0x6F,  0x43,  0xC1,  0x5A,  0x1F,  0xD7,  0x5A,  0x21,  0x1A,  0x66,  0x6D,  0xE5,  0x50,  0xBE,  0xB6,  0xE8,  0x17,  0x57,  0x01,  0xCD,  0x4A,  0x07,  0x7B,  0x21,  0xA0,  0x20 };// 密文

    uint8_t inputBuffer[0xff] = { 0 };
    printf("Please input your flag: ");
    scanf("%s", inputBuffer);

    int input_len = strlen((char*)inputBuffer);
    if (input_len != 32)
    {
        printf("wrong length\r\n");
        system("pause");
        return 0;
    }
    
    aesEncrypt(inputBuffer, input_len, key, encryptData);


    for (int i = 0; i < 32; i++)
    {
        if (ciphertext[i] != encryptData[i])
        {
            printf("Wrong\r\n");
            system("pause");
            return 0;
        }
    }

    printf("Right\r\n");

    system("pause");
	return 0;
}
```

blackmanba{A3s_15_D3l1c10us_w0w}

## 解密脚本
### C语言实现
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

// S盒
uint8_t Sbox[256] = {
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,
    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,
    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,
    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,
    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,
    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,
    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,
    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,
    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,
    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,
    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,
    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,
    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,
    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,
    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,
    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16
};

// 逆S盒
uint8_t invSbox[256] =
{
    0
};

// 轮常数 (前10个)
uint32_t Rcon[10] = {
    0x01000000, 0x02000000,
    0x04000000, 0x08000000,
    0x10000000, 0x20000000,
    0x40000000, 0x80000000,
    0x1b000000, 0x36000000 };

// 列混合所用矩阵
uint8_t colM[4][4] =
{
    2, 3, 1, 1,
    1, 2, 3, 1,
    1, 1, 2, 3,
    3, 1, 1, 2
};

// 逆矩阵
uint8_t inv_colM[4][4] =
{
    0xe, 0xb, 0xd, 0x9,
    0x9, 0xe, 0xb, 0xd,
    0xd, 0x9, 0xe, 0xb,
    0xb, 0xd, 0x9, 0xe
};

// 将4字节串转化为4字节整数(大端序)
uint32_t BytesToIntB(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) |
        ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) |
        ((uint32_t)bytes[3]);
}

// 列混合所用辅助函数
//GF(2⁸) 乘法实现:
uint8_t GFMul2(uint8_t s)
{
    uint8_t result = s << 1;
    if (s & 0x80)
    {
        result ^= 0x1b;
    }
    return result;
}

uint8_t GFMul3(uint8_t s)
{
    return GFMul2(s) ^ s;
}
uint8_t GFMul4(uint8_t s)
{
    return GFMul2(GFMul2(s));
}
uint8_t GFMul8(uint8_t s)
{
    return GFMul2(GFMul4(s));
}
uint8_t GFMul9(uint8_t s)
{
    return GFMul8(s) ^ s;
}
uint8_t GFMul11(uint8_t s)
{
    return GFMul9(s) ^ GFMul2(s);
}
uint8_t GFMul12(uint8_t s)
{
    return GFMul8(s) ^ GFMul4(s);
}
uint8_t GFMul13(uint8_t s)
{
    return GFMul12(s) ^ s;
}
uint8_t GFMul14(uint8_t s)
{
    return GFMul12(s) ^ GFMul2(s);
}

uint8_t GFMul(int n, uint8_t s)
{
    switch (n)
    {
    case 1:
        return s;
    case 2:
        return GFMul2(s);
    case 3:
        return GFMul3(s);
    case 4:
        return GFMul4(s);
    case 8:
        return GFMul8(s);
    case 9:
        return GFMul9(s);
    case 11:
        return GFMul11(s);
    case 12:
        return GFMul12(s);
    case 13:
        return GFMul13(s);
    case 14:
        return GFMul14(s);
    default:
        printf("not found target GFMul number\r\n");
        system("pause");
        exit(EXIT_FAILURE);
    }
}

// 将4字节整数数据转化为字节串
void IntToBytesB(uint8_t* bytes, uint32_t intData)
{
    bytes[0] = (intData >> 24) & 0xff;
    bytes[1] = (intData >> 16) & 0xff;
    bytes[2] = (intData >> 8) & 0xff;
    bytes[3] = intData & 0xff;
}

// 密钥扩展中的T函数
uint32_t T(uint32_t num, uint32_t round)
{

    // 子循环: 将1个字中的4个字节循环左移1个字节。
    uint8_t t1 = (num >> 16) & 0xff;
    uint8_t t2 = (num >> 8) & 0xff;
    uint8_t t3 = num & 0xff;
    uint8_t t4 = (num >> 24) & 0xff;

    // 字节代换: 用S盒进行字节代换
    t1 = Sbox[t1];
    t2 = Sbox[t2];
    t3 = Sbox[t3];
    t4 = Sbox[t4];

    // 转化为4字节大端序整数
    uint32_t result =
        ((uint32_t)t1 << 24) |
        ((uint32_t)t2 << 16) |
        ((uint32_t)t3 << 8) |
        ((uint32_t)t4);

    // 轮常量异或: 与同轮常量Rcon[round / 4 - 1]异或
    result ^= Rcon[round / 4 - 1];

    return result;
}

// 密钥扩展
void key_expansion(uint8_t* key, uint32_t* rk)
{
    // 将16字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 4; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展10轮密钥
    for (int i = 4; i < 44; i++)
    {
        // 如果i为4的整数倍需要进行T函数处理
        if (i % 4 == 0)
        {
            rk[i] = rk[i - 4] ^ T(rk[i - 1], i);
        }
        else
        {
            rk[i] = rk[i - 4] ^ rk[i - 1];
        }
    }

}

// 生成逆S盒
void GenerateInvSbox()
{
    for (int i = 0; i < 256; i++)
    {
        invSbox[Sbox[i]] = i;
    }
}

// 逆字节代换
void invSubBytes(uint8_t inputBuffer[4][4])
{
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] = invSbox[inputBuffer[i][j]];
        }
    }
}

// 逆行移位
void invShiftRow(uint8_t inputBuffer[4][4])
{
    uint8_t temp[4] = { 0 };
    // 第一行不移位

    // 第二行右移1位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[1][i];
    }
    inputBuffer[1][0] = temp[3];
    inputBuffer[1][1] = temp[0];
    inputBuffer[1][2] = temp[1];
    inputBuffer[1][3] = temp[2];

    // 第三行右移2位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[2][i];
    }
    inputBuffer[2][0] = temp[2];
    inputBuffer[2][1] = temp[3];
    inputBuffer[2][2] = temp[0];
    inputBuffer[2][3] = temp[1];

    // 第四行右移3位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[3][i];
    }
    inputBuffer[3][0] = temp[1];
    inputBuffer[3][1] = temp[2];
    inputBuffer[3][2] = temp[3];
    inputBuffer[3][3] = temp[0];

}

// 逆列混合
void invMixColumns(uint8_t inputBuffer[4][4])
{
    uint8_t tempArray[4][4];
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            tempArray[i][j] = inputBuffer[i][j];
        }
    }

    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] =
                GFMul(inv_colM[i][0], tempArray[0][j]) ^
                GFMul(inv_colM[i][1], tempArray[1][j]) ^
                GFMul(inv_colM[i][2], tempArray[2][j]) ^
                GFMul(inv_colM[i][3], tempArray[3][j]);
        }
    }
}

// 加解密所用轮密钥加相同
void AddRoundKey(uint8_t intputBuffer[4][4], uint32_t* rk ,int round)
{
    uint8_t tempArray[4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        IntToBytesB(tempArray, rk[round * 4 + i]);
        for (int j = 0; j < 4; j++)
        {
            intputBuffer[j][i] ^= tempArray[j];
        }
    }
}

void aesDecrypt_128(uint8_t* ciphertext, uint32_t* rk, uint8_t* plaintext)
{
    // 将密文转化为密文矩阵
    uint8_t cipherArray[4][4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            cipherArray[j][i] = ciphertext[i * 4 + j];
        }
    }

    // 从第10轮加密开始逆解密
    AddRoundKey(cipherArray, rk, 10);
    invShiftRow(cipherArray);
    invSubBytes(cipherArray);

    // 从第9轮解密到第1轮
    for (int i = 9; i > 0; i--)
    {
        AddRoundKey(cipherArray, rk, i);
        invMixColumns(cipherArray);
        invShiftRow(cipherArray);
        invSubBytes(cipherArray);
    }

    // 初始轮密钥加
    AddRoundKey(cipherArray, rk, 0);

    // 将明文矩阵存储到明文数组中
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            plaintext[i * 4 + j] = cipherArray[j][i];
        }
    }
}

void aesDecryptECB_128(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext)
{
    // 检查密钥长度是否为16字节
    if (key_len != 16)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[44] = { 0 };
    key_expansion(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_128(&ciphertext[i * 16], rk, &plaintext[i * 16]);
    }

}

int main()
{
    uint8_t key[16] = { 0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A };
    uint8_t ciphertext[32] = { 0x25,  0x1C,  0xCC,  0xA0,  0xD8,  0x56,  0x6F,  0x43,  0xC1,  0x5A,  0x1F,  0xD7,  0x5A,  0x21,  0x1A,  0x66,  0x6D,  0xE5,  0x50,  0xBE,  0xB6,  0xE8,  0x17,  0x57,  0x01,  0xCD,  0x4A,  0x07,  0x7B,  0x21,  0xA0,  0x20 };
    uint8_t plaintext[0xff] = { 0 };

    int key_len = sizeof(key) / sizeof(key[0]);
    int cipher_len = sizeof(ciphertext) / sizeof(ciphertext[0]);

    aesDecryptECB_128(ciphertext, key, cipher_len, key_len, plaintext);
    printf("%s\r\n", plaintext);

	return 0;
}
```

### python实现
```python
# S盒

Sbox = [

    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,

    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,

    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,

    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,

    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,

    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,

    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,

    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,

    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,

    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,

    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,

    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,

    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,

    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,

    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,

    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16

]

  

# 逆S盒

inv_Sbox = [0 for _ in range(256)]

  

# 轮常数

Rcon = [

    0x01000000, 0x02000000,

    0x04000000, 0x08000000,

    0x10000000, 0x20000000,

    0x40000000, 0x80000000,

    0x1b000000, 0x36000000

]

  

# 列混合所用矩阵

colM = [

    2, 3, 1, 1,

    1, 2, 3, 1,

    1, 1, 2, 3,

    3, 1, 1, 2

]

  

# 解密列混合所用逆矩阵

inv_colM = [

    [0xe, 0xb, 0xd, 0x9],

    [0x9, 0xe, 0xb, 0xd],

    [0xd, 0x9, 0xe, 0xb],

    [0xb, 0xd, 0x9, 0xe]

]

  

# 列混合所用辅助函数: GF(2⁸) 乘法实现

def GFMul2(s):

    s = s & 0xff

    result = s << 1

    if (s & 0x80):

        result ^= 0x1b

    return result & 0xff

  

def GFMul3(s):

    s = s & 0xff

    return GFMul2(s) ^ s

  

def GFMul4(s):

    s = s & 0xff

    return GFMul2(GFMul2(s))

  

def GFMul8(s):

    s = s & 0xff

    return GFMul4(GFMul2(s))

  

def GFMul9(s):

    s = s & 0xff

    return GFMul8(s) ^ s

  

def GFMul11(s):

    s = s & 0xff

    return GFMul9(s) ^ GFMul2(s)

  

def GFMul12(s):

    s = s & 0xff

    return GFMul8(s) ^ GFMul4(s)

  

def GFMul13(s):

    s = s & 0xff

    return GFMul12(s) ^ s

  

def GFMul14(s):

    s = s & 0xff

    return GFMul12(s) ^ GFMul2(s)

  

def GFMul(n, s):

    match n:

        case 1:

            return s & 0xff

        case 2:

            return GFMul2(s)

        case 3:

            return GFMul3(s)

        case 4:

            return GFMul4(s)

        case 8:

            return GFMul8(s)

        case 9:

            return GFMul9(s)

        case 11:

            return GFMul11(s)

        case 12:

            return GFMul12(s)

        case 13:

            return GFMul13(s)

        case 14:

            return GFMul14(s)

        case _:

            print("Not found target GFMul value")

            return 0

  

def key_expansion128(key, rk):

    # 将16字节key以4字节大端序初始化存储在rk中

    for i in range(4):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展10轮密钥

    for i in range(4, 44, 1):

        # i为4的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 4 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 4 - 1]

            rk[i] = rk[i - 4] ^ temp

        else :

            rk[i] = rk[i - 4] ^ rk[i - 1]

  

def key_expansion192(key, rk):

    # 将24字节key以4字节大端序初始化存储在rk中

    for i in range(6):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展12轮密钥

    for i in range(6, 52, 1):

        # i为6的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 6 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 6 - 1]

            rk[i] = rk[i - 6] ^ temp

        else :

            rk[i] = rk[i - 6] ^ rk[i - 1]

  

def key_expansion256(key, rk):

    # 将32字节key以4字节大端序初始化存储在rk中

    for i in range(8):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展14轮密钥

    for i in range(8, 60, 1):

        # i为6的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 8 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 8 - 1]

            rk[i] = rk[i - 8] ^ temp

  

        # 256加密特有:i % 8 == 4时只执行字节代换

        elif (i % 8 == 4):

            t0 = Sbox[(rk[i - 1] >> 24) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t2 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t3 = Sbox[(rk[i - 1]) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

  

            rk[i] = rk[i - 8] ^ temp

  

        else :

            rk[i] = rk[i - 8] ^ rk[i - 1]

  

# 生成逆S盒

def generate_inv_Sbox():

    for i in range(256):

        inv_Sbox[Sbox[i]] = i

  

# 逆字节代换

def inv_SubBytes(state):

    for i in range(16):

        state[i] = inv_Sbox[state[i]]

# 逆行移位

def inv_ShiftRow(state):

    # 第一行不移位

  

    # 第二行右移1位

    state[1], state[5], state[9], state[13] = state[13], state[1], state[5], state[9]

  

    # 第三行右移2位

    state[2], state[6], state[10], state[14] = state[10], state[14], state[2], state[6]

  

    # 第四行右移3位

    state[3], state[7], state[11], state[15] = state[7], state[11], state[15], state[3]

  

# 逆列混合

def inv_MixColumns(state):

    temp_array = []

    for i in range(16):

        temp_array.append(state[i])

  

    # 逆矩阵乘密文矩阵

    for i in range(4):

        for j in range(4):

            state[i + j * 4] = (

                GFMul(inv_colM[i][0], temp_array[j * 4 + 0]) ^

                GFMul(inv_colM[i][1], temp_array[j * 4 + 1]) ^

                GFMul(inv_colM[i][2], temp_array[j * 4 + 2]) ^

                GFMul(inv_colM[i][3], temp_array[j * 4 + 3])

            )

  

# 逆轮密钥加

def inv_AddRoundKey(state, round, rk):

    t0 = (int.from_bytes(state[0:4], "big") ^ rk[round * 4]) & 0xffffffff

    t1 = (int.from_bytes(state[4:8], "big") ^ rk[round * 4 + 1]) & 0xffffffff

    t2 = (int.from_bytes(state[8:12], "big") ^ rk[round * 4 + 2]) & 0xffffffff

    t3 = (int.from_bytes(state[12:16], "big") ^ rk[round * 4 + 3]) & 0xffffffff

  

    temp_bytes = t0.to_bytes(4,"big")

    temp_bytes += t1.to_bytes(4,"big")

    temp_bytes += t2.to_bytes(4,"big")

    temp_bytes += t3.to_bytes(4,"big")

  

    for i in range(16):

        state[i] = temp_bytes[i]

  

def aes128_decrypt(state, rk):

  

    # 从第10轮开始解密

    inv_AddRoundKey(state, 10, rk)

    inv_ShiftRow(state)

    inv_SubBytes(state)

  

    # 从第9轮解密到第1轮

    for i in range(9, 0, -1):

        inv_AddRoundKey(state, i, rk)

        inv_MixColumns(state)

        inv_ShiftRow(state)

        inv_SubBytes(state)

    # 解密初始轮密钥加

    inv_AddRoundKey(state, 0, rk)

  
  

def aes128ECB_decrypt(ciphertext, key):

    # 检查密钥长度是否为16字节

    if len(key) != 16:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(44)]

    key_expansion128(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes128_decrypt(state, rk)

        plaintext += state

    print(plaintext)

  
  

def main():

    key = bytes([0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A])

    ciphertext = bytes([0x25,  0x1C,  0xCC,  0xA0,  0xD8,  0x56,  0x6F,  0x43,  0xC1,  0x5A,  0x1F,  0xD7,  0x5A,  0x21,  0x1A,  0x66,  0x6D,  0xE5,  0x50,  0xBE,  0xB6,  0xE8,  0x17,  0x57,  0x01,  0xCD,  0x4A,  0x07,  0x7B,  0x21,  0xA0,  0x20])

    aes128ECB_decrypt(ciphertext, key)

  

if __name__ == "__main__":

    main()
```

# aes192ECB模式

## 加密脚本
```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>

// Subbytes所用S盒
uint8_t Sbox[256] = {
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,
    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,
    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,
    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,
    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,
    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,
    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,
    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,
    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,
    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,
    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,
    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,
    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,
    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,
    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,
    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16
};

// 轮常数 (前10个)
uint32_t Rcon[10] = {
    0x01000000, 0x02000000,
    0x04000000, 0x08000000,
    0x10000000, 0x20000000,
    0x40000000, 0x80000000,
    0x1b000000, 0x36000000 };

// 列混合所用矩阵
uint8_t colM[4][4] =
{
    2, 3, 1, 1,
    1, 2, 3, 1,
    1, 1, 2, 3,
    3, 1, 1, 2
};

// 将4字节串转化为4字节整数(大端序)
uint32_t BytesToIntB(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) |
        ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) |
        ((uint32_t)bytes[3]);
}

// 列混合所用辅助函数
//GF(2⁸) 乘法实现:
uint8_t GFMul2(uint8_t s)
{
    uint8_t result = s << 1;
    if (s & 0x80)
    {
        result ^= 0x1b;
    }
    return result;
}

uint8_t GFMul3(uint8_t s)
{
    return GFMul2(s) ^ s;
}
uint8_t GFMul4(uint8_t s)
{
    return GFMul2(GFMul2(s));
}
uint8_t GFMul8(uint8_t s)
{
    return GFMul2(GFMul4(s));
}
uint8_t GFMul9(uint8_t s)
{
    return GFMul8(s) ^ s;
}
uint8_t GFMul11(uint8_t s)
{
    return GFMul9(s) ^ GFMul2(s);
}
uint8_t GFMul12(uint8_t s)
{
    return GFMul8(s) ^ GFMul4(s);
}
uint8_t GFMul13(uint8_t s)
{
    return GFMul12(s) ^ s;
}
uint8_t GFMul14(uint8_t s)
{
    return GFMul12(s) ^ GFMul2(s);
}

uint8_t GFMul(int n, uint8_t s)
{
    switch (n)
    {
    case 1:
        return s;
    case 2:
        return GFMul2(s);
    case 3:
        return GFMul3(s);
    case 4:
        return GFMul4(s);
    case 8:
        return GFMul8(s);
    case 9:
        return GFMul9(s);
    case 11:
        return GFMul11(s);
    case 12:
        return GFMul12(s);
    case 13:
        return GFMul13(s);
    case 14:
        return GFMul14(s);
    default:
        printf("not found target GFMul number\r\n");
        system("pause");
        exit(EXIT_FAILURE);
    }
}

// 将4字节整数数据转化为字节串
void IntToBytesB(uint8_t* bytes, uint32_t intData)
{
    bytes[0] = (intData >> 24) & 0xff;
    bytes[1] = (intData >> 16) & 0xff;
    bytes[2] = (intData >> 8) & 0xff;
    bytes[3] = intData & 0xff;
}

// 密钥扩展
void key_expansion(uint8_t* key, uint32_t* rk)
{
    // 将24字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 6; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展12轮密钥
    for (int i = 6; i < 52; i++)
    {
        // i为6整数倍
        if (i % 6 == 0)
        {
            // 左移一位，字节代换，异或轮常数
            uint8_t t0 = Sbox[(rk[i - 1] >> 16) & 0xff];
            uint8_t t1 = Sbox[(rk[i - 1] >> 8) & 0xff];
            uint8_t t2 = Sbox[rk[i - 1] & 0xff];
            uint8_t t3 = Sbox[(rk[i - 1] >> 24) & 0xff];

            uint32_t temp =
                (uint32_t)t0 << 24 |
                (uint32_t)t1 << 16 |
                (uint32_t)t2 << 8 |
                (uint32_t)t3;

            temp ^= Rcon[i / 6 - 1];

            rk[i] = rk[i - 6] ^ temp;
        }
        else
        {
            rk[i] = rk[i - 1] ^ rk[i - 6];
        }
    }

}

// 字节代换
void SubBytes(uint8_t inputBuffer[4][4])
{
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] = Sbox[inputBuffer[i][j]];
        }
    }
}

// 行移位
void ShiftRows(uint8_t inputbuffer[4][4])
{
    uint8_t temp[4] = { 0 };
    // 第一行不移位

    // 第二行左移1位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputbuffer[1][i];
    }
    inputbuffer[1][0] = temp[1];
    inputbuffer[1][1] = temp[2];
    inputbuffer[1][2] = temp[3];
    inputbuffer[1][3] = temp[0];

    // 第三行左移2位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputbuffer[2][i];
    }
    inputbuffer[2][0] = temp[2];
    inputbuffer[2][1] = temp[3];
    inputbuffer[2][2] = temp[0];
    inputbuffer[2][3] = temp[1];

    // 第四行行左移3位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputbuffer[3][i];
    }
    inputbuffer[3][0] = temp[3];
    inputbuffer[3][1] = temp[0];
    inputbuffer[3][2] = temp[1];
    inputbuffer[3][3] = temp[2];

}

// 列混合
void MixColumns(uint8_t inputBuffer[4][4])
{
    uint8_t tempArray[4][4];
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            tempArray[i][j] = inputBuffer[i][j];
        }
    }

    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] =
                GFMul(colM[i][0], tempArray[0][j]) ^
                GFMul(colM[i][1], tempArray[1][j]) ^
                GFMul(colM[i][2], tempArray[2][j]) ^
                GFMul(colM[i][3], tempArray[3][j]);
        }
    }

}

// 轮密钥加
void AddRoundKey(uint8_t inputBuffer[4][4], uint32_t* rk, int round)
{
    // 存储密钥
    uint8_t tempArray[4];

    for (int i = 0; i < 4; i++)
    {
        // 将4字节整数转化为字节串
        IntToBytesB(tempArray, rk[round * 4 + i]);

        for (int j = 0; j < 4; j++)
        {
            inputBuffer[j][i] ^= tempArray[j];
        }
    }
}

void aes192Encrypt(uint8_t* inputBuffer, uint32_t* rk, uint8_t* encryptData)
{
    // 将输入转化为明文矩阵
    uint8_t inputArray[4][4];
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputArray[j][i] = inputBuffer[i * 4 + j];
        }
    }

    // 初始轮密钥加
    AddRoundKey(inputArray, rk, 0);

    // 11轮加密循环
    for (int i = 1; i < 12; i++)
    {
        SubBytes(inputArray);
        ShiftRows(inputArray);
        MixColumns(inputArray);
        AddRoundKey(inputArray, rk, i);
    }

    // 最后一轮加密，无MixColumns
    SubBytes(inputArray);
    ShiftRows(inputArray);
    AddRoundKey(inputArray, rk, 12);

    // 将密文矩阵转化为字节串
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            encryptData[i * 4 + j] = inputArray[j][i];
        }
    }

}

void aesEncrypt(uint8_t* inputBuffer, int input_len, uint8_t* key, uint8_t* encryptData)
{
    // aes192模式下轮密钥为(12 + 1) * 4个字
    uint32_t rk[52] = { 0 };
    key_expansion(key, rk);


    int blockSize = input_len / 16;
    for (int i = 0; i < blockSize; i++)
    {
        aes192Encrypt(&inputBuffer[i * 16], rk, &encryptData[i * 16]);
    }

}

int main()
{
    // 固定24字节密钥
    uint8_t key[24] = { 0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38 };
    uint8_t encryptData[0xff] = { 0 }; // 存放加密后的数据
    uint8_t ciphertext[] = { 0x10, 0xAB, 0xC7, 0x3E, 0xD7, 0xA7, 0x16, 0x36, 0xC5, 0x71, 0xB2, 0xBC, 0x38, 0xF5, 0xD9, 0x95, 0x09, 0xB0, 0xC1, 0x56, 0xFC, 0x4D, 0xBB, 0x92, 0x50, 0xC3, 0x85, 0x8F, 0x65, 0x15, 0x9C, 0x45 };// 密文

    uint8_t inputBuffer[0xff] = { 0 };
    printf("Please input your flag: ");
    scanf("%s", inputBuffer);

    int input_len = strlen((char*)inputBuffer);
    if (input_len != 32)
    {
        printf("wrong length\r\n");
        system("pause");
        return 0;
    }

    aesEncrypt(inputBuffer, input_len, key, encryptData);


    for (int i = 0; i < 32; i++)
    {
        if (ciphertext[i] != encryptData[i])
        {
            printf("Wrong\r\n");
            system("pause");
            return 0;
        }
    }

    printf("Right\r\n");

    system("pause");
    return 0;
}
```


## 解密脚本
### C语言实现
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

// S盒
uint8_t Sbox[256] = {
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,
    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,
    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,
    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,
    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,
    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,
    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,
    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,
    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,
    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,
    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,
    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,
    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,
    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,
    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,
    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16
};

// 逆S盒
uint8_t invSbox[256] =
{
    0
};


// 轮常数 (前10个)
uint32_t Rcon[10] = {
    0x01000000, 0x02000000,
    0x04000000, 0x08000000,
    0x10000000, 0x20000000,
    0x40000000, 0x80000000,
    0x1b000000, 0x36000000 };

// 列混合所用矩阵
uint8_t colM[4][4] =
{
    2, 3, 1, 1,
    1, 2, 3, 1,
    1, 1, 2, 3,
    3, 1, 1, 2
};

// 逆矩阵
uint8_t inv_colM[4][4] =
{
    0xe, 0xb, 0xd, 0x9,
    0x9, 0xe, 0xb, 0xd,
    0xd, 0x9, 0xe, 0xb,
    0xb, 0xd, 0x9, 0xe
};

// 将4字节串转化为4字节整数(大端序)
uint32_t BytesToIntB(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) |
        ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) |
        ((uint32_t)bytes[3]);
}


// 列混合所用辅助函数
//GF(2⁸) 乘法实现:
uint8_t GFMul2(uint8_t s)
{
    uint8_t result = s << 1;
    if (s & 0x80)
    {
        result ^= 0x1b;
    }
    return result;
}

uint8_t GFMul3(uint8_t s)
{
    return GFMul2(s) ^ s;
}
uint8_t GFMul4(uint8_t s)
{
    return GFMul2(GFMul2(s));
}
uint8_t GFMul8(uint8_t s)
{
    return GFMul2(GFMul4(s));
}
uint8_t GFMul9(uint8_t s)
{
    return GFMul8(s) ^ s;
}
uint8_t GFMul11(uint8_t s)
{
    return GFMul9(s) ^ GFMul2(s);
}
uint8_t GFMul12(uint8_t s)
{
    return GFMul8(s) ^ GFMul4(s);
}
uint8_t GFMul13(uint8_t s)
{
    return GFMul12(s) ^ s;
}
uint8_t GFMul14(uint8_t s)
{
    return GFMul12(s) ^ GFMul2(s);
}

uint8_t GFMul(int n, uint8_t s)
{
    switch (n)
    {
    case 1:
        return s;
    case 2:
        return GFMul2(s);
    case 3:
        return GFMul3(s);
    case 4:
        return GFMul4(s);
    case 8:
        return GFMul8(s);
    case 9:
        return GFMul9(s);
    case 11:
        return GFMul11(s);
    case 12:
        return GFMul12(s);
    case 13:
        return GFMul13(s);
    case 14:
        return GFMul14(s);
    default:
        printf("not found target GFMul number\r\n");
        system("pause");
        exit(EXIT_FAILURE);
    }
}



// 将4字节整数数据转化为字节串
void IntToBytesB(uint8_t* bytes, uint32_t intData)
{
    bytes[0] = (intData >> 24) & 0xff;
    bytes[1] = (intData >> 16) & 0xff;
    bytes[2] = (intData >> 8) & 0xff;
    bytes[3] = intData & 0xff;
}

// 128位密钥扩展中的T函数
uint32_t T(uint32_t num, uint32_t round)
{

    // 子循环: 将1个字中的4个字节循环左移1个字节。
    uint8_t t1 = (num >> 16) & 0xff;
    uint8_t t2 = (num >> 8) & 0xff;
    uint8_t t3 = num & 0xff;
    uint8_t t4 = (num >> 24) & 0xff;

    // 字节代换: 用S盒进行字节代换
    t1 = Sbox[t1];
    t2 = Sbox[t2];
    t3 = Sbox[t3];
    t4 = Sbox[t4];

    // 转化为4字节大端序整数
    uint32_t result =
        ((uint32_t)t1 << 24) |
        ((uint32_t)t2 << 16) |
        ((uint32_t)t3 << 8) |
        ((uint32_t)t4);

    // 轮常量异或: 与同轮常量Rcon[round / 4 - 1]异或
    result ^= Rcon[round / 4 - 1];

    return result;
}

// 128位密钥扩展
void key_expansion128(uint8_t* key, uint32_t* rk)
{
    // 将16字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 4; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展10轮密钥
    for (int i = 4; i < 44; i++)
    {
        // 如果i为4的整数倍需要进行T函数处理
        if (i % 4 == 0)
        {
            rk[i] = rk[i - 4] ^ T(rk[i - 1], i);
        }
        else
        {
            rk[i] = rk[i - 4] ^ rk[i - 1];
        }
    }

}

// 192位密钥扩展
void key_expansion192(uint8_t* key, uint32_t* rk)
{
    // 将24字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 6; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展10轮密钥
    for (int i = 6; i < 52; i++)
    {
        // i为6整数倍
        if (i % 6 == 0)
        {
            // 左移一位，字节代换，异或轮常数
            uint8_t t0 = Sbox[(rk[i - 1] >> 16) & 0xff];
            uint8_t t1 = Sbox[(rk[i - 1] >> 8) & 0xff];
            uint8_t t2 = Sbox[rk[i - 1] & 0xff];
            uint8_t t3 = Sbox[(rk[i - 1] >> 24) & 0xff];

            uint32_t temp =
                (uint32_t)t0 << 24 |
                (uint32_t)t1 << 16 |
                (uint32_t)t2 << 8 |
                (uint32_t)t3;

            temp ^= Rcon[i / 6 - 1];

            rk[i] = rk[i - 6] ^ temp;
        }
        else
        {
            rk[i] = rk[i - 1] ^ rk[i - 6];
        }
    }

}

// 生成逆S盒
void GenerateInvSbox()
{
    for (int i = 0; i < 256; i++)
    {
        invSbox[Sbox[i]] = i;
    }
}

// 逆字节代换
void invSubBytes(uint8_t inputBuffer[4][4])
{
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] = invSbox[inputBuffer[i][j]];
        }
    }
}

// 逆行移位
void invShiftRow(uint8_t inputBuffer[4][4])
{
    uint8_t temp[4] = { 0 };
    // 第一行不移位

    // 第二行右移1位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[1][i];
    }
    inputBuffer[1][0] = temp[3];
    inputBuffer[1][1] = temp[0];
    inputBuffer[1][2] = temp[1];
    inputBuffer[1][3] = temp[2];

    // 第三行右移2位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[2][i];
    }
    inputBuffer[2][0] = temp[2];
    inputBuffer[2][1] = temp[3];
    inputBuffer[2][2] = temp[0];
    inputBuffer[2][3] = temp[1];

    // 第四行右移3位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[3][i];
    }
    inputBuffer[3][0] = temp[1];
    inputBuffer[3][1] = temp[2];
    inputBuffer[3][2] = temp[3];
    inputBuffer[3][3] = temp[0];

}

// 逆列混合
void invMixColumns(uint8_t inputBuffer[4][4])
{
    uint8_t tempArray[4][4];
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            tempArray[i][j] = inputBuffer[i][j];
        }
    }

    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] =
                GFMul(inv_colM[i][0], tempArray[0][j]) ^
                GFMul(inv_colM[i][1], tempArray[1][j]) ^
                GFMul(inv_colM[i][2], tempArray[2][j]) ^
                GFMul(inv_colM[i][3], tempArray[3][j]);
        }
    }
}

// 加解密所用轮密钥加相同
void AddRoundKey(uint8_t intputBuffer[4][4], uint32_t* rk ,int round)
{
    uint8_t tempArray[4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        IntToBytesB(tempArray, rk[round * 4 + i]);
        for (int j = 0; j < 4; j++)
        {
            intputBuffer[j][i] ^= tempArray[j];
        }
    }
}

void aesDecrypt_128(uint8_t* ciphertext, uint32_t* rk, uint8_t* plaintext)
{
    // 将密文转化为密文矩阵
    uint8_t cipherArray[4][4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            cipherArray[j][i] = ciphertext[i * 4 + j];
        }
    }

    // 从第10轮加密开始逆解密
    AddRoundKey(cipherArray, rk, 10);
    invShiftRow(cipherArray);
    invSubBytes(cipherArray);

    // 从第9轮解密到第1轮
    for (int i = 9; i > 0; i--)
    {
        AddRoundKey(cipherArray, rk, i);
        invMixColumns(cipherArray);
        invShiftRow(cipherArray);
        invSubBytes(cipherArray);
    }

    // 初始轮密钥加
    AddRoundKey(cipherArray, rk, 0);

    // 将明文矩阵存储到明文数组中
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            plaintext[i * 4 + j] = cipherArray[j][i];
        }
    }
}

void aesDecrypt_192(uint8_t* ciphertext, uint32_t* rk, uint8_t* plaintext)
{
    // 将密文转化为密文矩阵
    uint8_t cipherArray[4][4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            cipherArray[j][i] = ciphertext[i * 4 + j];
        }
    }

    // 从第12轮加密开始逆解密
    AddRoundKey(cipherArray, rk, 12);
    invShiftRow(cipherArray);
    invSubBytes(cipherArray);

    // 从第11轮解密到第1轮
    for (int i = 11; i > 0; i--)
    {
        AddRoundKey(cipherArray, rk, i);
        invMixColumns(cipherArray);
        invShiftRow(cipherArray);
        invSubBytes(cipherArray);
    }

    // 初始轮密钥加
    AddRoundKey(cipherArray, rk, 0);

    // 将明文矩阵存储到明文数组中
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            plaintext[i * 4 + j] = cipherArray[j][i];
        }
    }
}


void aesDecryptECB_128(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext)
{
    // 检查密钥长度是否为16字节
    if (key_len != 16)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[44] = { 0 };
    key_expansion128(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_128(&ciphertext[i * 16], rk, &plaintext[i * 16]);
    }

}

void aesDecryptECB_192(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext)
{
    // 检查密钥长度是否为16字节
    if (key_len != 24)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[52] = { 0 };
    key_expansion192(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_192(&ciphertext[i * 16], rk, &plaintext[i * 16]);
    }

}

int main()
{
    uint8_t key[24] = { 0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38 };
    uint8_t ciphertext[32] = { 0x10, 0xAB, 0xC7, 0x3E, 0xD7, 0xA7, 0x16, 0x36, 0xC5, 0x71, 0xB2, 0xBC, 0x38, 0xF5, 0xD9, 0x95, 0x09, 0xB0, 0xC1, 0x56, 0xFC, 0x4D, 0xBB, 0x92, 0x50, 0xC3, 0x85, 0x8F, 0x65, 0x15, 0x9C, 0x45 };
    uint8_t plaintext[0xff] = { 0 };

    int key_len = sizeof(key) / sizeof(key[0]);
    int cipher_len = sizeof(ciphertext) / sizeof(ciphertext[0]);

    aesDecryptECB_192(ciphertext, key, cipher_len, key_len, plaintext);
    printf("%s\r\n", plaintext);

	return 0;
}

```
### python实现
```python
# S盒

Sbox = [

    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,

    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,

    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,

    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,

    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,

    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,

    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,

    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,

    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,

    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,

    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,

    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,

    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,

    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,

    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,

    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16

]

  

# 逆S盒

inv_Sbox = [0 for _ in range(256)]

  

# 轮常数

Rcon = [

    0x01000000, 0x02000000,

    0x04000000, 0x08000000,

    0x10000000, 0x20000000,

    0x40000000, 0x80000000,

    0x1b000000, 0x36000000

]

  

# 列混合所用矩阵

colM = [

    2, 3, 1, 1,

    1, 2, 3, 1,

    1, 1, 2, 3,

    3, 1, 1, 2

]

  

# 解密列混合所用逆矩阵

inv_colM = [

    [0xe, 0xb, 0xd, 0x9],

    [0x9, 0xe, 0xb, 0xd],

    [0xd, 0x9, 0xe, 0xb],

    [0xb, 0xd, 0x9, 0xe]

]

  

# 列混合所用辅助函数: GF(2⁸) 乘法实现

def GFMul2(s):

    s = s & 0xff

    result = s << 1

    if (s & 0x80):

        result ^= 0x1b

    return result & 0xff

  

def GFMul3(s):

    s = s & 0xff

    return GFMul2(s) ^ s

  

def GFMul4(s):

    s = s & 0xff

    return GFMul2(GFMul2(s))

  

def GFMul8(s):

    s = s & 0xff

    return GFMul4(GFMul2(s))

  

def GFMul9(s):

    s = s & 0xff

    return GFMul8(s) ^ s

  

def GFMul11(s):

    s = s & 0xff

    return GFMul9(s) ^ GFMul2(s)

  

def GFMul12(s):

    s = s & 0xff

    return GFMul8(s) ^ GFMul4(s)

  

def GFMul13(s):

    s = s & 0xff

    return GFMul12(s) ^ s

  

def GFMul14(s):

    s = s & 0xff

    return GFMul12(s) ^ GFMul2(s)

  

def GFMul(n, s):

    match n:

        case 1:

            return s & 0xff

        case 2:

            return GFMul2(s)

        case 3:

            return GFMul3(s)

        case 4:

            return GFMul4(s)

        case 8:

            return GFMul8(s)

        case 9:

            return GFMul9(s)

        case 11:

            return GFMul11(s)

        case 12:

            return GFMul12(s)

        case 13:

            return GFMul13(s)

        case 14:

            return GFMul14(s)

        case _:

            print("Not found target GFMul value")

            return 0

  

def key_expansion128(key, rk):

    # 将16字节key以4字节大端序初始化存储在rk中

    for i in range(4):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展10轮密钥

    for i in range(4, 44, 1):

        # i为4的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 4 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 4 - 1]

            rk[i] = rk[i - 4] ^ temp

        else :

            rk[i] = rk[i - 4] ^ rk[i - 1]

  

def key_expansion192(key, rk):

    # 将24字节key以4字节大端序初始化存储在rk中

    for i in range(6):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展12轮密钥

    for i in range(6, 52, 1):

        # i为6的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 6 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 6 - 1]

            rk[i] = rk[i - 6] ^ temp

        else :

            rk[i] = rk[i - 6] ^ rk[i - 1]

  

def key_expansion256(key, rk):

    # 将32字节key以4字节大端序初始化存储在rk中

    for i in range(8):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展14轮密钥

    for i in range(8, 60, 1):

        # i为6的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 8 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 8 - 1]

            rk[i] = rk[i - 8] ^ temp

  

        # 256加密特有:i % 8 == 4时只执行字节代换

        elif (i % 8 == 4):

            t0 = Sbox[(rk[i - 1] >> 24) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t2 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t3 = Sbox[(rk[i - 1]) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

  

            rk[i] = rk[i - 8] ^ temp

  

        else :

            rk[i] = rk[i - 8] ^ rk[i - 1]

  

# 生成逆S盒

def generate_inv_Sbox():

    for i in range(256):

        inv_Sbox[Sbox[i]] = i

  

# 逆字节代换

def inv_SubBytes(state):

    for i in range(16):

        state[i] = inv_Sbox[state[i]]

# 逆行移位

def inv_ShiftRow(state):

    # 第一行不移位

  

    # 第二行右移1位

    state[1], state[5], state[9], state[13] = state[13], state[1], state[5], state[9]

  

    # 第三行右移2位

    state[2], state[6], state[10], state[14] = state[10], state[14], state[2], state[6]

  

    # 第四行右移3位

    state[3], state[7], state[11], state[15] = state[7], state[11], state[15], state[3]

  

# 逆列混合

def inv_MixColumns(state):

    temp_array = []

    for i in range(16):

        temp_array.append(state[i])

  

    # 逆矩阵乘密文矩阵

    for i in range(4):

        for j in range(4):

            state[i + j * 4] = (

                GFMul(inv_colM[i][0], temp_array[j * 4 + 0]) ^

                GFMul(inv_colM[i][1], temp_array[j * 4 + 1]) ^

                GFMul(inv_colM[i][2], temp_array[j * 4 + 2]) ^

                GFMul(inv_colM[i][3], temp_array[j * 4 + 3])

            )

  

# 逆轮密钥加

def inv_AddRoundKey(state, round, rk):

    t0 = (int.from_bytes(state[0:4], "big") ^ rk[round * 4]) & 0xffffffff

    t1 = (int.from_bytes(state[4:8], "big") ^ rk[round * 4 + 1]) & 0xffffffff

    t2 = (int.from_bytes(state[8:12], "big") ^ rk[round * 4 + 2]) & 0xffffffff

    t3 = (int.from_bytes(state[12:16], "big") ^ rk[round * 4 + 3]) & 0xffffffff

  

    temp_bytes = t0.to_bytes(4,"big")

    temp_bytes += t1.to_bytes(4,"big")

    temp_bytes += t2.to_bytes(4,"big")

    temp_bytes += t3.to_bytes(4,"big")

  

    for i in range(16):

        state[i] = temp_bytes[i]

  

def aes128_decrypt(state, rk):

  

    # 从第10轮开始解密

    inv_AddRoundKey(state, 10, rk)

    inv_ShiftRow(state)

    inv_SubBytes(state)

  

    # 从第9轮解密到第1轮

    for i in range(9, 0, -1):

        inv_AddRoundKey(state, i, rk)

        inv_MixColumns(state)

        inv_ShiftRow(state)

        inv_SubBytes(state)

    # 解密初始轮密钥加

    inv_AddRoundKey(state, 0, rk)

  
  

def aes128ECB_decrypt(ciphertext, key):

    # 检查密钥长度是否为16字节

    if len(key) != 16:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(44)]

    key_expansion128(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes128_decrypt(state, rk)

        plaintext += state

    print(plaintext)

  
  

def aes192_decrypt(state, rk):

  

    # 从第12轮开始解密

    inv_AddRoundKey(state, 12, rk)

    inv_ShiftRow(state)

    inv_SubBytes(state)

  

    # 从第11轮解密到第1轮

    for i in range(11, 0, -1):

        inv_AddRoundKey(state, i, rk)

        inv_MixColumns(state)

        inv_ShiftRow(state)

        inv_SubBytes(state)

    # 解密初始轮密钥加

    inv_AddRoundKey(state, 0, rk)

  
  

def aes192ECB_decrypt(ciphertext, key):

    # 检查密钥长度是否为24字节

    if len(key) != 24:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(52)]

    key_expansion192(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes192_decrypt(state, rk)

        plaintext += state

    print(plaintext)

  

def main():

    key = bytes([0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A])

    ciphertext = bytes([0x25,  0x1C,  0xCC,  0xA0,  0xD8,  0x56,  0x6F,  0x43,  0xC1,  0x5A,  0x1F,  0xD7,  0x5A,  0x21,  0x1A,  0x66,  0x6D,  0xE5,  0x50,  0xBE,  0xB6,  0xE8,  0x17,  0x57,  0x01,  0xCD,  0x4A,  0x07,  0x7B,  0x21,  0xA0,  0x20])

    aes128ECB_decrypt(ciphertext, key)

  

    key = bytes([ 0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38])

    ciphertext = bytes([0x10, 0xAB, 0xC7, 0x3E, 0xD7, 0xA7, 0x16, 0x36, 0xC5, 0x71, 0xB2, 0xBC, 0x38, 0xF5, 0xD9, 0x95, 0x09, 0xB0, 0xC1, 0x56, 0xFC, 0x4D, 0xBB, 0x92, 0x50, 0xC3, 0x85, 0x8F, 0x65, 0x15, 0x9C, 0x45 ])

    aes192ECB_decrypt(ciphertext, key)

  
  

if __name__ == "__main__":

    main()
```

# aes256ECB模式

## 加密脚本
```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h> 

// Subbytes所用S盒
uint8_t Sbox[256] = {
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,
    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,
    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,
    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,
    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,
    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,
    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,
    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,
    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,
    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,
    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,
    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,
    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,
    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,
    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,
    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16
};

// 轮常数 (前10个)
uint32_t Rcon[10] = {
    0x01000000, 0x02000000,
    0x04000000, 0x08000000,
    0x10000000, 0x20000000,
    0x40000000, 0x80000000,
    0x1b000000, 0x36000000 };

// 列混合所用矩阵
uint8_t colM[4][4] =
{
    2, 3, 1, 1,
    1, 2, 3, 1,
    1, 1, 2, 3,
    3, 1, 1, 2
};

// 将4字节串转化为4字节整数(大端序)
uint32_t BytesToIntB(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) |
        ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) |
        ((uint32_t)bytes[3]);
}

// 列混合所用辅助函数
//GF(2⁸) 乘法实现:
uint8_t GFMul2(uint8_t s)
{
    uint8_t result = s << 1;
    if (s & 0x80)
    {
        result ^= 0x1b;
    }
    return result;
}

uint8_t GFMul3(uint8_t s)
{
    return GFMul2(s) ^ s;
}
uint8_t GFMul4(uint8_t s)
{
    return GFMul2(GFMul2(s));
}
uint8_t GFMul8(uint8_t s)
{
    return GFMul2(GFMul4(s));
}
uint8_t GFMul9(uint8_t s)
{
    return GFMul8(s) ^ s;
}
uint8_t GFMul11(uint8_t s)
{
    return GFMul9(s) ^ GFMul2(s);
}
uint8_t GFMul12(uint8_t s)
{
    return GFMul8(s) ^ GFMul4(s);
}
uint8_t GFMul13(uint8_t s)
{
    return GFMul12(s) ^ s;
}
uint8_t GFMul14(uint8_t s)
{
    return GFMul12(s) ^ GFMul2(s);
}

uint8_t GFMul(int n, uint8_t s)
{
    switch (n)
    {
    case 1:
        return s;
    case 2:
        return GFMul2(s);
    case 3:
        return GFMul3(s);
    case 4:
        return GFMul4(s);
    case 8:
        return GFMul8(s);
    case 9:
        return GFMul9(s);
    case 11:
        return GFMul11(s);
    case 12:
        return GFMul12(s);
    case 13:
        return GFMul13(s);
    case 14:
        return GFMul14(s);
    default:
        printf("not found target GFMul number\r\n");
        system("pause");
        exit(EXIT_FAILURE);
    }
}

// 将4字节整数数据转化为字节串
void IntToBytesB(uint8_t* bytes, uint32_t intData)
{
    bytes[0] = (intData >> 24) & 0xff;
    bytes[1] = (intData >> 16) & 0xff;
    bytes[2] = (intData >> 8) & 0xff;
    bytes[3] = intData & 0xff;
}

// 密钥扩展
void key_expansion(uint8_t* key, uint32_t* rk)
{
    // 将32字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 8; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展14轮密钥
    for (int i = 8; i < 60; i++)
    {
        // i为8整数倍
        if (i % 8 == 0)
        {
            // 左移一位，字节代换，异或轮常数
            uint8_t t0 = Sbox[(rk[i - 1] >> 16) & 0xff];
            uint8_t t1 = Sbox[(rk[i - 1] >> 8) & 0xff];
            uint8_t t2 = Sbox[rk[i - 1] & 0xff];
            uint8_t t3 = Sbox[(rk[i - 1] >> 24) & 0xff];

            uint32_t temp =
                (uint32_t)t0 << 24 |
                (uint32_t)t1 << 16 |
                (uint32_t)t2 << 8 |
                (uint32_t)t3;

            temp ^= Rcon[i / 8 - 1];

            rk[i] = rk[i - 8] ^ temp;
        }
        else if (i % 8 == 4)
        {
            // aes256加密特有: 仅进行字节代换
            uint8_t t0 = Sbox[(rk[i - 1] >> 24) & 0xff];
            uint8_t t1 = Sbox[(rk[i - 1] >> 16) & 0xff];
            uint8_t t2 = Sbox[(rk[i - 1] >> 8) & 0xff];
            uint8_t t3 = Sbox[rk[i - 1] & 0xff];

            uint32_t temp =
                (uint32_t)t0 << 24 |
                (uint32_t)t1 << 16 |
                (uint32_t)t2 << 8 |
                (uint32_t)t3;

            rk[i] = rk[i - 8] ^ temp;
        }
        else
        {
            rk[i] = rk[i - 1] ^ rk[i - 8];
        }
    }

}

// 字节代换
void SubBytes(uint8_t inputBuffer[4][4])
{
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] = Sbox[inputBuffer[i][j]];
        }
    }
}

// 行移位
void ShiftRows(uint8_t inputbuffer[4][4])
{
    uint8_t temp[4] = { 0 };
    // 第一行不移位

    // 第二行左移1位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputbuffer[1][i];
    }
    inputbuffer[1][0] = temp[1];
    inputbuffer[1][1] = temp[2];
    inputbuffer[1][2] = temp[3];
    inputbuffer[1][3] = temp[0];

    // 第三行左移2位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputbuffer[2][i];
    }
    inputbuffer[2][0] = temp[2];
    inputbuffer[2][1] = temp[3];
    inputbuffer[2][2] = temp[0];
    inputbuffer[2][3] = temp[1];

    // 第四行行左移3位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputbuffer[3][i];
    }
    inputbuffer[3][0] = temp[3];
    inputbuffer[3][1] = temp[0];
    inputbuffer[3][2] = temp[1];
    inputbuffer[3][3] = temp[2];

}

// 列混合
void MixColumns(uint8_t inputBuffer[4][4])
{
    uint8_t tempArray[4][4];
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            tempArray[i][j] = inputBuffer[i][j];
        }
    }

    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] =
                GFMul(colM[i][0], tempArray[0][j]) ^
                GFMul(colM[i][1], tempArray[1][j]) ^
                GFMul(colM[i][2], tempArray[2][j]) ^
                GFMul(colM[i][3], tempArray[3][j]);
        }
    }

}

// 轮密钥加
void AddRoundKey(uint8_t inputBuffer[4][4], uint32_t* rk, int round)
{
    // 存储密钥
    uint8_t tempArray[4];

    for (int i = 0; i < 4; i++)
    {
        // 将4字节整数转化为字节串
        IntToBytesB(tempArray, rk[round * 4 + i]);

        for (int j = 0; j < 4; j++)
        {
            inputBuffer[j][i] ^= tempArray[j];
        }
    }
}

void aes256Encrypt(uint8_t* inputBuffer, uint32_t* rk, uint8_t* encryptData)
{
    // 将输入转化为明文矩阵
    uint8_t inputArray[4][4];
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputArray[j][i] = inputBuffer[i * 4 + j];
        }
    }

    // 初始轮密钥加
    AddRoundKey(inputArray, rk, 0);

    // 13轮加密循环
    for (int i = 1; i < 14; i++)
    {
        SubBytes(inputArray);
        ShiftRows(inputArray);
        MixColumns(inputArray);
        AddRoundKey(inputArray, rk, i);
    }

    // 最后一轮加密，无MixColumns
    SubBytes(inputArray);
    ShiftRows(inputArray);
    AddRoundKey(inputArray, rk, 14);

    // 将密文矩阵转化为字节串
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            encryptData[i * 4 + j] = inputArray[j][i];
        }
    }

}

void aesEncrypt(uint8_t* inputBuffer, int input_len, uint8_t* key, uint8_t* encryptData)
{
    // aes256模式下轮密钥为(14 + 1) * 4个字
    uint32_t rk[60] = { 0 };
    key_expansion(key, rk);


    int blockSize = input_len / 16;
    for (int i = 0; i < blockSize; i++)
    {
        aes256Encrypt(&inputBuffer[i * 16], rk, &encryptData[i * 16]);
    }

}

int main()
{
    // 固定32字节密钥
    uint8_t key[32] = { 0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38,0x39,0x40,0x41,0x42,0x43,0x44,0x45,0x46 };
    uint8_t encryptData[0xff] = { 0 }; // 存放加密后的数据
    uint8_t ciphertext[] = { 0xB6, 0x99, 0x22, 0x31, 0x3B, 0xC7, 0x83, 0x86, 0xD8, 0xB7, 0x04, 0x30, 0x3F, 0x0E, 0x2A, 0x1B, 0x63, 0x0E, 0x3F, 0xF2, 0xEA, 0x9A, 0xCF, 0xD3, 0x88, 0xC0, 0x39, 0xB5, 0x72, 0xBC, 0x58, 0x14 };// 密文

    uint8_t inputBuffer[0xff] = { 0 };
    printf("Please input your flag: ");
    scanf("%s", inputBuffer);

    int input_len = strlen((char*)inputBuffer);
    if (input_len != 32)
    {
        printf("wrong length\r\n");
        system("pause");
        return 0;
    }

    aesEncrypt(inputBuffer, input_len, key, encryptData);


    for (int i = 0; i < 32; i++)
    {
        if (ciphertext[i] != encryptData[i])
        {
            printf("Wrong\r\n");
            system("pause");
            return 0;
        }
    }

    printf("Right\r\n");

    system("pause");
    return 0;
}
```

## 解密脚本
### C语言实现
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

// S盒
uint8_t Sbox[256] = {
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,
    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,
    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,
    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,
    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,
    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,
    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,
    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,
    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,
    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,
    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,
    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,
    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,
    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,
    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,
    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16
};

// 逆S盒 - 在GenerateInvSbox函数中生成
uint8_t invSbox[256] =
{
    0
};


// 轮常数 (前10个)
uint32_t Rcon[10] = {
    0x01000000, 0x02000000,
    0x04000000, 0x08000000,
    0x10000000, 0x20000000,
    0x40000000, 0x80000000,
    0x1b000000, 0x36000000 };

// 列混合所用矩阵
uint8_t colM[4][4] =
{
    2, 3, 1, 1,
    1, 2, 3, 1,
    1, 1, 2, 3,
    3, 1, 1, 2
};

// 逆矩阵
uint8_t inv_colM[4][4] =
{
    0xe, 0xb, 0xd, 0x9,
    0x9, 0xe, 0xb, 0xd,
    0xd, 0x9, 0xe, 0xb,
    0xb, 0xd, 0x9, 0xe
};

// 将4字节串转化为4字节整数(大端序)
uint32_t BytesToIntB(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) |
        ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) |
        ((uint32_t)bytes[3]);
}


// 列混合所用辅助函数
//GF(2⁸) 乘法实现:
uint8_t GFMul2(uint8_t s)
{
    uint8_t result = s << 1;
    if (s & 0x80)
    {
        result ^= 0x1b;
    }
    return result;
}

uint8_t GFMul3(uint8_t s)
{
    return GFMul2(s) ^ s;
}
uint8_t GFMul4(uint8_t s)
{
    return GFMul2(GFMul2(s));
}
uint8_t GFMul8(uint8_t s)
{
    return GFMul2(GFMul4(s));
}
uint8_t GFMul9(uint8_t s)
{
    return GFMul8(s) ^ s;
}
uint8_t GFMul11(uint8_t s)
{
    return GFMul9(s) ^ GFMul2(s);
}
uint8_t GFMul12(uint8_t s)
{
    return GFMul8(s) ^ GFMul4(s);
}
uint8_t GFMul13(uint8_t s)
{
    return GFMul12(s) ^ s;
}
uint8_t GFMul14(uint8_t s)
{
    return GFMul12(s) ^ GFMul2(s);
}

uint8_t GFMul(int n, uint8_t s)
{
    switch (n)
    {
    case 1:
        return s;
    case 2:
        return GFMul2(s);
    case 3:
        return GFMul3(s);
    case 4:
        return GFMul4(s);
    case 8:
        return GFMul8(s);
    case 9:
        return GFMul9(s);
    case 11:
        return GFMul11(s);
    case 12:
        return GFMul12(s);
    case 13:
        return GFMul13(s);
    case 14:
        return GFMul14(s);
    default:
        printf("not found target GFMul number\r\n");
        system("pause");
        exit(EXIT_FAILURE);
    }
}



// 将4字节整数数据转化为字节串
void IntToBytesB(uint8_t* bytes, uint32_t intData)
{
    bytes[0] = (intData >> 24) & 0xff;
    bytes[1] = (intData >> 16) & 0xff;
    bytes[2] = (intData >> 8) & 0xff;
    bytes[3] = intData & 0xff;
}

// 128位密钥扩展中的T函数
uint32_t T(uint32_t num, uint32_t round)
{

    // 子循环: 将1个字中的4个字节循环左移1个字节。
    uint8_t t1 = (num >> 16) & 0xff;
    uint8_t t2 = (num >> 8) & 0xff;
    uint8_t t3 = num & 0xff;
    uint8_t t4 = (num >> 24) & 0xff;

    // 字节代换: 用S盒进行字节代换
    t1 = Sbox[t1];
    t2 = Sbox[t2];
    t3 = Sbox[t3];
    t4 = Sbox[t4];

    // 转化为4字节大端序整数
    uint32_t result =
        ((uint32_t)t1 << 24) |
        ((uint32_t)t2 << 16) |
        ((uint32_t)t3 << 8) |
        ((uint32_t)t4);

    // 轮常量异或: 与同轮常量Rcon[round / 4 - 1]异或
    result ^= Rcon[round / 4 - 1];

    return result;
}

// 128位密钥扩展
void key_expansion128(uint8_t* key, uint32_t* rk)
{
    // 将16字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 4; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展10轮密钥
    for (int i = 4; i < 44; i++)
    {
        // 如果i为4的整数倍需要进行T函数处理
        if (i % 4 == 0)
        {
            rk[i] = rk[i - 4] ^ T(rk[i - 1], i);
        }
        else
        {
            rk[i] = rk[i - 4] ^ rk[i - 1];
        }
    }

}

// 192位密钥扩展
void key_expansion192(uint8_t* key, uint32_t* rk)
{
    // 将24字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 6; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展10轮密钥
    for (int i = 6; i < 52; i++)
    {
        // i为6整数倍
        if (i % 6 == 0)
        {
            // 左移一位，字节代换，异或轮常数
            uint8_t t0 = Sbox[(rk[i - 1] >> 16) & 0xff];
            uint8_t t1 = Sbox[(rk[i - 1] >> 8) & 0xff];
            uint8_t t2 = Sbox[rk[i - 1] & 0xff];
            uint8_t t3 = Sbox[(rk[i - 1] >> 24) & 0xff];

            uint32_t temp =
                (uint32_t)t0 << 24 |
                (uint32_t)t1 << 16 |
                (uint32_t)t2 << 8 |
                (uint32_t)t3;

            temp ^= Rcon[i / 6 - 1];

            rk[i] = rk[i - 6] ^ temp;
        }
        else
        {
            rk[i] = rk[i - 1] ^ rk[i - 6];
        }
    }

}

// 256位密钥扩展
void key_expansion256(uint8_t* key, uint32_t* rk)
{
    // 将32字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 8; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展14轮密钥
    for (int i = 8; i < 60; i++)
    {
        // i为8整数倍
        if (i % 8 == 0)
        {
            // 左移一位，字节代换，异或轮常数
            uint8_t t0 = Sbox[(rk[i - 1] >> 16) & 0xff];
            uint8_t t1 = Sbox[(rk[i - 1] >> 8) & 0xff];
            uint8_t t2 = Sbox[rk[i - 1] & 0xff];
            uint8_t t3 = Sbox[(rk[i - 1] >> 24) & 0xff];

            uint32_t temp =
                (uint32_t)t0 << 24 |
                (uint32_t)t1 << 16 |
                (uint32_t)t2 << 8 |
                (uint32_t)t3;

            temp ^= Rcon[i / 8 - 1];

            rk[i] = rk[i - 8] ^ temp;
        }
        else if (i % 8 == 4)
        {
            // aes256加密特有: 仅进行字节代换
            uint8_t t0 = Sbox[(rk[i - 1] >> 24) & 0xff];
            uint8_t t1 = Sbox[(rk[i - 1] >> 16) & 0xff];
            uint8_t t2 = Sbox[(rk[i - 1] >> 8) & 0xff];
            uint8_t t3 = Sbox[rk[i - 1] & 0xff];

            uint32_t temp =
                (uint32_t)t0 << 24 |
                (uint32_t)t1 << 16 |
                (uint32_t)t2 << 8 |
                (uint32_t)t3;

            rk[i] = rk[i - 8] ^ temp;
        }
        else
        {
            rk[i] = rk[i - 1] ^ rk[i - 8];
        }
    }

}

// 生成逆S盒
void GenerateInvSbox()
{
    for (int i = 0; i < 256; i++)
    {
        invSbox[Sbox[i]] = i;
    }
}

// 逆字节代换
void invSubBytes(uint8_t inputBuffer[4][4])
{
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] = invSbox[inputBuffer[i][j]];
        }
    }
}

// 逆行移位
void invShiftRow(uint8_t inputBuffer[4][4])
{
    uint8_t temp[4] = { 0 };
    // 第一行不移位

    // 第二行右移1位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[1][i];
    }
    inputBuffer[1][0] = temp[3];
    inputBuffer[1][1] = temp[0];
    inputBuffer[1][2] = temp[1];
    inputBuffer[1][3] = temp[2];

    // 第三行右移2位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[2][i];
    }
    inputBuffer[2][0] = temp[2];
    inputBuffer[2][1] = temp[3];
    inputBuffer[2][2] = temp[0];
    inputBuffer[2][3] = temp[1];

    // 第四行右移3位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[3][i];
    }
    inputBuffer[3][0] = temp[1];
    inputBuffer[3][1] = temp[2];
    inputBuffer[3][2] = temp[3];
    inputBuffer[3][3] = temp[0];

}

// 逆列混合
void invMixColumns(uint8_t inputBuffer[4][4])
{
    uint8_t tempArray[4][4];
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            tempArray[i][j] = inputBuffer[i][j];
        }
    }

    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] =
                GFMul(inv_colM[i][0], tempArray[0][j]) ^
                GFMul(inv_colM[i][1], tempArray[1][j]) ^
                GFMul(inv_colM[i][2], tempArray[2][j]) ^
                GFMul(inv_colM[i][3], tempArray[3][j]);
        }
    }
}

// 加解密所用轮密钥加相同
void AddRoundKey(uint8_t intputBuffer[4][4], uint32_t* rk ,int round)
{
    uint8_t tempArray[4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        IntToBytesB(tempArray, rk[round * 4 + i]);
        for (int j = 0; j < 4; j++)
        {
            intputBuffer[j][i] ^= tempArray[j];
        }
    }
}

void aesDecrypt_128(uint8_t* ciphertext, uint32_t* rk, uint8_t* plaintext)
{
    // 将密文转化为密文矩阵
    uint8_t cipherArray[4][4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            cipherArray[j][i] = ciphertext[i * 4 + j];
        }
    }

    // 从第10轮加密开始逆解密
    AddRoundKey(cipherArray, rk, 10);
    invShiftRow(cipherArray);
    invSubBytes(cipherArray);

    // 从第9轮解密到第1轮
    for (int i = 9; i > 0; i--)
    {
        AddRoundKey(cipherArray, rk, i);
        invMixColumns(cipherArray);
        invShiftRow(cipherArray);
        invSubBytes(cipherArray);
    }

    // 初始轮密钥加
    AddRoundKey(cipherArray, rk, 0);

    // 将明文矩阵存储到明文数组中
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            plaintext[i * 4 + j] = cipherArray[j][i];
        }
    }
}

void aesDecrypt_192(uint8_t* ciphertext, uint32_t* rk, uint8_t* plaintext)
{
    // 将密文转化为密文矩阵
    uint8_t cipherArray[4][4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            cipherArray[j][i] = ciphertext[i * 4 + j];
        }
    }

    // 从第10轮加密开始逆解密
    AddRoundKey(cipherArray, rk, 12);
    invShiftRow(cipherArray);
    invSubBytes(cipherArray);

    // 从第9轮解密到第1轮
    for (int i = 11; i > 0; i--)
    {
        AddRoundKey(cipherArray, rk, i);
        invMixColumns(cipherArray);
        invShiftRow(cipherArray);
        invSubBytes(cipherArray);
    }

    // 初始轮密钥加
    AddRoundKey(cipherArray, rk, 0);

    // 将明文矩阵存储到明文数组中
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            plaintext[i * 4 + j] = cipherArray[j][i];
        }
    }
}

void aesDecrypt_256(uint8_t* ciphertext, uint32_t* rk, uint8_t* plaintext)
{
    // 将密文转化为密文矩阵
    uint8_t cipherArray[4][4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            cipherArray[j][i] = ciphertext[i * 4 + j];
        }
    }

    // 从第14轮加密开始逆解密
    AddRoundKey(cipherArray, rk, 14);
    invShiftRow(cipherArray);
    invSubBytes(cipherArray);

    // 从第13轮解密到第1轮
    for (int i = 13; i > 0; i--)
    {
        AddRoundKey(cipherArray, rk, i);
        invMixColumns(cipherArray);
        invShiftRow(cipherArray);
        invSubBytes(cipherArray);
    }

    // 初始轮密钥加
    AddRoundKey(cipherArray, rk, 0);

    // 将明文矩阵存储到明文数组中
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            plaintext[i * 4 + j] = cipherArray[j][i];
        }
    }
}

void aesDecryptECB_128(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext)
{
    // 检查密钥长度是否为16字节
    if (key_len != 16)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[44] = { 0 };
    key_expansion128(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_128(&ciphertext[i * 16], rk, &plaintext[i * 16]);
    }

}

void aesDecryptECB_192(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext)
{
    // 检查密钥长度是否为16字节
    if (key_len != 24)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[52] = { 0 };
    key_expansion192(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_192(&ciphertext[i * 16], rk, &plaintext[i * 16]);
    }

}

void aesDecryptECB_256(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext)
{
    // 检查密钥长度是否为32字节
    if (key_len != 32)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[60] = { 0 };
    key_expansion256(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_256(&ciphertext[i * 16], rk, &plaintext[i * 16]);
    }
}

int main()
{
    uint8_t key[32] = { 0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38,0x39,0x40,0x41,0x42,0x43,0x44,0x45,0x46 };
    uint8_t ciphertext[32] = { 0xB6, 0x99, 0x22, 0x31, 0x3B, 0xC7, 0x83, 0x86, 0xD8, 0xB7, 0x04, 0x30, 0x3F, 0x0E, 0x2A, 0x1B, 0x63, 0x0E, 0x3F, 0xF2, 0xEA, 0x9A, 0xCF, 0xD3, 0x88, 0xC0, 0x39, 0xB5, 0x72, 0xBC, 0x58, 0x14 };
    uint8_t plaintext[0xff] = { 0 };

    int key_len = sizeof(key) / sizeof(key[0]);
    int cipher_len = sizeof(ciphertext) / sizeof(ciphertext[0]);

    aesDecryptECB_256(ciphertext, key, cipher_len, key_len, plaintext);
    printf("%s\r\n", plaintext);

	return 0;
}

```

### python实现
```python
# S盒

Sbox = [

    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,

    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,

    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,

    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,

    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,

    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,

    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,

    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,

    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,

    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,

    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,

    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,

    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,

    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,

    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,

    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16

]

  

# 逆S盒

inv_Sbox = [0 for _ in range(256)]

  

# 轮常数

Rcon = [

    0x01000000, 0x02000000,

    0x04000000, 0x08000000,

    0x10000000, 0x20000000,

    0x40000000, 0x80000000,

    0x1b000000, 0x36000000

]

  

# 列混合所用矩阵

colM = [

    2, 3, 1, 1,

    1, 2, 3, 1,

    1, 1, 2, 3,

    3, 1, 1, 2

]

  

# 解密列混合所用逆矩阵

inv_colM = [

    [0xe, 0xb, 0xd, 0x9],

    [0x9, 0xe, 0xb, 0xd],

    [0xd, 0x9, 0xe, 0xb],

    [0xb, 0xd, 0x9, 0xe]

]

  

# 列混合所用辅助函数: GF(2⁸) 乘法实现

def GFMul2(s):

    s = s & 0xff

    result = s << 1

    if (s & 0x80):

        result ^= 0x1b

    return result & 0xff

  

def GFMul3(s):

    s = s & 0xff

    return GFMul2(s) ^ s

  

def GFMul4(s):

    s = s & 0xff

    return GFMul2(GFMul2(s))

  

def GFMul8(s):

    s = s & 0xff

    return GFMul4(GFMul2(s))

  

def GFMul9(s):

    s = s & 0xff

    return GFMul8(s) ^ s

  

def GFMul11(s):

    s = s & 0xff

    return GFMul9(s) ^ GFMul2(s)

  

def GFMul12(s):

    s = s & 0xff

    return GFMul8(s) ^ GFMul4(s)

  

def GFMul13(s):

    s = s & 0xff

    return GFMul12(s) ^ s

  

def GFMul14(s):

    s = s & 0xff

    return GFMul12(s) ^ GFMul2(s)

  

def GFMul(n, s):

    match n:

        case 1:

            return s & 0xff

        case 2:

            return GFMul2(s)

        case 3:

            return GFMul3(s)

        case 4:

            return GFMul4(s)

        case 8:

            return GFMul8(s)

        case 9:

            return GFMul9(s)

        case 11:

            return GFMul11(s)

        case 12:

            return GFMul12(s)

        case 13:

            return GFMul13(s)

        case 14:

            return GFMul14(s)

        case _:

            print("Not found target GFMul value")

            return 0

  

def key_expansion128(key, rk):

    # 将16字节key以4字节大端序初始化存储在rk中

    for i in range(4):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展10轮密钥

    for i in range(4, 44, 1):

        # i为4的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 4 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 4 - 1]

            rk[i] = rk[i - 4] ^ temp

        else :

            rk[i] = rk[i - 4] ^ rk[i - 1]

  

def key_expansion192(key, rk):

    # 将24字节key以4字节大端序初始化存储在rk中

    for i in range(6):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展12轮密钥

    for i in range(6, 52, 1):

        # i为6的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 6 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 6 - 1]

            rk[i] = rk[i - 6] ^ temp

        else :

            rk[i] = rk[i - 6] ^ rk[i - 1]

  

def key_expansion256(key, rk):

    # 将32字节key以4字节大端序初始化存储在rk中

    for i in range(8):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展14轮密钥

    for i in range(8, 60, 1):

        # i为6的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 8 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 8 - 1]

            rk[i] = rk[i - 8] ^ temp

  

        # 256加密特有:i % 8 == 4时只执行字节代换

        elif (i % 8 == 4):

            t0 = Sbox[(rk[i - 1] >> 24) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t2 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t3 = Sbox[(rk[i - 1]) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

  

            rk[i] = rk[i - 8] ^ temp

  

        else :

            rk[i] = rk[i - 8] ^ rk[i - 1]

  

# 生成逆S盒

def generate_inv_Sbox():

    for i in range(256):

        inv_Sbox[Sbox[i]] = i

  

# 逆字节代换

def inv_SubBytes(state):

    for i in range(16):

        state[i] = inv_Sbox[state[i]]

# 逆行移位

def inv_ShiftRow(state):

    # 第一行不移位

  

    # 第二行右移1位

    state[1], state[5], state[9], state[13] = state[13], state[1], state[5], state[9]

  

    # 第三行右移2位

    state[2], state[6], state[10], state[14] = state[10], state[14], state[2], state[6]

  

    # 第四行右移3位

    state[3], state[7], state[11], state[15] = state[7], state[11], state[15], state[3]

  

# 逆列混合

def inv_MixColumns(state):

    temp_array = []

    for i in range(16):

        temp_array.append(state[i])

  

    # 逆矩阵乘密文矩阵

    for i in range(4):

        for j in range(4):

            state[i + j * 4] = (

                GFMul(inv_colM[i][0], temp_array[j * 4 + 0]) ^

                GFMul(inv_colM[i][1], temp_array[j * 4 + 1]) ^

                GFMul(inv_colM[i][2], temp_array[j * 4 + 2]) ^

                GFMul(inv_colM[i][3], temp_array[j * 4 + 3])

            )

  

# 逆轮密钥加

def inv_AddRoundKey(state, round, rk):

    t0 = (int.from_bytes(state[0:4], "big") ^ rk[round * 4]) & 0xffffffff

    t1 = (int.from_bytes(state[4:8], "big") ^ rk[round * 4 + 1]) & 0xffffffff

    t2 = (int.from_bytes(state[8:12], "big") ^ rk[round * 4 + 2]) & 0xffffffff

    t3 = (int.from_bytes(state[12:16], "big") ^ rk[round * 4 + 3]) & 0xffffffff

  

    temp_bytes = t0.to_bytes(4,"big")

    temp_bytes += t1.to_bytes(4,"big")

    temp_bytes += t2.to_bytes(4,"big")

    temp_bytes += t3.to_bytes(4,"big")

  

    for i in range(16):

        state[i] = temp_bytes[i]

  

def aes128_decrypt(state, rk):

  

    # 从第10轮开始解密

    inv_AddRoundKey(state, 10, rk)

    inv_ShiftRow(state)

    inv_SubBytes(state)

  

    # 从第9轮解密到第1轮

    for i in range(9, 0, -1):

        inv_AddRoundKey(state, i, rk)

        inv_MixColumns(state)

        inv_ShiftRow(state)

        inv_SubBytes(state)

    # 解密初始轮密钥加

    inv_AddRoundKey(state, 0, rk)

  
  

def aes128ECB_decrypt(ciphertext, key):

    # 检查密钥长度是否为16字节

    if len(key) != 16:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(44)]

    key_expansion128(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes128_decrypt(state, rk)

        plaintext += state

    print(plaintext)

  
  

def aes192_decrypt(state, rk):

  

    # 从第12轮开始解密

    inv_AddRoundKey(state, 12, rk)

    inv_ShiftRow(state)

    inv_SubBytes(state)

  

    # 从第11轮解密到第1轮

    for i in range(11, 0, -1):

        inv_AddRoundKey(state, i, rk)

        inv_MixColumns(state)

        inv_ShiftRow(state)

        inv_SubBytes(state)

    # 解密初始轮密钥加

    inv_AddRoundKey(state, 0, rk)

  
  

def aes192ECB_decrypt(ciphertext, key):

    # 检查密钥长度是否为24字节

    if len(key) != 24:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(52)]

    key_expansion192(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes192_decrypt(state, rk)

        plaintext += state

    print(plaintext)

  

def aes256_decrypt(state, rk):

  

    # 从第14轮开始解密

    inv_AddRoundKey(state, 14, rk)

    inv_ShiftRow(state)

    inv_SubBytes(state)

  

    # 从第13轮解密到第1轮

    for i in range(13, 0, -1):

        inv_AddRoundKey(state, i, rk)

        inv_MixColumns(state)

        inv_ShiftRow(state)

        inv_SubBytes(state)

    # 解密初始轮密钥加

    inv_AddRoundKey(state, 0, rk)

  
  

def aes256ECB_decrypt(ciphertext, key):

    # 检查密钥长度是否为32字节

    if len(key) != 32:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(60)]

    key_expansion256(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes256_decrypt(state, rk)

        plaintext += state

    print(plaintext)

  
  
  

def main():

    key = bytes([0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A])

    ciphertext = bytes([0x25,  0x1C,  0xCC,  0xA0,  0xD8,  0x56,  0x6F,  0x43,  0xC1,  0x5A,  0x1F,  0xD7,  0x5A,  0x21,  0x1A,  0x66,  0x6D,  0xE5,  0x50,  0xBE,  0xB6,  0xE8,  0x17,  0x57,  0x01,  0xCD,  0x4A,  0x07,  0x7B,  0x21,  0xA0,  0x20])

    aes128ECB_decrypt(ciphertext, key)

  

    key = bytes([ 0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38])

    ciphertext = bytes([0x10, 0xAB, 0xC7, 0x3E, 0xD7, 0xA7, 0x16, 0x36, 0xC5, 0x71, 0xB2, 0xBC, 0x38, 0xF5, 0xD9, 0x95, 0x09, 0xB0, 0xC1, 0x56, 0xFC, 0x4D, 0xBB, 0x92, 0x50, 0xC3, 0x85, 0x8F, 0x65, 0x15, 0x9C, 0x45 ])

    aes192ECB_decrypt(ciphertext, key)

  

    key = bytes([0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38,0x39,0x40,0x41,0x42,0x43,0x44,0x45,0x46 ])

    ciphertext = bytes([0xB6, 0x99, 0x22, 0x31, 0x3B, 0xC7, 0x83, 0x86, 0xD8, 0xB7, 0x04, 0x30, 0x3F, 0x0E, 0x2A, 0x1B, 0x63, 0x0E, 0x3F, 0xF2, 0xEA, 0x9A, 0xCF, 0xD3, 0x88, 0xC0, 0x39, 0xB5, 0x72, 0xBC, 0x58, 0x14])

    aes256ECB_decrypt(ciphertext, key)

  

if __name__ == "__main__":

    main()
```


# 解密脚本整合(包含CBC模式)
仅包含 ecb,cbc模式下的128,192,256加密模式

## C语言
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

// S盒
uint8_t Sbox[256] = {
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,
    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,
    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,
    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,
    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,
    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,
    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,
    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,
    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,
    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,
    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,
    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,
    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,
    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,
    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,
    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16
};

// 逆S盒 - 在GenerateInvSbox函数中生成
uint8_t invSbox[256] =
{
    0
};


// 轮常数 (前10个)
uint32_t Rcon[10] = {
    0x01000000, 0x02000000,
    0x04000000, 0x08000000,
    0x10000000, 0x20000000,
    0x40000000, 0x80000000,
    0x1b000000, 0x36000000 };

// 列混合所用矩阵
uint8_t colM[4][4] =
{
    2, 3, 1, 1,
    1, 2, 3, 1,
    1, 1, 2, 3,
    3, 1, 1, 2
};

// 逆矩阵
uint8_t inv_colM[4][4] =
{
    0xe, 0xb, 0xd, 0x9,
    0x9, 0xe, 0xb, 0xd,
    0xd, 0x9, 0xe, 0xb,
    0xb, 0xd, 0x9, 0xe
};

// 将4字节串转化为4字节整数(大端序)
uint32_t BytesToIntB(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) |
        ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) |
        ((uint32_t)bytes[3]);
}


// 列混合所用辅助函数
//GF(2⁸) 乘法实现:
uint8_t GFMul2(uint8_t s)
{
    uint8_t result = s << 1;
    if (s & 0x80)
    {
        result ^= 0x1b;
    }
    return result;
}

uint8_t GFMul3(uint8_t s)
{
    return GFMul2(s) ^ s;
}
uint8_t GFMul4(uint8_t s)
{
    return GFMul2(GFMul2(s));
}
uint8_t GFMul8(uint8_t s)
{
    return GFMul2(GFMul4(s));
}
uint8_t GFMul9(uint8_t s)
{
    return GFMul8(s) ^ s;
}
uint8_t GFMul11(uint8_t s)
{
    return GFMul9(s) ^ GFMul2(s);
}
uint8_t GFMul12(uint8_t s)
{
    return GFMul8(s) ^ GFMul4(s);
}
uint8_t GFMul13(uint8_t s)
{
    return GFMul12(s) ^ s;
}
uint8_t GFMul14(uint8_t s)
{
    return GFMul12(s) ^ GFMul2(s);
}

uint8_t GFMul(int n, uint8_t s)
{
    switch (n)
    {
    case 1:
        return s;
    case 2:
        return GFMul2(s);
    case 3:
        return GFMul3(s);
    case 4:
        return GFMul4(s);
    case 8:
        return GFMul8(s);
    case 9:
        return GFMul9(s);
    case 11:
        return GFMul11(s);
    case 12:
        return GFMul12(s);
    case 13:
        return GFMul13(s);
    case 14:
        return GFMul14(s);
    default:
        printf("not found target GFMul number\r\n");
        system("pause");
        exit(EXIT_FAILURE);
    }
}



// 将4字节整数数据转化为字节串
void IntToBytesB(uint8_t* bytes, uint32_t intData)
{
    bytes[0] = (intData >> 24) & 0xff;
    bytes[1] = (intData >> 16) & 0xff;
    bytes[2] = (intData >> 8) & 0xff;
    bytes[3] = intData & 0xff;
}

// 128位密钥扩展中的T函数
uint32_t T(uint32_t num, uint32_t round)
{

    // 子循环: 将1个字中的4个字节循环左移1个字节。
    uint8_t t1 = (num >> 16) & 0xff;
    uint8_t t2 = (num >> 8) & 0xff;
    uint8_t t3 = num & 0xff;
    uint8_t t4 = (num >> 24) & 0xff;

    // 字节代换: 用S盒进行字节代换
    t1 = Sbox[t1];
    t2 = Sbox[t2];
    t3 = Sbox[t3];
    t4 = Sbox[t4];

    // 转化为4字节大端序整数
    uint32_t result =
        ((uint32_t)t1 << 24) |
        ((uint32_t)t2 << 16) |
        ((uint32_t)t3 << 8) |
        ((uint32_t)t4);

    // 轮常量异或: 与同轮常量Rcon[round / 4 - 1]异或
    result ^= Rcon[round / 4 - 1];

    return result;
}

// 128位密钥扩展
void key_expansion128(uint8_t* key, uint32_t* rk)
{
    // 将16字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 4; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展10轮密钥
    for (int i = 4; i < 44; i++)
    {
        // 如果i为4的整数倍需要进行T函数处理
        if (i % 4 == 0)
        {
            rk[i] = rk[i - 4] ^ T(rk[i - 1], i);
        }
        else
        {
            rk[i] = rk[i - 4] ^ rk[i - 1];
        }
    }

}

// 192位密钥扩展
void key_expansion192(uint8_t* key, uint32_t* rk)
{
    // 将24字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 6; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展10轮密钥
    for (int i = 6; i < 52; i++)
    {
        // i为6整数倍
        if (i % 6 == 0)
        {
            // 左移一位，字节代换，异或轮常数
            uint8_t t0 = Sbox[(rk[i - 1] >> 16) & 0xff];
            uint8_t t1 = Sbox[(rk[i - 1] >> 8) & 0xff];
            uint8_t t2 = Sbox[rk[i - 1] & 0xff];
            uint8_t t3 = Sbox[(rk[i - 1] >> 24) & 0xff];

            uint32_t temp =
                (uint32_t)t0 << 24 |
                (uint32_t)t1 << 16 |
                (uint32_t)t2 << 8 |
                (uint32_t)t3;

            temp ^= Rcon[i / 6 - 1];

            rk[i] = rk[i - 6] ^ temp;
        }
        else
        {
            rk[i] = rk[i - 1] ^ rk[i - 6];
        }
    }

}

// 256位密钥扩展
void key_expansion256(uint8_t* key, uint32_t* rk)
{
    // 将32字节key以4字节大端序初始化存储在rk中
    for (int i = 0; i < 8; i++)
    {
        rk[i] = BytesToIntB(&key[i * 4]);
    }

    // 扩展14轮密钥
    for (int i = 8; i < 60; i++)
    {
        // i为8整数倍
        if (i % 8 == 0)
        {
            // 左移一位，字节代换，异或轮常数
            uint8_t t0 = Sbox[(rk[i - 1] >> 16) & 0xff];
            uint8_t t1 = Sbox[(rk[i - 1] >> 8) & 0xff];
            uint8_t t2 = Sbox[rk[i - 1] & 0xff];
            uint8_t t3 = Sbox[(rk[i - 1] >> 24) & 0xff];

            uint32_t temp =
                (uint32_t)t0 << 24 |
                (uint32_t)t1 << 16 |
                (uint32_t)t2 << 8 |
                (uint32_t)t3;

            temp ^= Rcon[i / 8 - 1];

            rk[i] = rk[i - 8] ^ temp;
        }
        else if (i % 8 == 4)
        {
            // aes256加密特有: 仅进行字节代换
            uint8_t t0 = Sbox[(rk[i - 1] >> 24) & 0xff];
            uint8_t t1 = Sbox[(rk[i - 1] >> 16) & 0xff];
            uint8_t t2 = Sbox[(rk[i - 1] >> 8) & 0xff];
            uint8_t t3 = Sbox[rk[i - 1] & 0xff];

            uint32_t temp =
                (uint32_t)t0 << 24 |
                (uint32_t)t1 << 16 |
                (uint32_t)t2 << 8 |
                (uint32_t)t3;

            rk[i] = rk[i - 8] ^ temp;
        }
        else
        {
            rk[i] = rk[i - 1] ^ rk[i - 8];
        }
    }

}

// 生成逆S盒
void GenerateInvSbox()
{
    for (int i = 0; i < 256; i++)
    {
        invSbox[Sbox[i]] = i;
    }
}

// 逆字节代换
void invSubBytes(uint8_t inputBuffer[4][4])
{
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] = invSbox[inputBuffer[i][j]];
        }
    }
}

// 逆行移位
void invShiftRow(uint8_t inputBuffer[4][4])
{
    uint8_t temp[4] = { 0 };
    // 第一行不移位

    // 第二行右移1位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[1][i];
    }
    inputBuffer[1][0] = temp[3];
    inputBuffer[1][1] = temp[0];
    inputBuffer[1][2] = temp[1];
    inputBuffer[1][3] = temp[2];

    // 第三行右移2位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[2][i];
    }
    inputBuffer[2][0] = temp[2];
    inputBuffer[2][1] = temp[3];
    inputBuffer[2][2] = temp[0];
    inputBuffer[2][3] = temp[1];

    // 第四行右移3位
    for (int i = 0; i < 4; i++)
    {
        temp[i] = inputBuffer[3][i];
    }
    inputBuffer[3][0] = temp[1];
    inputBuffer[3][1] = temp[2];
    inputBuffer[3][2] = temp[3];
    inputBuffer[3][3] = temp[0];

}

// 逆列混合
void invMixColumns(uint8_t inputBuffer[4][4])
{
    uint8_t tempArray[4][4];
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            tempArray[i][j] = inputBuffer[i][j];
        }
    }

    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            inputBuffer[i][j] =
                GFMul(inv_colM[i][0], tempArray[0][j]) ^
                GFMul(inv_colM[i][1], tempArray[1][j]) ^
                GFMul(inv_colM[i][2], tempArray[2][j]) ^
                GFMul(inv_colM[i][3], tempArray[3][j]);
        }
    }
}

// 加解密所用轮密钥加相同
void AddRoundKey(uint8_t intputBuffer[4][4], uint32_t* rk ,int round)
{
    uint8_t tempArray[4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        IntToBytesB(tempArray, rk[round * 4 + i]);
        for (int j = 0; j < 4; j++)
        {
            intputBuffer[j][i] ^= tempArray[j];
        }
    }
}

void aesDecrypt_128(uint8_t* ciphertext, uint32_t* rk, uint8_t* plaintext)
{
    // 将密文转化为密文矩阵
    uint8_t cipherArray[4][4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            cipherArray[j][i] = ciphertext[i * 4 + j];
        }
    }

    // 从第10轮加密开始逆解密
    AddRoundKey(cipherArray, rk, 10);
    invShiftRow(cipherArray);
    invSubBytes(cipherArray);

    // 从第9轮解密到第1轮
    for (int i = 9; i > 0; i--)
    {
        AddRoundKey(cipherArray, rk, i);
        invMixColumns(cipherArray);
        invShiftRow(cipherArray);
        invSubBytes(cipherArray);
    }

    // 初始轮密钥加
    AddRoundKey(cipherArray, rk, 0);

    // 将明文矩阵存储到明文数组中
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            plaintext[i * 4 + j] = cipherArray[j][i];
        }
    }
}

void aesDecrypt_192(uint8_t* ciphertext, uint32_t* rk, uint8_t* plaintext)
{
    // 将密文转化为密文矩阵
    uint8_t cipherArray[4][4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            cipherArray[j][i] = ciphertext[i * 4 + j];
        }
    }

    // 从第10轮加密开始逆解密
    AddRoundKey(cipherArray, rk, 12);
    invShiftRow(cipherArray);
    invSubBytes(cipherArray);

    // 从第9轮解密到第1轮
    for (int i = 11; i > 0; i--)
    {
        AddRoundKey(cipherArray, rk, i);
        invMixColumns(cipherArray);
        invShiftRow(cipherArray);
        invSubBytes(cipherArray);
    }

    // 初始轮密钥加
    AddRoundKey(cipherArray, rk, 0);

    // 将明文矩阵存储到明文数组中
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            plaintext[i * 4 + j] = cipherArray[j][i];
        }
    }
}

void aesDecrypt_256(uint8_t* ciphertext, uint32_t* rk, uint8_t* plaintext)
{
    // 将密文转化为密文矩阵
    uint8_t cipherArray[4][4] = { 0 };
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            cipherArray[j][i] = ciphertext[i * 4 + j];
        }
    }

    // 从第14轮加密开始逆解密
    AddRoundKey(cipherArray, rk, 14);
    invShiftRow(cipherArray);
    invSubBytes(cipherArray);

    // 从第13轮解密到第1轮
    for (int i = 13; i > 0; i--)
    {
        AddRoundKey(cipherArray, rk, i);
        invMixColumns(cipherArray);
        invShiftRow(cipherArray);
        invSubBytes(cipherArray);
    }

    // 初始轮密钥加
    AddRoundKey(cipherArray, rk, 0);

    // 将明文矩阵存储到明文数组中
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            plaintext[i * 4 + j] = cipherArray[j][i];
        }
    }
}

void aesDecryptECB_128(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext)
{
    // 检查密钥长度是否为16字节
    if (key_len != 16)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[44] = { 0 };
    key_expansion128(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_128(&ciphertext[i * 16], rk, &plaintext[i * 16]);
    }

}

void aesDecryptECB_192(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext)
{
    // 检查密钥长度是否为16字节
    if (key_len != 24)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[52] = { 0 };
    key_expansion192(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_192(&ciphertext[i * 16], rk, &plaintext[i * 16]);
    }

}

void aesDecryptECB_256(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext)
{
    // 检查密钥长度是否为32字节
    if (key_len != 32)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[60] = { 0 };
    key_expansion256(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_256(&ciphertext[i * 16], rk, &plaintext[i * 16]);
    }
}

void aesDecryptCBC_128(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext, uint8_t* IV)
{
    // 检查密钥长度是否为16字节
    if (key_len != 16)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[44] = { 0 };
    key_expansion128(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_128(&ciphertext[i * 16], rk, &plaintext[i * 16]);
        if (i == 0)
        {
            for (int j = 0; j < 16; j++)
            {
                plaintext[j] ^= IV[j];
            }
        }
        else
        {
            for (int j = 0; j < 16; j++)
            {
                plaintext[i * 16 + j] ^= ciphertext[(i - 1) * 16 + j];
            }
        }
    }

}

void aesDecryptCBC_192(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext, uint8_t* IV)
{
    // 检查密钥长度是否为24字节
    if (key_len != 24)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[52] = { 0 };
    key_expansion192(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_192(&ciphertext[i * 16], rk, &plaintext[i * 16]);
        if (i == 0)
        {
            for (int j = 0; j < 16; j++)
            {
                plaintext[j] ^= IV[j];
            }
        }
        else
        {
            for (int j = 0; j < 16; j++)
            {
                plaintext[i * 16 + j] ^= ciphertext[(i - 1) * 16 + j];
            }
        }
    }

}

void aesDecryptCBC_256(uint8_t* ciphertext, uint8_t* key, int cipher_len, int key_len, uint8_t* plaintext, uint8_t* IV)
{
    // 检查密钥长度是否为32字节
    if (key_len != 32)
    {
        printf("Wrong Key Length\r\n");
        return;
    }

    // 检查密文长度是否为16字节整数倍
    if (cipher_len % 16 != 0)
    {
        printf("Wrong ciphertext length\r\n");
        return;
    }

    // 密钥扩展
    uint32_t rk[60] = { 0 };
    key_expansion256(key, rk);

    // 生成逆S盒
    GenerateInvSbox();

    int blockSize = cipher_len / 16;

    for (int i = 0; i < blockSize; i++)
    {
        aesDecrypt_256(&ciphertext[i * 16], rk, &plaintext[i * 16]);
        if (i == 0)
        {
            for (int j = 0; j < 16; j++)
            {
                plaintext[j] ^= IV[j];
            }
        }
        else
        {
            for (int j = 0; j < 16; j++)
            {
                plaintext[i * 16 + j] ^= ciphertext[(i - 1) * 16 + j];
            }
        }
    }

}


int main()
{
    uint8_t key[32] = { 0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38,0x39,0x40,0x41,0x42,0x43,0x44,0x45,0x46 };
    uint8_t ciphertext[32] = { 0x84,0xE7,0x59,0x62,0xB5,0xBA,0x99,0x58,0xDD,0xAE,0x56,0xB4,0xE4,0x0D,0xF4,0x4F,0x8D,0xDC,0xC9,0x7D,0x46,0xAD,0x80,0xC3,0x17,0x67,0xD8,0x18,0x2C,0xA9,0xB4,0x26 };
    uint8_t IV[16] = { 0x45,0x84,0x43,0xEB,0x29,0xE5,0xEC,0x52,0x30,0x58,0xC0,0x6F,0xA9,0xB6,0x2A,0xA9 };
    uint8_t plaintext[0xff] = { 0 };


    int key_len = sizeof(key) / sizeof(key[0]);
    int cipher_len = sizeof(ciphertext) / sizeof(ciphertext[0]);

    aesDecryptCBC_256(ciphertext, key, cipher_len, key_len, plaintext, IV);
    printf("%s\r\n", plaintext);

	return 0;
}

```

## python
```python
# S盒

Sbox = [

    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,

    0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,

    0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,

    0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,

    0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,

    0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,

    0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,

    0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,

    0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,

    0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,

    0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,

    0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,

    0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,

    0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,

    0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,

    0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16

]

  

# 逆S盒

inv_Sbox = [0 for _ in range(256)]

  

# 轮常数

Rcon = [

    0x01000000, 0x02000000,

    0x04000000, 0x08000000,

    0x10000000, 0x20000000,

    0x40000000, 0x80000000,

    0x1b000000, 0x36000000

]

  

# 列混合所用矩阵

colM = [

    2, 3, 1, 1,

    1, 2, 3, 1,

    1, 1, 2, 3,

    3, 1, 1, 2

]

  

# 解密列混合所用逆矩阵

inv_colM = [

    [0xe, 0xb, 0xd, 0x9],

    [0x9, 0xe, 0xb, 0xd],

    [0xd, 0x9, 0xe, 0xb],

    [0xb, 0xd, 0x9, 0xe]

]

  

# 列混合所用辅助函数: GF(2⁸) 乘法实现

def GFMul2(s):

    s = s & 0xff

    result = s << 1

    if (s & 0x80):

        result ^= 0x1b

    return result & 0xff

  

def GFMul3(s):

    s = s & 0xff

    return GFMul2(s) ^ s

  

def GFMul4(s):

    s = s & 0xff

    return GFMul2(GFMul2(s))

  

def GFMul8(s):

    s = s & 0xff

    return GFMul4(GFMul2(s))

  

def GFMul9(s):

    s = s & 0xff

    return GFMul8(s) ^ s

  

def GFMul11(s):

    s = s & 0xff

    return GFMul9(s) ^ GFMul2(s)

  

def GFMul12(s):

    s = s & 0xff

    return GFMul8(s) ^ GFMul4(s)

  

def GFMul13(s):

    s = s & 0xff

    return GFMul12(s) ^ s

  

def GFMul14(s):

    s = s & 0xff

    return GFMul12(s) ^ GFMul2(s)

  

def GFMul(n, s):

    match n:

        case 1:

            return s & 0xff

        case 2:

            return GFMul2(s)

        case 3:

            return GFMul3(s)

        case 4:

            return GFMul4(s)

        case 8:

            return GFMul8(s)

        case 9:

            return GFMul9(s)

        case 11:

            return GFMul11(s)

        case 12:

            return GFMul12(s)

        case 13:

            return GFMul13(s)

        case 14:

            return GFMul14(s)

        case _:

            print("Not found target GFMul value")

            return 0

  

def key_expansion128(key, rk):

    # 将16字节key以4字节大端序初始化存储在rk中

    for i in range(4):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展10轮密钥

    for i in range(4, 44, 1):

        # i为4的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 4 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 4 - 1]

            rk[i] = rk[i - 4] ^ temp

        else :

            rk[i] = rk[i - 4] ^ rk[i - 1]

  

def key_expansion192(key, rk):

    # 将24字节key以4字节大端序初始化存储在rk中

    for i in range(6):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展12轮密钥

    for i in range(6, 52, 1):

        # i为6的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 6 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 6 - 1]

            rk[i] = rk[i - 6] ^ temp

        else :

            rk[i] = rk[i - 6] ^ rk[i - 1]

  

def key_expansion256(key, rk):

    # 将32字节key以4字节大端序初始化存储在rk中

    for i in range(8):

        rk[i] = int.from_bytes(key[i*4:i*4+4], "big")

    # 扩展14轮密钥

    for i in range(8, 60, 1):

        # i为6的整数倍时执行左移1位->字节代换->与轮常数异或

        if (i % 8 == 0):

            # 左移1位的同时字节代换

            t0 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t2 = Sbox[(rk[i - 1]) & 0xff]

            t3 = Sbox[(rk[i - 1] >> 24) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

            # 与常轮数异或

            temp ^= Rcon[i // 8 - 1]

            rk[i] = rk[i - 8] ^ temp

  

        # 256加密特有:i % 8 == 4时只执行字节代换

        elif (i % 8 == 4):

            t0 = Sbox[(rk[i - 1] >> 24) & 0xff]

            t1 = Sbox[(rk[i - 1] >> 16) & 0xff]

            t2 = Sbox[(rk[i - 1] >> 8) & 0xff]

            t3 = Sbox[(rk[i - 1]) & 0xff]

            # 转化为4字节整数

            temp = (t0 << 24) | (t1 << 16) | (t2 << 8) | t3

  

            rk[i] = rk[i - 8] ^ temp

  

        else :

            rk[i] = rk[i - 8] ^ rk[i - 1]

  

# 生成逆S盒

def generate_inv_Sbox():

    for i in range(256):

        inv_Sbox[Sbox[i]] = i

  

# 逆字节代换

def inv_SubBytes(state):

    for i in range(16):

        state[i] = inv_Sbox[state[i]]

# 逆行移位

def inv_ShiftRow(state):

    # 第一行不移位

  

    # 第二行右移1位

    state[1], state[5], state[9], state[13] = state[13], state[1], state[5], state[9]

  

    # 第三行右移2位

    state[2], state[6], state[10], state[14] = state[10], state[14], state[2], state[6]

  

    # 第四行右移3位

    state[3], state[7], state[11], state[15] = state[7], state[11], state[15], state[3]

  

# 逆列混合

def inv_MixColumns(state):

    temp_array = []

    for i in range(16):

        temp_array.append(state[i])

  

    # 逆矩阵乘密文矩阵

    for i in range(4):

        for j in range(4):

            state[i + j * 4] = (

                GFMul(inv_colM[i][0], temp_array[j * 4 + 0]) ^

                GFMul(inv_colM[i][1], temp_array[j * 4 + 1]) ^

                GFMul(inv_colM[i][2], temp_array[j * 4 + 2]) ^

                GFMul(inv_colM[i][3], temp_array[j * 4 + 3])

            )

  

# 逆轮密钥加

def inv_AddRoundKey(state, round, rk):

    t0 = (int.from_bytes(state[0:4], "big") ^ rk[round * 4]) & 0xffffffff

    t1 = (int.from_bytes(state[4:8], "big") ^ rk[round * 4 + 1]) & 0xffffffff

    t2 = (int.from_bytes(state[8:12], "big") ^ rk[round * 4 + 2]) & 0xffffffff

    t3 = (int.from_bytes(state[12:16], "big") ^ rk[round * 4 + 3]) & 0xffffffff

  

    temp_bytes = t0.to_bytes(4,"big")

    temp_bytes += t1.to_bytes(4,"big")

    temp_bytes += t2.to_bytes(4,"big")

    temp_bytes += t3.to_bytes(4,"big")

  

    for i in range(16):

        state[i] = temp_bytes[i]

  

def aes128_decrypt(state, rk):

  

    # 从第10轮开始解密

    inv_AddRoundKey(state, 10, rk)

    inv_ShiftRow(state)

    inv_SubBytes(state)

  

    # 从第9轮解密到第1轮

    for i in range(9, 0, -1):

        inv_AddRoundKey(state, i, rk)

        inv_MixColumns(state)

        inv_ShiftRow(state)

        inv_SubBytes(state)

    # 解密初始轮密钥加

    inv_AddRoundKey(state, 0, rk)

  

def aes192_decrypt(state, rk):

  

    # 从第12轮开始解密

    inv_AddRoundKey(state, 12, rk)

    inv_ShiftRow(state)

    inv_SubBytes(state)

  

    # 从第11轮解密到第1轮

    for i in range(11, 0, -1):

        inv_AddRoundKey(state, i, rk)

        inv_MixColumns(state)

        inv_ShiftRow(state)

        inv_SubBytes(state)

    # 解密初始轮密钥加

    inv_AddRoundKey(state, 0, rk)

  

def aes256_decrypt(state, rk):

  

    # 从第14轮开始解密

    inv_AddRoundKey(state, 14, rk)

    inv_ShiftRow(state)

    inv_SubBytes(state)

  

    # 从第13轮解密到第1轮

    for i in range(13, 0, -1):

        inv_AddRoundKey(state, i, rk)

        inv_MixColumns(state)

        inv_ShiftRow(state)

        inv_SubBytes(state)

    # 解密初始轮密钥加

    inv_AddRoundKey(state, 0, rk)

  
  

def aes128ECB_decrypt(ciphertext, key):

    # 检查密钥长度是否为16字节

    if len(key) != 16:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(44)]

    key_expansion128(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes128_decrypt(state, rk)

        plaintext += state

    print(plaintext)

  

def aes192ECB_decrypt(ciphertext, key):

    # 检查密钥长度是否为24字节

    if len(key) != 24:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(52)]

    key_expansion192(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes192_decrypt(state, rk)

        plaintext += state

    print(plaintext)

  

def aes256ECB_decrypt(ciphertext, key):

    # 检查密钥长度是否为32字节

    if len(key) != 32:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(60)]

    key_expansion256(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes256_decrypt(state, rk)

        plaintext += state

    print(plaintext)

  

def aes128CBC_decrypt(ciphertext, key, IV):

    # 检查密钥长度是否为16字节

    if len(key) != 16:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(44)]

    key_expansion128(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes128_decrypt(state, rk)

        if (i == 0):

            for j in range(16):

                state[j] ^= IV[j]

        else:

            for j in range(16):

                state[j] ^= ciphertext[(i - 1) * 16 + j]

  

        plaintext += state

    print(plaintext)

  

def aes192CBC_decrypt(ciphertext, key, IV):

    # 检查密钥长度是否为24字节

    if len(key) != 24:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(52)]

    key_expansion192(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes192_decrypt(state, rk)

        if (i == 0):

            for j in range(16):

                state[j] ^= IV[j]

        else:

            for j in range(16):

                state[j] ^= ciphertext[(i - 1) * 16 + j]

  

        plaintext += state

    print(plaintext)

  

def aes256CBC_decrypt(ciphertext, key, IV):

    # 检查密钥长度是否为32字节

    if len(key) != 32:

        print("Key Length Wrong")

        return

    # 检查密文是否为16字节整数倍

    if len(ciphertext) % 16 != 0:

        print("ciphertext length wrong")

    # 密钥扩展

    rk = [0 for _ in range(60)]

    key_expansion256(key, rk)

  

    # 生成逆S盒

    generate_inv_Sbox()

  

    block_size = len(ciphertext) // 16 # 计算分组数量

    plaintext = b''              # 存储明文

    for i in range(block_size):

        state = bytearray(ciphertext[i*16:i*16+16])

        aes256_decrypt(state, rk)

        if (i == 0):

            for j in range(16):

                state[j] ^= IV[j]

        else:

            for j in range(16):

                state[j] ^= ciphertext[(i - 1) * 16 + j]

  

        plaintext += state

    print(plaintext)

  

def main():

    key = bytes([0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A])

    ciphertext = bytes([0x25,  0x1C,  0xCC,  0xA0,  0xD8,  0x56,  0x6F,  0x43,  0xC1,  0x5A,  0x1F,  0xD7,  0x5A,  0x21,  0x1A,  0x66,  0x6D,  0xE5,  0x50,  0xBE,  0xB6,  0xE8,  0x17,  0x57,  0x01,  0xCD,  0x4A,  0x07,  0x7B,  0x21,  0xA0,  0x20])

    aes128ECB_decrypt(ciphertext, key)

  

    key = bytes([ 0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38])

    ciphertext = bytes([0x10, 0xAB, 0xC7, 0x3E, 0xD7, 0xA7, 0x16, 0x36, 0xC5, 0x71, 0xB2, 0xBC, 0x38, 0xF5, 0xD9, 0x95, 0x09, 0xB0, 0xC1, 0x56, 0xFC, 0x4D, 0xBB, 0x92, 0x50, 0xC3, 0x85, 0x8F, 0x65, 0x15, 0x9C, 0x45 ])

    aes192ECB_decrypt(ciphertext, key)

  

    key = bytes([0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38,0x39,0x40,0x41,0x42,0x43,0x44,0x45,0x46 ])

    ciphertext = bytes([0xB6, 0x99, 0x22, 0x31, 0x3B, 0xC7, 0x83, 0x86, 0xD8, 0xB7, 0x04, 0x30, 0x3F, 0x0E, 0x2A, 0x1B, 0x63, 0x0E, 0x3F, 0xF2, 0xEA, 0x9A, 0xCF, 0xD3, 0x88, 0xC0, 0x39, 0xB5, 0x72, 0xBC, 0x58, 0x14])

    aes256ECB_decrypt(ciphertext, key)

  

    key = bytes([0x54, 0x68, 0x75, 0x72, 0x73, 0x64, 0x61, 0x79, 0x56, 0x49, 0x76, 0x6F, 0x35, 0x30, 0x6A, 0x6A, 0x31,0x32,0x33,0x34,0x35,0x36,0x37,0x38,0x39,0x40,0x41,0x42,0x43,0x44,0x45,0x46])

    ciphertext = bytes([0x84,0xE7,0x59,0x62,0xB5,0xBA,0x99,0x58,0xDD,0xAE,0x56,0xB4,0xE4,0x0D,0xF4,0x4F,0x8D,0xDC,0xC9,0x7D,0x46,0xAD,0x80,0xC3,0x17,0x67,0xD8,0x18,0x2C,0xA9,0xB4,0x26])

    IV = bytes([0x45,0x84,0x43,0xEB,0x29,0xE5,0xEC,0x52,0x30,0x58,0xC0,0x6F,0xA9,0xB6,0x2A,0xA9])

    aes256CBC_decrypt(ciphertext, key, IV)

  

if __name__ == "__main__":

    main()
```