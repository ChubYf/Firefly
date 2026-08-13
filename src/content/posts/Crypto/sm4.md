---
title: sm4
published: 2026-07-05
description: sm4加解密算法实现
image: ./cover.jpg
tags: [逆向]
category: Crypto
draft: false
---


# ecb加密模式
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

uint8_t Sbox[256] = {
    0xd6,0x90,0xe9,0xfe,0xcc,0xe1,0x3d,0xb7,0x16,0xb6,0x14,0xc2,0x28,0xfb,0x2c,0x05,
    0x2b,0x67,0x9a,0x76,0x2a,0xbe,0x04,0xc3,0xaa,0x44,0x13,0x26,0x49,0x86,0x06,0x99,
    0x9c,0x42,0x50,0xf4,0x91,0xef,0x98,0x7a,0x33,0x54,0x0b,0x43,0xed,0xcf,0xac,0x62,
    0xe4,0xb3,0x1c,0xa9,0xc9,0x08,0xe8,0x95,0x80,0xdf,0x94,0xfa,0x75,0x8f,0x3f,0xa6,
    0x47,0x07,0xa7,0xfc,0xf3,0x73,0x17,0xba,0x83,0x59,0x3c,0x19,0xe6,0x85,0x4f,0xa8,
    0x68,0x6b,0x81,0xb2,0x71,0x64,0xda,0x8b,0xf8,0xeb,0x0f,0x4b,0x70,0x56,0x9d,0x35,
    0x1e,0x24,0x0e,0x5e,0x63,0x58,0xd1,0xa2,0x25,0x22,0x7c,0x3b,0x01,0x21,0x78,0x87,
    0xd4,0x00,0x46,0x57,0x9f,0xd3,0x27,0x52,0x4c,0x36,0x02,0xe7,0xa0,0xc4,0xc8,0x9e,
    0xea,0xbf,0x8a,0xd2,0x40,0xc7,0x38,0xb5,0xa3,0xf7,0xf2,0xce,0xf9,0x61,0x15,0xa1,
    0xe0,0xae,0x5d,0xa4,0x9b,0x34,0x1a,0x55,0xad,0x93,0x32,0x30,0xf5,0x8c,0xb1,0xe3,
    0x1d,0xf6,0xe2,0x2e,0x82,0x66,0xca,0x60,0xc0,0x29,0x23,0xab,0x0d,0x53,0x4e,0x6f,
    0xd5,0xdb,0x37,0x45,0xde,0xfd,0x8e,0x2f,0x03,0xff,0x6a,0x72,0x6d,0x6c,0x5b,0x51,
    0x8d,0x1b,0xaf,0x92,0xbb,0xdd,0xbc,0x7f,0x11,0xd9,0x5c,0x41,0x1f,0x10,0x5a,0xd8,
    0x0a,0xc1,0x31,0x88,0xa5,0xcd,0x7b,0xbd,0x2d,0x74,0xd0,0x12,0xb8,0xe5,0xb4,0xb0,
    0x89,0x69,0x97,0x4a,0x0c,0x96,0x77,0x7e,0x65,0xb9,0xf1,0x09,0xc5,0x6e,0xc6,0x84,
    0x18,0xf0,0x7d,0xec,0x3a,0xdc,0x4d,0x20,0x79,0xee,0x5f,0x3e,0xd7,0xcb,0x39,0x48
};

/* 系统参数 FK */
uint32_t FK[4] = {
    0xa3b1bac6, 0x56aa3350, 0x677d9197, 0xb27022dc
};

/* 固定参数 CK（32个） */
uint32_t CK[32] = {
    0x00070e15,0x1c232a31,0x383f464d,0x545b6269,
    0x70777e85,0x8c939aa1,0xa8afb6bd,0xc4cbd2d9,
    0xe0e7eef5,0xfc030a11,0x181f262d,0x343b4249,
    0x50575e65,0x6c737a81,0x888f969d,0xa4abb2b9,
    0xc0c7ced5,0xdce3eaf1,0xf8ff060d,0x141b2229,
    0x30373e45,0x4c535a61,0x686f767d,0x848b9299,
    0xa0a7aeb5,0xbcc3cad1,0xd8dfe6ed,0xf4fb0209,
    0x10171e25,0x2c333a41,0x484f565d,0x646b7279
};

//32位循环左移
uint32_t rol32(uint32_t value, uint32_t shift)
{
	return (value << shift) | (value >> (32 - shift));
}

//32位循环右移
uint32_t ror32(uint32_t value, uint32_t shift)
{
	return (value >> shift) | (value << (32 - shift));
}

// 将4字节数组转为uint32_t整数（大端序)
uint32_t BytesToInt(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) | ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) | ((uint32_t)bytes[3]);
}

// 将uint32_t整数写入4字节数组（大端序）
void IntToBytes(uint8_t* bytes, uint32_t data)
{
    bytes[0] = (uint8_t)(data >> 24);
    bytes[1] = (uint8_t)(data >> 16);
    bytes[2] = (uint8_t)(data >> 8);
    bytes[3] = (uint8_t)(data);
}

// S盒替换：输入32位，输出32位
uint32_t SboxReplace(uint32_t input)
{
    uint32_t output = 0;
    output |= (uint32_t)Sbox[(input >> 24) & 0xff] << 24;
    output |= (uint32_t)Sbox[(input >> 16) & 0xFF] << 16;
    output |= (uint32_t)Sbox[(input >> 8) & 0xFF] << 8;
    output |= (uint32_t)Sbox[input & 0xFF];
    return output;
}

// 合成置换T
uint32_t TReplace(uint32_t input)
{
    uint32_t output = SboxReplace(input);
    return output ^ rol32(output, 2) ^ rol32(output, 10) ^
        rol32(output, 18) ^ rol32(output, 24);
}

// 密钥扩展合成置换T'
uint32_t KeyTReplace(uint32_t input)
{
    uint32_t output = SboxReplace(input);
    return output ^ rol32(output, 13) ^ rol32(output, 23);
}

// 密钥key扩展后生成32轮密钥rk
void KeyExpansion(const uint8_t* key, uint32_t* rk)
{
    uint32_t K[36]; // K[0] ~ K[3]为初始状态, K[4] ~ K[35] 为中间状态
    
    // 1. 将 16 字节密钥转换为 4 个 32 位字（大端序）
    K[0] = BytesToInt((uint8_t*)&key[0]);
    K[1] = BytesToInt((uint8_t*)&key[4]);
    K[2] = BytesToInt((uint8_t*)&key[8]);
    K[3] = BytesToInt((uint8_t*)&key[12]);

    /* 2. 异或系统参数 FK */
    for (int i = 0; i < 4; i++)
    {
        K[i] ^= FK[i];
    }

    /* 3. 迭代生成轮密钥 */
    for (int i = 0; i < 32; i++)
    {
        uint32_t temp = K[i + 1] ^ K[i + 2] ^ K[i + 3] ^ CK[i];
        K[i + 4] = K[i] ^ KeyTReplace(temp);
        rk[i] = K[i + 4];
    }

}

// PKCS#7 填充 如果待加密数据不是16字节的整数倍，就需要在加密前进行填充，使其达到16字节的倍数
int pkcs7_padding(uint8_t* data, int data_len, int block_size)
{
    // SM4 block_size 固定为16
    // 1. 计算需要填充的字节数 
    uint8_t pad_len = block_size - (data_len % block_size);

    // 2. 在原始数据末尾追加 pad_len 个字节，每个字节的值都是 pad_len
    for (int i = 0; i < pad_len; i++)
    {
        data[data_len + i] = pad_len;
    }

    // 3. 返回填充后的新数据长度
    return data_len + pad_len;
}

void sm4_Encrypt(uint8_t* input, uint32_t* rk, uint8_t* encryptData)
{
    uint32_t x[36] = { 0 };

    /* 将128位输入分为4个32位字 */
    x[0] = BytesToInt((uint8_t*)&input[0]);
    x[1] = BytesToInt((uint8_t*)&input[4]);
    x[2] = BytesToInt((uint8_t*)&input[8]);
    x[3] = BytesToInt((uint8_t*)&input[12]);

    /* 32轮迭代 */
    for (int i = 0; i < 32; i++)
    {
        x[i + 4] = x[i] ^ TReplace(x[i + 1] ^ x[i + 2] ^ x[i + 3] ^ rk[i]);
    }

    // 反序输出,并将4字节转化为1字节存储在encryptData中
    IntToBytes((uint8_t*)&encryptData[0], x[35]);
    IntToBytes((uint8_t*)&encryptData[4], x[34]);
    IntToBytes((uint8_t*)&encryptData[8], x[33]);
    IntToBytes((uint8_t*)&encryptData[12], x[32]);
}

void ECBEncrypt(uint8_t* inputBuffer, uint8_t* key, int input_len, uint8_t* encryptData)
{
    uint32_t rk[32] = { 0 };
    KeyExpansion(key, rk);// 密钥扩展

    int total_len = pkcs7_padding(inputBuffer, input_len, 16);// pkcs7填充明文

    int blocks = total_len / 16; // 计算一共有多少组
    
    // 每次加密一组(16字节)，并把加密数据存放再encryptData中
    for (int i = 0; i < blocks; i++)
    {
        sm4_Encrypt((uint8_t*)&inputBuffer[i * 16], rk, (uint8_t*)&encryptData[i * 16]);
    }
    

}

int main()
{
    // 固定16字节密钥
    uint8_t key[16] = { 98,108,97,99,107,109,97,110,98,97,54,54,54,54,54,54 };
    uint8_t encryptData[0xff] = { 0 }; // 存放加密后的数据
    uint8_t ciphertext[] = { 141,29,120,62,65,51,144,223,11,4,251,202,83,153,238,37,120,2,152,45,39,85,6,113,138,137,75,173,110,8,215,43 };// 密文

    uint8_t inputBuffer[0xff] = { 0 };
    printf("Please input your flag: ");
    scanf("%s", inputBuffer);
    
    int input_len = strlen((char*)inputBuffer);
    if (input_len != 28)
    {
        printf("wrong length\r\n");
        system("pause");
        return 0;
    }

    ECBEncrypt(inputBuffer, key, input_len, encryptData);

    for (int i = 0; i < input_len; i++)
    {
        if (encryptData[i] != ciphertext[i])
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



# ecb解密
解密算法与加密算法相同，只需把轮密钥逆序输入即可
## c语言实现
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

uint8_t Sbox[256] = {
    0xd6,0x90,0xe9,0xfe,0xcc,0xe1,0x3d,0xb7,0x16,0xb6,0x14,0xc2,0x28,0xfb,0x2c,0x05,
    0x2b,0x67,0x9a,0x76,0x2a,0xbe,0x04,0xc3,0xaa,0x44,0x13,0x26,0x49,0x86,0x06,0x99,
    0x9c,0x42,0x50,0xf4,0x91,0xef,0x98,0x7a,0x33,0x54,0x0b,0x43,0xed,0xcf,0xac,0x62,
    0xe4,0xb3,0x1c,0xa9,0xc9,0x08,0xe8,0x95,0x80,0xdf,0x94,0xfa,0x75,0x8f,0x3f,0xa6,
    0x47,0x07,0xa7,0xfc,0xf3,0x73,0x17,0xba,0x83,0x59,0x3c,0x19,0xe6,0x85,0x4f,0xa8,
    0x68,0x6b,0x81,0xb2,0x71,0x64,0xda,0x8b,0xf8,0xeb,0x0f,0x4b,0x70,0x56,0x9d,0x35,
    0x1e,0x24,0x0e,0x5e,0x63,0x58,0xd1,0xa2,0x25,0x22,0x7c,0x3b,0x01,0x21,0x78,0x87,
    0xd4,0x00,0x46,0x57,0x9f,0xd3,0x27,0x52,0x4c,0x36,0x02,0xe7,0xa0,0xc4,0xc8,0x9e,
    0xea,0xbf,0x8a,0xd2,0x40,0xc7,0x38,0xb5,0xa3,0xf7,0xf2,0xce,0xf9,0x61,0x15,0xa1,
    0xe0,0xae,0x5d,0xa4,0x9b,0x34,0x1a,0x55,0xad,0x93,0x32,0x30,0xf5,0x8c,0xb1,0xe3,
    0x1d,0xf6,0xe2,0x2e,0x82,0x66,0xca,0x60,0xc0,0x29,0x23,0xab,0x0d,0x53,0x4e,0x6f,
    0xd5,0xdb,0x37,0x45,0xde,0xfd,0x8e,0x2f,0x03,0xff,0x6a,0x72,0x6d,0x6c,0x5b,0x51,
    0x8d,0x1b,0xaf,0x92,0xbb,0xdd,0xbc,0x7f,0x11,0xd9,0x5c,0x41,0x1f,0x10,0x5a,0xd8,
    0x0a,0xc1,0x31,0x88,0xa5,0xcd,0x7b,0xbd,0x2d,0x74,0xd0,0x12,0xb8,0xe5,0xb4,0xb0,
    0x89,0x69,0x97,0x4a,0x0c,0x96,0x77,0x7e,0x65,0xb9,0xf1,0x09,0xc5,0x6e,0xc6,0x84,
    0x18,0xf0,0x7d,0xec,0x3a,0xdc,0x4d,0x20,0x79,0xee,0x5f,0x3e,0xd7,0xcb,0x39,0x48
};

/* 系统参数 FK */
uint32_t FK[4] = {
    0xa3b1bac6, 0x56aa3350, 0x677d9197, 0xb27022dc
};

/* 固定参数 CK（32个） */
uint32_t CK[32] = {
    0x00070e15,0x1c232a31,0x383f464d,0x545b6269,
    0x70777e85,0x8c939aa1,0xa8afb6bd,0xc4cbd2d9,
    0xe0e7eef5,0xfc030a11,0x181f262d,0x343b4249,
    0x50575e65,0x6c737a81,0x888f969d,0xa4abb2b9,
    0xc0c7ced5,0xdce3eaf1,0xf8ff060d,0x141b2229,
    0x30373e45,0x4c535a61,0x686f767d,0x848b9299,
    0xa0a7aeb5,0xbcc3cad1,0xd8dfe6ed,0xf4fb0209,
    0x10171e25,0x2c333a41,0x484f565d,0x646b7279
};

//32位循环左移
uint32_t rol32(uint32_t value, uint32_t shift)
{
    return (value << shift) | (value >> (32 - shift));
}

//32位循环右移
uint32_t ror32(uint32_t value, uint32_t shift)
{
    return (value >> shift) | (value << (32 - shift));
}

// 将4字节数组转为uint32_t整数（大端序)
uint32_t BytesToInt(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) | ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) | ((uint32_t)bytes[3]);
}

// 将uint32_t整数写入4字节数组（大端序）
void IntToBytes(uint8_t* bytes, uint32_t data)
{
    bytes[0] = (uint8_t)(data >> 24);
    bytes[1] = (uint8_t)(data >> 16);
    bytes[2] = (uint8_t)(data >> 8);
    bytes[3] = (uint8_t)(data);
}

// S盒替换：输入32位，输出32位
uint32_t SboxReplace(uint32_t input)
{
    uint32_t output = 0;
    output |= (uint32_t)Sbox[(input >> 24) & 0xff] << 24;
    output |= (uint32_t)Sbox[(input >> 16) & 0xFF] << 16;
    output |= (uint32_t)Sbox[(input >> 8) & 0xFF] << 8;
    output |= (uint32_t)Sbox[input & 0xFF];
    return output;
}

// 合成置换T
uint32_t TReplace(uint32_t input)
{
    uint32_t output = SboxReplace(input);
    return output ^ rol32(output, 2) ^ rol32(output, 10) ^
        rol32(output, 18) ^ rol32(output, 24);
}

// 密钥扩展合成置换T'
uint32_t KeyTReplace(uint32_t input)
{
    uint32_t output = SboxReplace(input);
    return output ^ rol32(output, 13) ^ rol32(output, 23);
}

// 密钥key扩展后生成32轮密钥rk
void KeyExpansion(const uint8_t* key, uint32_t* rk)
{
    uint32_t K[36]; // K[0] ~ K[3]为初始状态, K[4] ~ K[35] 为中间状态

    // 1. 将 16 字节密钥转换为 4 个 32 位字（大端序）
    K[0] = BytesToInt((uint8_t*)&key[0]);
    K[1] = BytesToInt((uint8_t*)&key[4]);
    K[2] = BytesToInt((uint8_t*)&key[8]);
    K[3] = BytesToInt((uint8_t*)&key[12]);

    /* 2. 异或系统参数 FK */
    for (int i = 0; i < 4; i++)
    {
        K[i] ^= FK[i];
    }

    /* 3. 迭代生成轮密钥 */
    for (int i = 0; i < 32; i++)
    {
        uint32_t temp = K[i + 1] ^ K[i + 2] ^ K[i + 3] ^ CK[i];
        K[i + 4] = K[i] ^ KeyTReplace(temp);
        rk[i] = K[i + 4];
    }

}

// PKCS#7 填充 如果待加密数据不是16字节的整数倍，就需要在加密前进行填充，使其达到16字节的倍数
int pkcs7_padding(uint8_t* data, int data_len, int block_size)
{
    // SM4 block_size 固定为16
    // 1. 计算需要填充的字节数 
    uint8_t pad_len = block_size - (data_len % block_size);

    // 2. 在原始数据末尾追加 pad_len 个字节，每个字节的值都是 pad_len
    for (int i = 0; i < pad_len; i++)
    {
        data[data_len + i] = pad_len;
    }

    // 3. 返回填充后的新数据长度
    return data_len + pad_len;
}

void sm4_Encrypt(uint8_t* input, uint32_t* rk, uint8_t* encryptData)
{
    uint32_t x[36] = { 0 };

    /* 将128位输入分为4个32位字 */
    x[0] = BytesToInt((uint8_t*)&input[0]);
    x[1] = BytesToInt((uint8_t*)&input[4]);
    x[2] = BytesToInt((uint8_t*)&input[8]);
    x[3] = BytesToInt((uint8_t*)&input[12]);

    /* 32轮迭代 */
    for (int i = 0; i < 32; i++)
    {
        x[i + 4] = x[i] ^ TReplace(x[i + 1] ^ x[i + 2] ^ x[i + 3] ^ rk[i]);
    }

    // 反序输出,并将4字节转化为1字节存储在encryptData中
    IntToBytes((uint8_t*)&encryptData[0], x[35]);
    IntToBytes((uint8_t*)&encryptData[4], x[34]);
    IntToBytes((uint8_t*)&encryptData[8], x[33]);
    IntToBytes((uint8_t*)&encryptData[12], x[32]);
}

void ECBEncrypt(uint8_t* inputBuffer, uint8_t* key, int input_len, uint8_t* encryptData)
{
    uint32_t rk[32] = { 0 };
    KeyExpansion(key, rk);// 密钥扩展

    int total_len = pkcs7_padding(inputBuffer, input_len, 16);// pkcs7填充明文

    int blocks = total_len / 16; // 计算一共有多少组

    // 每次加密一组(16字节)，并把加密数据存放再encryptData中
    for (int i = 0; i < blocks; i++)
    {
        sm4_Encrypt((uint8_t*)&inputBuffer[i * 16], rk, (uint8_t*)&encryptData[i * 16]);
    }


}

void ECBdecrypt(uint8_t* ciphertext, uint8_t* key, int cipher_len, uint8_t* plaintext)
{
    // 逆序轮密钥
    uint32_t inv_rk[32] = { 0 };
    KeyExpansion(key, inv_rk);
    uint32_t temp = 0;
    for (int i = 0; i < 16; i++)
    {
        temp = inv_rk[i];
        inv_rk[i] = inv_rk[31 - i];
        inv_rk[31 - i] = temp;
    }

    // 后续解密操作和加密相同
    int blocks = cipher_len / 16;
    for (int i = 0; i < blocks; i++)
    {
        sm4_Encrypt((uint8_t*)&ciphertext[i * 16], inv_rk, (uint8_t*)&plaintext[i * 16]);
    }

}

int main()
{
    // 固定16字节密钥
    uint8_t key[16] = { 98,108,97,99,107,109,97,110,98,97,54,54,54,54,54,54 };
    uint8_t plaintext[0xff] = { 0 }; // 存储明文
    uint8_t ciphertext[] = { 141,29,120,62,65,51,144,223,11,4,251,202,83,153,238,37,120,2,152,45,39,85,6,113,138,137,75,173,110,8,215,43 };// 密文

    int cipher_len = sizeof(ciphertext) / sizeof(ciphertext[0]);

    ECBdecrypt(ciphertext, key, cipher_len, plaintext);

    printf("%s\r\n", plaintext);

    return 0;
}
```


## python实现

```python
MASK_32 = 0xffffffff # 32位全一掩码

  

# S盒

Sbox = [

    0xd6,0x90,0xe9,0xfe,0xcc,0xe1,0x3d,0xb7,0x16,0xb6,0x14,0xc2,0x28,0xfb,0x2c,0x05,

    0x2b,0x67,0x9a,0x76,0x2a,0xbe,0x04,0xc3,0xaa,0x44,0x13,0x26,0x49,0x86,0x06,0x99,

    0x9c,0x42,0x50,0xf4,0x91,0xef,0x98,0x7a,0x33,0x54,0x0b,0x43,0xed,0xcf,0xac,0x62,

    0xe4,0xb3,0x1c,0xa9,0xc9,0x08,0xe8,0x95,0x80,0xdf,0x94,0xfa,0x75,0x8f,0x3f,0xa6,

    0x47,0x07,0xa7,0xfc,0xf3,0x73,0x17,0xba,0x83,0x59,0x3c,0x19,0xe6,0x85,0x4f,0xa8,

    0x68,0x6b,0x81,0xb2,0x71,0x64,0xda,0x8b,0xf8,0xeb,0x0f,0x4b,0x70,0x56,0x9d,0x35,

    0x1e,0x24,0x0e,0x5e,0x63,0x58,0xd1,0xa2,0x25,0x22,0x7c,0x3b,0x01,0x21,0x78,0x87,

    0xd4,0x00,0x46,0x57,0x9f,0xd3,0x27,0x52,0x4c,0x36,0x02,0xe7,0xa0,0xc4,0xc8,0x9e,

    0xea,0xbf,0x8a,0xd2,0x40,0xc7,0x38,0xb5,0xa3,0xf7,0xf2,0xce,0xf9,0x61,0x15,0xa1,

    0xe0,0xae,0x5d,0xa4,0x9b,0x34,0x1a,0x55,0xad,0x93,0x32,0x30,0xf5,0x8c,0xb1,0xe3,

    0x1d,0xf6,0xe2,0x2e,0x82,0x66,0xca,0x60,0xc0,0x29,0x23,0xab,0x0d,0x53,0x4e,0x6f,

    0xd5,0xdb,0x37,0x45,0xde,0xfd,0x8e,0x2f,0x03,0xff,0x6a,0x72,0x6d,0x6c,0x5b,0x51,

    0x8d,0x1b,0xaf,0x92,0xbb,0xdd,0xbc,0x7f,0x11,0xd9,0x5c,0x41,0x1f,0x10,0x5a,0xd8,

    0x0a,0xc1,0x31,0x88,0xa5,0xcd,0x7b,0xbd,0x2d,0x74,0xd0,0x12,0xb8,0xe5,0xb4,0xb0,

    0x89,0x69,0x97,0x4a,0x0c,0x96,0x77,0x7e,0x65,0xb9,0xf1,0x09,0xc5,0x6e,0xc6,0x84,

    0x18,0xf0,0x7d,0xec,0x3a,0xdc,0x4d,0x20,0x79,0xee,0x5f,0x3e,0xd7,0xcb,0x39,0x48

]

  

#/* 系统参数 FK */

FK = [

    0xa3b1bac6, 0x56aa3350, 0x677d9197, 0xb27022dc

]

  

#/* 固定参数 CK（32个） */

CK = [

    0x00070e15,0x1c232a31,0x383f464d,0x545b6269,

    0x70777e85,0x8c939aa1,0xa8afb6bd,0xc4cbd2d9,

    0xe0e7eef5,0xfc030a11,0x181f262d,0x343b4249,

    0x50575e65,0x6c737a81,0x888f969d,0xa4abb2b9,

    0xc0c7ced5,0xdce3eaf1,0xf8ff060d,0x141b2229,

    0x30373e45,0x4c535a61,0x686f767d,0x848b9299,

    0xa0a7aeb5,0xbcc3cad1,0xd8dfe6ed,0xf4fb0209,

    0x10171e25,0x2c333a41,0x484f565d,0x646b7279

]

  

# 32位循环左移

def rol32(value, n):

    value = value & MASK_32

    return ((value << n) | (value >> (32 - n))) & MASK_32

  

# 32位循环右移

def ror32(value, n):

    value = value & MASK_32

    return ((value >> n) | (value << (32 - n))) & MASK_32

  

#将 4 字节大端序字节串转换为 32 位无符号整数。

def bytes_to_uint32(data):

    return int.from_bytes(data, byteorder='big')

  

#将 32 位无符号整数转换为 4 字节大端序字节串。

def uint32_to_bytes(data):

    return data.to_bytes(4, byteorder='big')

  

#S盒替换

def sbox_replace(x):

    return (

        (Sbox[(x >> 24) & 0xff] << 24) |

        (Sbox[(x >> 16) & 0xff] << 16) |

        (Sbox[(x >> 8) & 0xff] << 8) |

        Sbox[x & 0xff]

    )

  

# 加密轮函数合成置换T

def T_replace(x):

    b = sbox_replace(x)

    return b ^ rol32(b, 2) ^ rol32(b, 10) ^ rol32(b, 18) ^ rol32(b, 24)

  

# 密钥扩展合成置换T

def keyT_replace(x):

    b = sbox_replace(x)

    return b ^ rol32(b, 13) ^ rol32(b, 23)

  

# 密钥扩展

def key_expansion(key):

    #将16字节密钥拆成4个4字节整数

    k = []

    for i in range(4):

        k.append(bytes_to_uint32(key[i*4:i*4 + 4]))

  

    # 与FK异或

    for i in range(4):

        k[i] ^= FK[i]

  

    # 生成轮密钥rk

    rk = []

    for i in range(32):

        temp = k[i + 1] ^ k[i + 2] ^ k[i + 3] ^ CK[i]

        newkey = k[i] ^ keyT_replace(temp)

        k.append(newkey)

        rk.append(newkey)

    return rk

  

# 逆序轮密钥rk

def inv_key_generate(key):

    rk = key_expansion(key)

    inv_rk = []

  

    for i in range(31,-1,-1):

        inv_rk.append(rk[i])

  

    return inv_rk

  

def sm4_decrypt_ecb(ciphertext, inv_rk):

  

    # 密文初始化->将4字节转化为整数

    x = []

    for i in range(4):

        x.append(bytes_to_uint32(ciphertext[i*4:i*4+4]))

    # 32轮加密

    for i in range(32):

        x.append(x[i] ^ T_replace(x[i + 1] ^ x[i + 2] ^ x[i + 3] ^ inv_rk[i]))

  

    # 反序输出，并将4字节整数转化为字节存储在明文中

    plaintext = b''

    for i in range(35,31,-1):

        plaintext += uint32_to_bytes(x[i])

  

    return plaintext

  
  

def sm4_decrypt(ciphertext,key):

  

    if (len(key) != 16):

        print("密钥长度需满足16字节")

        return False

  

    ciphertext_len = len(ciphertext)

    if (ciphertext_len % 16 != 0):

        print("理论上密文需满足16字节整数倍")

        return False

  

    inv_rk = inv_key_generate(key) #生成轮密钥

  

    blocks_size = ciphertext_len // 16   #计算分组个数

    plaintext = b''  #存储明文

    for i in range(blocks_size):

        plaintext += sm4_decrypt_ecb(ciphertext[i*16:i*16+16], inv_rk)

  

    print(plaintext)

    return

  
  

def main():

  

    key = bytes([98,108,97,99,107,109,97,110,98,97,54,54,54,54,54,54])

    ciphertext = bytes([141,29,120,62,65,51,144,223,11,4,251,202,83,153,238,37,120,2,152,45,39,85,6,113,138,137,75,173,110,8,215,43 ])

    sm4_decrypt(ciphertext, key)

  

    return

  
  

if __name__ == "__main__":

    main()
```


# cbc加密模式

跟ecb差别在于 需要向量IV异或第一个明文块，且后续明文块加密需先异或前一个密文块
```python
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

uint8_t Sbox[256] = {
    0xd6,0x90,0xe9,0xfe,0xcc,0xe1,0x3d,0xb7,0x16,0xb6,0x14,0xc2,0x28,0xfb,0x2c,0x05,
    0x2b,0x67,0x9a,0x76,0x2a,0xbe,0x04,0xc3,0xaa,0x44,0x13,0x26,0x49,0x86,0x06,0x99,
    0x9c,0x42,0x50,0xf4,0x91,0xef,0x98,0x7a,0x33,0x54,0x0b,0x43,0xed,0xcf,0xac,0x62,
    0xe4,0xb3,0x1c,0xa9,0xc9,0x08,0xe8,0x95,0x80,0xdf,0x94,0xfa,0x75,0x8f,0x3f,0xa6,
    0x47,0x07,0xa7,0xfc,0xf3,0x73,0x17,0xba,0x83,0x59,0x3c,0x19,0xe6,0x85,0x4f,0xa8,
    0x68,0x6b,0x81,0xb2,0x71,0x64,0xda,0x8b,0xf8,0xeb,0x0f,0x4b,0x70,0x56,0x9d,0x35,
    0x1e,0x24,0x0e,0x5e,0x63,0x58,0xd1,0xa2,0x25,0x22,0x7c,0x3b,0x01,0x21,0x78,0x87,
    0xd4,0x00,0x46,0x57,0x9f,0xd3,0x27,0x52,0x4c,0x36,0x02,0xe7,0xa0,0xc4,0xc8,0x9e,
    0xea,0xbf,0x8a,0xd2,0x40,0xc7,0x38,0xb5,0xa3,0xf7,0xf2,0xce,0xf9,0x61,0x15,0xa1,
    0xe0,0xae,0x5d,0xa4,0x9b,0x34,0x1a,0x55,0xad,0x93,0x32,0x30,0xf5,0x8c,0xb1,0xe3,
    0x1d,0xf6,0xe2,0x2e,0x82,0x66,0xca,0x60,0xc0,0x29,0x23,0xab,0x0d,0x53,0x4e,0x6f,
    0xd5,0xdb,0x37,0x45,0xde,0xfd,0x8e,0x2f,0x03,0xff,0x6a,0x72,0x6d,0x6c,0x5b,0x51,
    0x8d,0x1b,0xaf,0x92,0xbb,0xdd,0xbc,0x7f,0x11,0xd9,0x5c,0x41,0x1f,0x10,0x5a,0xd8,
    0x0a,0xc1,0x31,0x88,0xa5,0xcd,0x7b,0xbd,0x2d,0x74,0xd0,0x12,0xb8,0xe5,0xb4,0xb0,
    0x89,0x69,0x97,0x4a,0x0c,0x96,0x77,0x7e,0x65,0xb9,0xf1,0x09,0xc5,0x6e,0xc6,0x84,
    0x18,0xf0,0x7d,0xec,0x3a,0xdc,0x4d,0x20,0x79,0xee,0x5f,0x3e,0xd7,0xcb,0x39,0x48
};

/* 系统参数 FK */
uint32_t FK[4] = {
    0xa3b1bac6, 0x56aa3350, 0x677d9197, 0xb27022dc
};

/* 固定参数 CK（32个） */
uint32_t CK[32] = {
    0x00070e15,0x1c232a31,0x383f464d,0x545b6269,
    0x70777e85,0x8c939aa1,0xa8afb6bd,0xc4cbd2d9,
    0xe0e7eef5,0xfc030a11,0x181f262d,0x343b4249,
    0x50575e65,0x6c737a81,0x888f969d,0xa4abb2b9,
    0xc0c7ced5,0xdce3eaf1,0xf8ff060d,0x141b2229,
    0x30373e45,0x4c535a61,0x686f767d,0x848b9299,
    0xa0a7aeb5,0xbcc3cad1,0xd8dfe6ed,0xf4fb0209,
    0x10171e25,0x2c333a41,0x484f565d,0x646b7279
};

//32位循环左移
uint32_t rol32(uint32_t value, uint32_t shift)
{
    return (value << shift) | (value >> (32 - shift));
}

//32位循环右移
uint32_t ror32(uint32_t value, uint32_t shift)
{
    return (value >> shift) | (value << (32 - shift));
}

// 将4字节数组转为uint32_t整数（大端序)
uint32_t BytesToInt(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) | ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) | ((uint32_t)bytes[3]);
}

// 将uint32_t整数写入4字节数组（大端序）
void IntToBytes(uint8_t* bytes, uint32_t data)
{
    bytes[0] = (uint8_t)(data >> 24);
    bytes[1] = (uint8_t)(data >> 16);
    bytes[2] = (uint8_t)(data >> 8);
    bytes[3] = (uint8_t)(data);
}

// S盒替换：输入32位，输出32位
uint32_t SboxReplace(uint32_t input)
{
    uint32_t output = 0;
    output |= (uint32_t)Sbox[(input >> 24) & 0xff] << 24;
    output |= (uint32_t)Sbox[(input >> 16) & 0xFF] << 16;
    output |= (uint32_t)Sbox[(input >> 8) & 0xFF] << 8;
    output |= (uint32_t)Sbox[input & 0xFF];
    return output;
}

// 合成置换T
uint32_t TReplace(uint32_t input)
{
    uint32_t output = SboxReplace(input);
    return output ^ rol32(output, 2) ^ rol32(output, 10) ^
        rol32(output, 18) ^ rol32(output, 24);
}

// 密钥扩展合成置换T'
uint32_t KeyTReplace(uint32_t input)
{
    uint32_t output = SboxReplace(input);
    return output ^ rol32(output, 13) ^ rol32(output, 23);
}

// 密钥key扩展后生成32轮密钥rk
void KeyExpansion(const uint8_t* key, uint32_t* rk)
{
    uint32_t K[36]; // K[0] ~ K[3]为初始状态, K[4] ~ K[35] 为中间状态

    // 1. 将 16 字节密钥转换为 4 个 32 位字（大端序）
    K[0] = BytesToInt((uint8_t*)&key[0]);
    K[1] = BytesToInt((uint8_t*)&key[4]);
    K[2] = BytesToInt((uint8_t*)&key[8]);
    K[3] = BytesToInt((uint8_t*)&key[12]);

    /* 2. 异或系统参数 FK */
    for (int i = 0; i < 4; i++)
    {
        K[i] ^= FK[i];
    }

    /* 3. 迭代生成轮密钥 */
    for (int i = 0; i < 32; i++)
    {
        uint32_t temp = K[i + 1] ^ K[i + 2] ^ K[i + 3] ^ CK[i];
        K[i + 4] = K[i] ^ KeyTReplace(temp);
        rk[i] = K[i + 4];
    }

}

// PKCS#7 填充 如果待加密数据不是16字节的整数倍，就需要在加密前进行填充，使其达到16字节的倍数
int pkcs7_padding(uint8_t* data, int data_len, int block_size)
{
    // SM4 block_size 固定为16
    // 1. 计算需要填充的字节数 
    uint8_t pad_len = block_size - (data_len % block_size);

    // 2. 在原始数据末尾追加 pad_len 个字节，每个字节的值都是 pad_len
    for (int i = 0; i < pad_len; i++)
    {
        data[data_len + i] = pad_len;
    }

    // 3. 返回填充后的新数据长度
    return data_len + pad_len;
}

void sm4_Encrypt(uint8_t* input, uint32_t* rk, uint8_t* encryptData)
{
    uint32_t x[36] = { 0 };

    /* 将128位输入分为4个32位字 */
    x[0] = BytesToInt((uint8_t*)&input[0]);
    x[1] = BytesToInt((uint8_t*)&input[4]);
    x[2] = BytesToInt((uint8_t*)&input[8]);
    x[3] = BytesToInt((uint8_t*)&input[12]);

    /* 32轮迭代 */
    for (int i = 0; i < 32; i++)
    {
        x[i + 4] = x[i] ^ TReplace(x[i + 1] ^ x[i + 2] ^ x[i + 3] ^ rk[i]);
    }

    // 反序输出,并将4字节转化为1字节存储在encryptData中
    IntToBytes((uint8_t*)&encryptData[0], x[35]);
    IntToBytes((uint8_t*)&encryptData[4], x[34]);
    IntToBytes((uint8_t*)&encryptData[8], x[33]);
    IntToBytes((uint8_t*)&encryptData[12], x[32]);
}

void CBCEncrypt(uint8_t* inputBuffer, uint8_t* key, int input_len, uint8_t* encryptData)
{
    uint8_t IV[16] = { 84,104,117,114,115,100,97,121,86,73,118,111,53,48,106,106 };  //初始化向量IV
    uint32_t rk[32] = { 0 };
    KeyExpansion(key, rk);// 密钥扩展

    int total_len = pkcs7_padding(inputBuffer, input_len, 16);// pkcs7填充明文

    int blocks = total_len / 16; // 计算一共有多少组

    // 将第一个明文块与初始化向量IV进行XOR操作。
    for (int i = 0; i < 16; i++)
    {
        inputBuffer[i] ^= IV[i];
    }

    // 每次加密一组(16字节)，加密前异或前一个密文块,然后把加密数据存放再encryptData中
    for (int i = 0; i < blocks; i++)
    {
        if (i != 0)
        {
            for (int j = 0; j < 16; j++)
            {
                inputBuffer[i * 16 + j] ^= encryptData[(i - 1) * 16 + j];
            }
        }
        sm4_Encrypt((uint8_t*)&inputBuffer[i * 16], rk, (uint8_t*)&encryptData[i * 16]);
    }


}

int main()
{
    // 固定16字节密钥
    uint8_t key[16] = { 98,108,97,99,107,109,97,110,98,97,54,54,54,54,54,54 };
    uint8_t encryptData[0xff] = { 0 }; // 存放加密后的数据
    uint8_t ciphertext[] = { 236,118,25,142,226,242,191,197,234,198,102,212,184,33,202,223,134,232,32,180,116,155,178,105,144,106,144,249,5,67,34,14 };// 密文

    uint8_t inputBuffer[0xff] = { 0 };
    printf("Please input your flag: ");
    scanf("%s", inputBuffer);

    int input_len = strlen((char*)inputBuffer);
    if (input_len != 28)
    {
        printf("wrong length\r\n");
        system("pause");
        return 0;
    }

    CBCEncrypt(inputBuffer, key, input_len, encryptData);

    for (int i = 0; i < input_len; i++)
    {
        if (encryptData[i] != ciphertext[i])
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

# cbc解密

## c语言实现
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

uint8_t Sbox[256] = {
    0xd6,0x90,0xe9,0xfe,0xcc,0xe1,0x3d,0xb7,0x16,0xb6,0x14,0xc2,0x28,0xfb,0x2c,0x05,
    0x2b,0x67,0x9a,0x76,0x2a,0xbe,0x04,0xc3,0xaa,0x44,0x13,0x26,0x49,0x86,0x06,0x99,
    0x9c,0x42,0x50,0xf4,0x91,0xef,0x98,0x7a,0x33,0x54,0x0b,0x43,0xed,0xcf,0xac,0x62,
    0xe4,0xb3,0x1c,0xa9,0xc9,0x08,0xe8,0x95,0x80,0xdf,0x94,0xfa,0x75,0x8f,0x3f,0xa6,
    0x47,0x07,0xa7,0xfc,0xf3,0x73,0x17,0xba,0x83,0x59,0x3c,0x19,0xe6,0x85,0x4f,0xa8,
    0x68,0x6b,0x81,0xb2,0x71,0x64,0xda,0x8b,0xf8,0xeb,0x0f,0x4b,0x70,0x56,0x9d,0x35,
    0x1e,0x24,0x0e,0x5e,0x63,0x58,0xd1,0xa2,0x25,0x22,0x7c,0x3b,0x01,0x21,0x78,0x87,
    0xd4,0x00,0x46,0x57,0x9f,0xd3,0x27,0x52,0x4c,0x36,0x02,0xe7,0xa0,0xc4,0xc8,0x9e,
    0xea,0xbf,0x8a,0xd2,0x40,0xc7,0x38,0xb5,0xa3,0xf7,0xf2,0xce,0xf9,0x61,0x15,0xa1,
    0xe0,0xae,0x5d,0xa4,0x9b,0x34,0x1a,0x55,0xad,0x93,0x32,0x30,0xf5,0x8c,0xb1,0xe3,
    0x1d,0xf6,0xe2,0x2e,0x82,0x66,0xca,0x60,0xc0,0x29,0x23,0xab,0x0d,0x53,0x4e,0x6f,
    0xd5,0xdb,0x37,0x45,0xde,0xfd,0x8e,0x2f,0x03,0xff,0x6a,0x72,0x6d,0x6c,0x5b,0x51,
    0x8d,0x1b,0xaf,0x92,0xbb,0xdd,0xbc,0x7f,0x11,0xd9,0x5c,0x41,0x1f,0x10,0x5a,0xd8,
    0x0a,0xc1,0x31,0x88,0xa5,0xcd,0x7b,0xbd,0x2d,0x74,0xd0,0x12,0xb8,0xe5,0xb4,0xb0,
    0x89,0x69,0x97,0x4a,0x0c,0x96,0x77,0x7e,0x65,0xb9,0xf1,0x09,0xc5,0x6e,0xc6,0x84,
    0x18,0xf0,0x7d,0xec,0x3a,0xdc,0x4d,0x20,0x79,0xee,0x5f,0x3e,0xd7,0xcb,0x39,0x48
};

/* 系统参数 FK */
uint32_t FK[4] = {
    0xa3b1bac6, 0x56aa3350, 0x677d9197, 0xb27022dc
};

/* 固定参数 CK（32个） */
uint32_t CK[32] = {
    0x00070e15,0x1c232a31,0x383f464d,0x545b6269,
    0x70777e85,0x8c939aa1,0xa8afb6bd,0xc4cbd2d9,
    0xe0e7eef5,0xfc030a11,0x181f262d,0x343b4249,
    0x50575e65,0x6c737a81,0x888f969d,0xa4abb2b9,
    0xc0c7ced5,0xdce3eaf1,0xf8ff060d,0x141b2229,
    0x30373e45,0x4c535a61,0x686f767d,0x848b9299,
    0xa0a7aeb5,0xbcc3cad1,0xd8dfe6ed,0xf4fb0209,
    0x10171e25,0x2c333a41,0x484f565d,0x646b7279
};

//32位循环左移
uint32_t rol32(uint32_t value, uint32_t shift)
{
    return (value << shift) | (value >> (32 - shift));
}

//32位循环右移
uint32_t ror32(uint32_t value, uint32_t shift)
{
    return (value >> shift) | (value << (32 - shift));
}

// 将4字节数组转为uint32_t整数（大端序)
uint32_t BytesToInt(uint8_t* bytes)
{
    return ((uint32_t)bytes[0] << 24) | ((uint32_t)bytes[1] << 16) |
        ((uint32_t)bytes[2] << 8) | ((uint32_t)bytes[3]);
}

// 将uint32_t整数写入4字节数组（大端序）
void IntToBytes(uint8_t* bytes, uint32_t data)
{
    bytes[0] = (uint8_t)(data >> 24);
    bytes[1] = (uint8_t)(data >> 16);
    bytes[2] = (uint8_t)(data >> 8);
    bytes[3] = (uint8_t)(data);
}

// S盒替换：输入32位，输出32位
uint32_t SboxReplace(uint32_t input)
{
    uint32_t output = 0;
    output |= (uint32_t)Sbox[(input >> 24) & 0xff] << 24;
    output |= (uint32_t)Sbox[(input >> 16) & 0xFF] << 16;
    output |= (uint32_t)Sbox[(input >> 8) & 0xFF] << 8;
    output |= (uint32_t)Sbox[input & 0xFF];
    return output;
}

// 合成置换T
uint32_t TReplace(uint32_t input)
{
    uint32_t output = SboxReplace(input);
    return output ^ rol32(output, 2) ^ rol32(output, 10) ^
        rol32(output, 18) ^ rol32(output, 24);
}

// 密钥扩展合成置换T'
uint32_t KeyTReplace(uint32_t input)
{
    uint32_t output = SboxReplace(input);
    return output ^ rol32(output, 13) ^ rol32(output, 23);
}

// 密钥key扩展后生成32轮密钥rk
void KeyExpansion(const uint8_t* key, uint32_t* rk)
{
    uint32_t K[36]; // K[0] ~ K[3]为初始状态, K[4] ~ K[35] 为中间状态

    // 1. 将 16 字节密钥转换为 4 个 32 位字（大端序）
    K[0] = BytesToInt((uint8_t*)&key[0]);
    K[1] = BytesToInt((uint8_t*)&key[4]);
    K[2] = BytesToInt((uint8_t*)&key[8]);
    K[3] = BytesToInt((uint8_t*)&key[12]);

    /* 2. 异或系统参数 FK */
    for (int i = 0; i < 4; i++)
    {
        K[i] ^= FK[i];
    }

    /* 3. 迭代生成轮密钥 */
    for (int i = 0; i < 32; i++)
    {
        uint32_t temp = K[i + 1] ^ K[i + 2] ^ K[i + 3] ^ CK[i];
        K[i + 4] = K[i] ^ KeyTReplace(temp);
        rk[i] = K[i + 4];
    }

}

// PKCS#7 填充 如果待加密数据不是16字节的整数倍，就需要在加密前进行填充，使其达到16字节的倍数
int pkcs7_padding(uint8_t* data, int data_len, int block_size)
{
    // SM4 block_size 固定为16
    // 1. 计算需要填充的字节数 
    uint8_t pad_len = block_size - (data_len % block_size);

    // 2. 在原始数据末尾追加 pad_len 个字节，每个字节的值都是 pad_len
    for (int i = 0; i < pad_len; i++)
    {
        data[data_len + i] = pad_len;
    }

    // 3. 返回填充后的新数据长度
    return data_len + pad_len;
}

void sm4_Encrypt(uint8_t* input, uint32_t* rk, uint8_t* encryptData)
{
    uint32_t x[36] = { 0 };

    /* 将128位输入分为4个32位字 */
    x[0] = BytesToInt((uint8_t*)&input[0]);
    x[1] = BytesToInt((uint8_t*)&input[4]);
    x[2] = BytesToInt((uint8_t*)&input[8]);
    x[3] = BytesToInt((uint8_t*)&input[12]);

    /* 32轮迭代 */
    for (int i = 0; i < 32; i++)
    {
        x[i + 4] = x[i] ^ TReplace(x[i + 1] ^ x[i + 2] ^ x[i + 3] ^ rk[i]);
    }

    // 反序输出,并将4字节转化为1字节存储在encryptData中
    IntToBytes((uint8_t*)&encryptData[0], x[35]);
    IntToBytes((uint8_t*)&encryptData[4], x[34]);
    IntToBytes((uint8_t*)&encryptData[8], x[33]);
    IntToBytes((uint8_t*)&encryptData[12], x[32]);
}

void ECBEncrypt(uint8_t* inputBuffer, uint8_t* key, int input_len, uint8_t* encryptData)
{
    uint32_t rk[32] = { 0 };
    KeyExpansion(key, rk);// 密钥扩展

    int total_len = pkcs7_padding(inputBuffer, input_len, 16);// pkcs7填充明文

    int blocks = total_len / 16; // 计算一共有多少组

    // 每次加密一组(16字节)，并把加密数据存放再encryptData中
    for (int i = 0; i < blocks; i++)
    {
        sm4_Encrypt((uint8_t*)&inputBuffer[i * 16], rk, (uint8_t*)&encryptData[i * 16]);
    }


}

void CBCEncrypt(uint8_t* inputBuffer, uint8_t* key, int input_len, uint8_t* encryptData)
{
    uint8_t IV[16] = { 84,104,117,114,115,100,97,121,86,73,118,111,53,48,106,106 };  //初始化向量IV
    uint32_t rk[32] = { 0 };
    KeyExpansion(key, rk);// 密钥扩展

    int total_len = pkcs7_padding(inputBuffer, input_len, 16);// pkcs7填充明文

    int blocks = total_len / 16; // 计算一共有多少组

    // 将第一个明文块与初始化向量IV进行XOR操作。
    for (int i = 0; i < 16; i++)
    {
        inputBuffer[i] ^= IV[i];
    }

    // 每次加密一组(16字节)，加密前异或前一个密文块,然后把加密数据存放再encryptData中
    for (int i = 0; i < blocks; i++)
    {
        if (i != 0)
        {
            for (int j = 0; j < 16; j++)
            {
                inputBuffer[i * 16 + j] ^= encryptData[(i - 1) * 16 + j];
            }
        }
        sm4_Encrypt((uint8_t*)&inputBuffer[i * 16], rk, (uint8_t*)&encryptData[i * 16]);
    }


}

void ECBdecrypt(uint8_t* ciphertext, uint8_t* key, int cipher_len, uint8_t* plaintext)
{
    // 逆序轮密钥
    uint32_t inv_rk[32] = { 0 };
    KeyExpansion(key, inv_rk);
    uint32_t temp = 0;
    for (int i = 0; i < 16; i++)
    {
        temp = inv_rk[i];
        inv_rk[i] = inv_rk[31 - i];
        inv_rk[31 - i] = temp;
    }

    // 后续解密操作和加密相同
    int blocks = cipher_len / 16;
    for (int i = 0; i < blocks; i++)
    {
        sm4_Encrypt((uint8_t*)&ciphertext[i * 16], inv_rk, (uint8_t*)&plaintext[i * 16]);
    }

}

void CBCdecrypt(uint8_t* ciphertext, uint8_t* key, int cipher_len, uint8_t* plaintext, uint8_t* IV)
{
    // 逆序轮密钥
    uint32_t inv_rk[32] = { 0 };
    KeyExpansion(key, inv_rk);
    uint32_t temp = 0;
    for (int i = 0; i < 16; i++)
    {
        temp = inv_rk[i];
        inv_rk[i] = inv_rk[31 - i];
        inv_rk[31 - i] = temp;
    }

    int blocks = cipher_len / 16;

    // 从最后的密文块开始往前解密
    for (int i = blocks - 1; i >= 0; i--)
    {
        sm4_Encrypt((uint8_t*)&ciphertext[i * 16], inv_rk, (uint8_t*)&plaintext[i * 16]);

        // i == 0 时异或IV, i != 0时异或前一个密文块
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
    // 固定16字节密钥
    uint8_t key[16] = { 98,108,97,99,107,109,97,110,98,97,54,54,54,54,54,54 };
    uint8_t plaintext[0xff] = { 0 }; // 存储明文
    uint8_t ciphertext[] = { 236,118,25,142,226,242,191,197,234,198,102,212,184,33,202,223,134,232,32,180,116,155,178,105,144,106,144,249,5,67,34,14 };// 密文
    uint8_t IV[16] = { 84,104,117,114,115,100,97,121,86,73,118,111,53,48,106,106 };

    int cipher_len = sizeof(ciphertext) / sizeof(ciphertext[0]);

    //ECBdecrypt(ciphertext, key, cipher_len, plaintext);

    CBCdecrypt(ciphertext, key, cipher_len, plaintext, IV);

    printf("%s\r\n", plaintext);

    return 0;
}
```


## python实现

```python
MASK_32 = 0xffffffff # 32位全一掩码

  

# S盒

Sbox = [

    0xd6,0x90,0xe9,0xfe,0xcc,0xe1,0x3d,0xb7,0x16,0xb6,0x14,0xc2,0x28,0xfb,0x2c,0x05,

    0x2b,0x67,0x9a,0x76,0x2a,0xbe,0x04,0xc3,0xaa,0x44,0x13,0x26,0x49,0x86,0x06,0x99,

    0x9c,0x42,0x50,0xf4,0x91,0xef,0x98,0x7a,0x33,0x54,0x0b,0x43,0xed,0xcf,0xac,0x62,

    0xe4,0xb3,0x1c,0xa9,0xc9,0x08,0xe8,0x95,0x80,0xdf,0x94,0xfa,0x75,0x8f,0x3f,0xa6,

    0x47,0x07,0xa7,0xfc,0xf3,0x73,0x17,0xba,0x83,0x59,0x3c,0x19,0xe6,0x85,0x4f,0xa8,

    0x68,0x6b,0x81,0xb2,0x71,0x64,0xda,0x8b,0xf8,0xeb,0x0f,0x4b,0x70,0x56,0x9d,0x35,

    0x1e,0x24,0x0e,0x5e,0x63,0x58,0xd1,0xa2,0x25,0x22,0x7c,0x3b,0x01,0x21,0x78,0x87,

    0xd4,0x00,0x46,0x57,0x9f,0xd3,0x27,0x52,0x4c,0x36,0x02,0xe7,0xa0,0xc4,0xc8,0x9e,

    0xea,0xbf,0x8a,0xd2,0x40,0xc7,0x38,0xb5,0xa3,0xf7,0xf2,0xce,0xf9,0x61,0x15,0xa1,

    0xe0,0xae,0x5d,0xa4,0x9b,0x34,0x1a,0x55,0xad,0x93,0x32,0x30,0xf5,0x8c,0xb1,0xe3,

    0x1d,0xf6,0xe2,0x2e,0x82,0x66,0xca,0x60,0xc0,0x29,0x23,0xab,0x0d,0x53,0x4e,0x6f,

    0xd5,0xdb,0x37,0x45,0xde,0xfd,0x8e,0x2f,0x03,0xff,0x6a,0x72,0x6d,0x6c,0x5b,0x51,

    0x8d,0x1b,0xaf,0x92,0xbb,0xdd,0xbc,0x7f,0x11,0xd9,0x5c,0x41,0x1f,0x10,0x5a,0xd8,

    0x0a,0xc1,0x31,0x88,0xa5,0xcd,0x7b,0xbd,0x2d,0x74,0xd0,0x12,0xb8,0xe5,0xb4,0xb0,

    0x89,0x69,0x97,0x4a,0x0c,0x96,0x77,0x7e,0x65,0xb9,0xf1,0x09,0xc5,0x6e,0xc6,0x84,

    0x18,0xf0,0x7d,0xec,0x3a,0xdc,0x4d,0x20,0x79,0xee,0x5f,0x3e,0xd7,0xcb,0x39,0x48

]

  

#/* 系统参数 FK */

FK = [

    0xa3b1bac6, 0x56aa3350, 0x677d9197, 0xb27022dc

]

  

#/* 固定参数 CK（32个） */

CK = [

    0x00070e15,0x1c232a31,0x383f464d,0x545b6269,

    0x70777e85,0x8c939aa1,0xa8afb6bd,0xc4cbd2d9,

    0xe0e7eef5,0xfc030a11,0x181f262d,0x343b4249,

    0x50575e65,0x6c737a81,0x888f969d,0xa4abb2b9,

    0xc0c7ced5,0xdce3eaf1,0xf8ff060d,0x141b2229,

    0x30373e45,0x4c535a61,0x686f767d,0x848b9299,

    0xa0a7aeb5,0xbcc3cad1,0xd8dfe6ed,0xf4fb0209,

    0x10171e25,0x2c333a41,0x484f565d,0x646b7279

]

  

# 32位循环左移

def rol32(value, n):

    value = value & MASK_32

    return ((value << n) | (value >> (32 - n))) & MASK_32

  

# 32位循环右移

def ror32(value, n):

    value = value & MASK_32

    return ((value >> n) | (value << (32 - n))) & MASK_32

  

#将 4 字节大端序字节串转换为 32 位无符号整数。

def bytes_to_uint32(data):

    return int.from_bytes(data, byteorder='big')

  

#将 32 位无符号整数转换为 4 字节大端序字节串。

def uint32_to_bytes(data):

    return data.to_bytes(4, byteorder='big')

  

#S盒替换

def sbox_replace(x):

    return (

        (Sbox[(x >> 24) & 0xff] << 24) |

        (Sbox[(x >> 16) & 0xff] << 16) |

        (Sbox[(x >> 8) & 0xff] << 8) |

        Sbox[x & 0xff]

    )

  

# 加密轮函数合成置换T

def T_replace(x):

    b = sbox_replace(x)

    return b ^ rol32(b, 2) ^ rol32(b, 10) ^ rol32(b, 18) ^ rol32(b, 24)

  

# 密钥扩展合成置换T

def keyT_replace(x):

    b = sbox_replace(x)

    return b ^ rol32(b, 13) ^ rol32(b, 23)

  

# 密钥扩展

def key_expansion(key):

    #将16字节密钥拆成4个4字节整数

    k = []

    for i in range(4):

        k.append(bytes_to_uint32(key[i*4:i*4 + 4]))

  

    # 与FK异或

    for i in range(4):

        k[i] ^= FK[i]

  

    # 生成轮密钥rk

    rk = []

    for i in range(32):

        temp = k[i + 1] ^ k[i + 2] ^ k[i + 3] ^ CK[i]

        newkey = k[i] ^ keyT_replace(temp)

        k.append(newkey)

        rk.append(newkey)

    return rk

  

# 逆序轮密钥rk

def inv_key_generate(key):

    rk = key_expansion(key)

    inv_rk = []

  

    for i in range(31,-1,-1):

        inv_rk.append(rk[i])

  

    return inv_rk

  

def sm4_decrypt(ciphertext, inv_rk):

  

    # 密文初始化->将4字节转化为整数

    x = []

    for i in range(4):

        x.append(bytes_to_uint32(ciphertext[i*4:i*4+4]))

    # 32轮加密

    for i in range(32):

        x.append(x[i] ^ T_replace(x[i + 1] ^ x[i + 2] ^ x[i + 3] ^ inv_rk[i]))

  

    # 反序输出，并将4字节整数转化为字节存储在明文中

    plaintext = b''

    for i in range(35,31,-1):

        plaintext += uint32_to_bytes(x[i])

  

    return plaintext

  
  

def sm4_ECBdecrypt(ciphertext,key):

  

    if (len(key) != 16):

        print("密钥长度需满足16字节")

        return False

  

    ciphertext_len = len(ciphertext)

    if (ciphertext_len % 16 != 0):

        print("理论上密文需满足16字节整数倍")

        return False

  

    inv_rk = inv_key_generate(key) #生成轮密钥

  

    blocks_size = ciphertext_len // 16   #计算分组个数

    plaintext = b''  #存储明文

    for i in range(blocks_size):

        plaintext += sm4_decrypt(ciphertext[i*16:i*16+16], inv_rk)

  

    print(plaintext)

    return

  
  

def sm4_CBCdecrypt(ciphertext, key, IV):

    if (len(IV) != 16):

        print("初始化向量IV需满足16字节")

        return False

  

    if (len(key) != 16):

        print("密钥长度需满足16字节")

        return False

  

    ciphertext_len = len(ciphertext)

    if (ciphertext_len % 16 != 0):

        print("理论上密文需满足16字节整数倍")

        return False

  

    inv_rk = inv_key_generate(key) #生成轮密钥

  

    blocks_size = ciphertext_len // 16   #计算分组个数

    plaintext = b'' #存储明文

  

    # cbc模式下如果为第一个明文块解密后需异或IV, 其他明文块需异或前一个密文快

    for i in range(blocks_size):

        temp = sm4_decrypt(ciphertext[i*16:i*16+16], inv_rk)

        # 异或IV

        if (i == 0):

            plaintext += bytes([temp[j] ^ IV[j] for j in range(16)])

        # 异或前一个密文块

        else:

            plaintext += bytes([temp[j] ^ ciphertext[(i -1) * 16 + j] for j in range(16)])

  

    print(plaintext)

    return

  
  

def main():

  

    key = bytes([98,108,97,99,107,109,97,110,98,97,54,54,54,54,54,54])

    ciphertext = bytes([236,118,25,142,226,242,191,197,234,198,102,212,184,33,202,223,134,232,32,180,116,155,178,105,144,106,144,249,5,67,34,14 ])

    IV = bytes([84,104,117,114,115,100,97,121,86,73,118,111,53,48,106,106])

    #sm4_ECBdecrypt(ciphertext, key)

    sm4_CBCdecrypt(ciphertext, key, IV)

  

    return

  
  

if __name__ == "__main__":

    main()
```