---
title: des
published: 2026-06-27
description: des加解密算法实现
image: ./cover.jpg
tags: [逆向]
category: Crypto
draft: false
---

# 加密脚本(含3des)
```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>

// =============== 置换表 ========================
// 初始置换IP
uint8_t IP[64] =
{
    58,50,42,34,26,18,10,2,
    60,52,44,36,28,20,12,4,
    62,54,46,38,30,22,14,6,
    64,56,48,40,32,24,16,8,
    57,49,41,33,25,17,9,1,
    59,51,43,35,27,19,11,3,
    61,53,45,37,29,21,13,5,
    63,55,47,39,31,23,15,7
};

// 逆初始置换ip_inv
uint8_t IP_INV[64] =
{
    40,8,48,16,56,24,64,32,
    39,7,47,15,55,23,63,31,
    38,6,46,14,54,22,62,30,
    37,5,45,13,53,21,61,29,
    36,4,44,12,52,20,60,28,
    35,3,43,11,51,19,59,27,
    34,2,42,10,50,18,58,26,
    33,1,41,9,49,17,57,25
};

// 扩展置换 E (32 - 48)
uint8_t E[48] =
{
    32,1,2,3,4,5,
    4,5,6,7,8,9,
    8,9,10,11,12,13,
    12,13,14,15,16,17,
    16,17,18,19,20,21,
    20,21,22,23,24,25,
    24,25,26,27,28,29,
    28,29,30,31,32,1
};

// P置换 (32 -> 32)
uint8_t P[32] =
{
    16,7,20,21,29,12,28,17,
    1,15,23,26,5,18,31,10,
    2,8,24,14,32,27,3,9,
    19,13,30,6,22,11,4,25
};

// 8个S盒 (每个 4x16)
uint8_t S[8][4][16] =
{
    { // S1
        {14,4,13,1,2,15,11,8,3,10,6,12,5,9,0,7},
        {0,15,7,4,14,2,13,1,10,6,12,11,9,5,3,8},
        {4,1,14,8,13,6,2,11,15,12,9,7,3,10,5,0},
        {15,12,8,2,4,9,1,7,5,11,3,14,10,0,6,13}
    },
    { // S2
        {15,1,8,14,6,11,3,4,9,7,2,13,12,0,5,10},
        {3,13,4,7,15,2,8,14,12,0,1,10,6,9,11,5},
        {0,14,7,11,10,4,13,1,5,8,12,6,9,3,2,15},
        {13,8,10,1,3,15,4,2,11,6,7,12,0,5,14,9}
    },
    { // S3
        {10,0,9,14,6,3,15,5,1,13,12,7,11,4,2,8},
        {13,7,0,9,3,4,6,10,2,8,5,14,12,11,15,1},
        {13,6,4,9,8,15,3,0,11,1,2,12,5,10,14,7},
        {1,10,13,0,6,9,8,7,4,15,14,3,11,5,2,12}
    },
    { // S4
        {7,13,14,3,0,6,9,10,1,2,8,5,11,12,4,15},
        {13,8,11,5,6,15,0,3,4,7,2,12,1,10,14,9},
        {10,6,9,0,12,11,7,13,15,1,3,14,5,2,8,4},
        {3,15,0,6,10,1,13,8,9,4,5,11,12,7,2,14}
    },
    { // S5
        {2,12,4,1,7,10,11,6,8,5,3,15,13,0,14,9},
        {14,11,2,12,4,7,13,1,5,0,15,10,3,9,8,6},
        {4,2,1,11,10,13,7,8,15,9,12,5,6,3,0,14},
        {11,8,12,7,1,14,2,13,6,15,0,9,10,4,5,3}
    },
    { // S6
        {12,1,10,15,9,2,6,8,0,13,3,4,14,7,5,11},
        {10,15,4,2,7,12,9,5,6,1,13,14,0,11,3,8},
        {9,14,15,5,2,8,12,3,7,0,4,10,1,13,11,6},
        {4,3,2,12,9,5,15,10,11,14,1,7,6,0,8,13}
    },
    { // S7
        {4,11,2,14,15,0,8,13,3,12,9,7,5,10,6,1},
        {13,0,11,7,4,9,1,10,14,3,5,12,2,15,8,6},
        {1,4,11,13,12,3,7,14,10,15,6,8,0,5,9,2},
        {6,11,13,8,1,4,10,7,9,5,0,15,14,2,3,12}
    },
    { // S8
        {13,2,8,4,6,15,11,1,10,9,3,14,5,0,12,7},
        {1,15,13,8,10,3,7,4,12,5,6,11,0,14,9,2},
        {7,11,4,1,9,12,14,2,0,6,10,13,15,3,5,8},
        {2,1,14,7,4,10,8,13,15,12,9,0,3,5,6,11}
    }
};

// PC-1 (密钥置换, 64 -> 56)
uint8_t PC1[56] =
{
    57,49,41,33,25,17,9,
    1,58,50,42,34,26,18,
    10,2,59,51,43,35,27,
    19,11,3,60,52,44,36,
    63,55,47,39,31,23,15,
    7,62,54,46,38,30,22,
    14,6,61,53,45,37,29,
    21,13,5,28,20,12,4
};

// PC-2 (密钥压缩, 56 -> 48)
uint8_t PC2[48] =
{
    14,17,11,24,1,5,
    3,28,15,6,21,10,
    23,19,12,4,26,8,
    16,7,27,20,13,2,
    41,52,31,37,47,55,
    30,40,51,45,33,48,
    44,49,39,56,34,53,
    46,42,50,36,29,32
};

// 每轮左移位数
uint8_t SHIFT[16] =
{
    1,1,2,2,2,2,2,2,1,2,2,2,2,2,2,1
};

// ==============================================





// ================= 辅助函数 ====================

// 置换函数：从 input 的 bit 位按 perm 顺序抽取，输出到 output
void permute(uint8_t* input, uint8_t* output, uint8_t* perm, int n)
{
    for (int i = 0; i < n; i++)
    {
        int src_bit = perm[i] - 1;
        output[i] = input[src_bit];
    }
}

// 异或: 对两个字节数组按位异或, 结果放out
void xor_bytes(uint8_t* a, uint8_t* b, uint8_t* out, int len)
{
    for (int i = 0; i < len; i++)
    {
        out[i] = a[i] ^ b[i];
    }
}

// 循环左移：对长度为 len 的位数组左移 n 位
static void left_shift(uint8_t* bits, int len, int n) {
    uint8_t temp[28];   // 最大28位
    n %= len;
    for (int i = 0; i < len; i++)
    {
        temp[i] = bits[(i + n) % len];
    }
    memcpy(bits, temp, len);
}

// 将 8 字节（64位）转换为位数组（每个字节展开为8位，高位在前）
void bytes_to_bits(uint8_t* bytes, uint8_t* bits)
{
    for (int i = 0; i < 8; i++)
    {
        for (int j = 0; j < 8; j++)
        {
            bits[i * 8 + j] = (bytes[i] >> (7 - j)) & 1;
        }
    }
}

// 将位数组（64位）转换回 8 字节
static void bits_to_bytes(const uint8_t* bits, uint8_t* bytes) {
    for (int i = 0; i < 8; i++) {
        uint8_t val = 0;
        for (int j = 0; j < 8; j++) {
            val |= (bits[i * 8 + j] << (7 - j));
        }
        bytes[i] = val;
    }
}

// ==============================================


// =============== 密钥调度 ======================
// 生成16个48位子密钥，存入 subkeys[16][48]
void key_schedule(uint8_t* key64, uint8_t subkeys[16][48])
{
    uint8_t key56[56];
    // PC-1 压缩至56位
    permute(key64, key56, PC1, 56);

    // 分成 C0, D0 (各28位)
    uint8_t C[28], D[28];
    for (int i = 0; i < 28; i++)
    {
        C[i] = key56[i];
        D[i] = key56[i + 28];
    }

    for (int round = 0; round < 16; round++)
    {
        // 循环左移
        left_shift(C, 28, SHIFT[round]);
        left_shift(D, 28, SHIFT[round]);

        // 合并 C+D 位56位
        uint8_t CD[56];
        memcpy(CD, C, 28);
        memcpy(CD + 28, D, 28);

        // PC-2 压缩至48位, 生成子密钥
        permute(CD, subkeys[round], PC2, 48);
    }

}

// 解密密钥生成(逆序子密钥)
void inv_key_schedule(uint8_t* key64, uint8_t subkeys[16][48])
{
    key_schedule(key64, subkeys);

    // 逆序子密钥
    uint8_t temp[48] = { 0 };
    for (int i = 0; i < 8; i++)
    {
        memcpy(temp, subkeys[i], 48);
        memcpy(subkeys[i], subkeys[15 - i], 48);
        memcpy(subkeys[15 - i], temp, 48);
    }

}

// =============================================



// =============== 轮函数f ======================
void f_function(uint8_t* R, uint8_t* K, uint8_t* out)
{
    uint8_t E_R[48];
    // 扩展置换 E
    permute(R, E_R, E, 48);

    // 与子密钥异或
    uint8_t X[48];
    xor_bytes(E_R, K, X, 48);

    // S盒替换 (48 -> 32)
    uint8_t S_out[32];
    for (int i = 0; i < 8; i++)
    {
        // 取6位: 第1位和第6位组成行，中间4位组成列
        int row = (X[i * 6] << 1) | X[i * 6 + 5];
        int col = (X[i * 6 + 1] << 3) | (X[i * 6 + 2] << 2) | (X[i * 6 + 3] << 1) | X[i * 6 + 4];
        uint8_t val = S[i][row][col];

        // 将4位值放入输出
        for (int j = 0; j < 4; j++)
        {
            S_out[i * 4 + j] = (val >> (3 - j)) & 1;
        }

    }

    // P置换
    permute(S_out, out, P, 32);

}
// =============================================



// des 加密
// 
// 加密一个64位块（8字节），key为8字节（64位，含奇偶校验）
void des_encrypt_block(uint8_t* plain_bits, uint8_t subkeys[16][48], uint8_t* cipher_bits)
{
    // 初始置换
    uint8_t IP_block[64];
    permute(plain_bits, IP_block, IP, 64);

    // 分成 L0, R0 (各32位)
    uint8_t L[32], R[32];
    memcpy(L, IP_block, 32);
    memcpy(R, IP_block + 32, 32);

    // 16轮迭代
    for (int round = 0; round < 16; round++)
    {
        uint8_t f_out[32];
        f_function(R, subkeys[round], f_out);

        uint8_t new_R[32];
        xor_bytes(L, f_out, new_R, 32);

        // 更新 L, R
        memcpy(L, R, 32);
        memcpy(R, new_R, 32);
    }

    // 合并 L32, R32 (注意：最后交换，所以输出是 R32 || L32)
    uint8_t pre_output[64];
    memcpy(pre_output, R, 32);
    memcpy(pre_output + 32, L, 32);

    // 逆初始置换
    permute(pre_output, cipher_bits, IP_INV, 64);

}

void des_encrypt_ecb(uint8_t* plaintext, int input_len, uint8_t* key, int key_len, uint8_t* ciphertext)
{
    if (input_len % 8 != 0 || key_len != 8)
    {
        printf("Length Wrong\n");
        return;
    }

    // 将字节转换为位数组
    uint8_t key_bits[64];
    bytes_to_bits(key, key_bits);

    // 16轮48位密钥生成
    uint8_t subkeys[16][48];
    key_schedule(key_bits, subkeys);

    int blockSize = input_len / 8;
    for (int i = 0; i < blockSize; i++)
    {
        uint8_t plain_bits[64], cipher_bits[64];
        bytes_to_bits(&plaintext[i * 8], plain_bits);
        des_encrypt_block(plain_bits, subkeys, cipher_bits);
        bits_to_bytes(cipher_bits, &ciphertext[i * 8]);
    }

}

void Triple_des_encrypt_ecb(uint8_t* plaintext, int input_len, uint8_t* key, int key_len, uint8_t* ciphertext)
{
    if (input_len % 8 != 0)
    {
        printf("Input Length Wrong\n");
        return;
    }

    uint8_t key_bits_first[64], key_bits_second[64], key_bits_third[64];
    if (key_len == 16)
    {
        // 将密钥字节转换为位数组
        bytes_to_bits(key, key_bits_first);
        bytes_to_bits(&key[8], key_bits_second);

        // 16轮48位密钥生成
        uint8_t subkeys_first[16][48], subkeys_second[16][48];
        key_schedule(key_bits_first, subkeys_first);
        inv_key_schedule(key_bits_second, subkeys_second);


        int blockSize = input_len / 8;
        for (int i = 0; i < blockSize; i++)
        {
            uint8_t plain_bits[64], cipher_bits_1[64], cipher_bits_2[64], cipher_bits_3[64];
            bytes_to_bits(&plaintext[i * 8], plain_bits);
            des_encrypt_block(plain_bits, subkeys_first, cipher_bits_1);
            des_encrypt_block(cipher_bits_1, subkeys_second, cipher_bits_2);
            des_encrypt_block(cipher_bits_2, subkeys_first, cipher_bits_3);
            bits_to_bytes(cipher_bits_3, &ciphertext[i * 8]);
        }


    }
    else if (key_len == 24)
    {
        // 将密钥字节转换为位数组
        bytes_to_bits(key, key_bits_first);
        bytes_to_bits(&key[8], key_bits_second);
        bytes_to_bits(&key[16], key_bits_third);

        // 16轮48位密钥生成
        uint8_t subkeys_first[16][48], subkeys_second[16][48], subkeys_third[16][48];
        key_schedule(key_bits_first, subkeys_first);
        inv_key_schedule(key_bits_second, subkeys_second);
        key_schedule(key_bits_third, subkeys_third);

        int blockSize = input_len / 8;
        for (int i = 0; i < blockSize; i++)
        {
            uint8_t plain_bits[64], cipher_bits_1[64], cipher_bits_2[64], cipher_bits_3[64];
            bytes_to_bits(&plaintext[i * 8], plain_bits);
            des_encrypt_block(plain_bits, subkeys_first, cipher_bits_1);
            des_encrypt_block(cipher_bits_1, subkeys_second, cipher_bits_2);
            des_encrypt_block(cipher_bits_2, subkeys_third, cipher_bits_3);
            bits_to_bytes(cipher_bits_3, &ciphertext[i * 8]);
        }
    }
    else
    {
        printf("Key Length Wrong\n");
        return;
    }

}


//blackmanba{D3s_01d_but_funny_vv}
//blackmanba{3D3s_01d_but_funny_v}
int main()
{
    uint8_t inputBuffer[0xff] = { 0 };
    uint8_t outputBuffer[0xff] = { 0 };
    uint8_t ciphertext[32] = { 0xBB,0x9F,0xB3,0xE1,0x41,0xF1,0xD8,0x9C,0x21,0xB4,0x59,0xA6,0x90,0x1F,0x67,0x68,0x4B,0x37,0x09,0xA2,0x69,0xFB,0xCF,0x64,0x0F,0x4E,0xDE,0x31,0x43,0x42,0x02,0x1E };
    uint8_t key[] = { 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x38, 0x39, 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x35, 0x35, 0x34, 0x36, 0x37, 0x38, 0x39, 0x31 };


    printf("Please input your flag: ");
    scanf("%s", inputBuffer);

    int length = strlen((char*)inputBuffer);
    int key_len = sizeof(key) / sizeof(key[0]);

    if (length != 32)
    {
        printf("wrong length\r\n");
        system("pause");
        return 0;
    }


    Triple_des_encrypt_ecb(inputBuffer, length, key, key_len, outputBuffer);

    for (int i = 0; i < length; i++)
    {
        if (ciphertext[i] != outputBuffer[i])
        {
            printf("Wrong\n");
            system("pause");
            return 0;
        }
    }

    printf("Correct\n");
    system("pause");
    return 0;
}
```



# 解密脚本
## C语言实现
```c
#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>

// =============== 置换表 ========================
// 初始置换IP
uint8_t IP[64] =
{
    58,50,42,34,26,18,10,2,
    60,52,44,36,28,20,12,4,
    62,54,46,38,30,22,14,6,
    64,56,48,40,32,24,16,8,
    57,49,41,33,25,17,9,1,
    59,51,43,35,27,19,11,3,
    61,53,45,37,29,21,13,5,
    63,55,47,39,31,23,15,7
};

// 逆初始置换ip_inv
uint8_t IP_INV[64] =
{
    40,8,48,16,56,24,64,32,
    39,7,47,15,55,23,63,31,
    38,6,46,14,54,22,62,30,
    37,5,45,13,53,21,61,29,
    36,4,44,12,52,20,60,28,
    35,3,43,11,51,19,59,27,
    34,2,42,10,50,18,58,26,
    33,1,41,9,49,17,57,25
};

// 扩展置换 E (32 - 48)
uint8_t E[48] =
{
    32,1,2,3,4,5,
    4,5,6,7,8,9,
    8,9,10,11,12,13,
    12,13,14,15,16,17,
    16,17,18,19,20,21,
    20,21,22,23,24,25,
    24,25,26,27,28,29,
    28,29,30,31,32,1
};

// P置换 (32 -> 32)
uint8_t P[32] =
{
    16,7,20,21,29,12,28,17,
    1,15,23,26,5,18,31,10,
    2,8,24,14,32,27,3,9,
    19,13,30,6,22,11,4,25
};

// 8个S盒 (每个 4x16)
uint8_t S[8][4][16] =
{
    { // S1
        {14,4,13,1,2,15,11,8,3,10,6,12,5,9,0,7},
        {0,15,7,4,14,2,13,1,10,6,12,11,9,5,3,8},
        {4,1,14,8,13,6,2,11,15,12,9,7,3,10,5,0},
        {15,12,8,2,4,9,1,7,5,11,3,14,10,0,6,13}
    },
    { // S2
        {15,1,8,14,6,11,3,4,9,7,2,13,12,0,5,10},
        {3,13,4,7,15,2,8,14,12,0,1,10,6,9,11,5},
        {0,14,7,11,10,4,13,1,5,8,12,6,9,3,2,15},
        {13,8,10,1,3,15,4,2,11,6,7,12,0,5,14,9}
    },
    { // S3
        {10,0,9,14,6,3,15,5,1,13,12,7,11,4,2,8},
        {13,7,0,9,3,4,6,10,2,8,5,14,12,11,15,1},
        {13,6,4,9,8,15,3,0,11,1,2,12,5,10,14,7},
        {1,10,13,0,6,9,8,7,4,15,14,3,11,5,2,12}
    },
    { // S4
        {7,13,14,3,0,6,9,10,1,2,8,5,11,12,4,15},
        {13,8,11,5,6,15,0,3,4,7,2,12,1,10,14,9},
        {10,6,9,0,12,11,7,13,15,1,3,14,5,2,8,4},
        {3,15,0,6,10,1,13,8,9,4,5,11,12,7,2,14}
    },
    { // S5
        {2,12,4,1,7,10,11,6,8,5,3,15,13,0,14,9},
        {14,11,2,12,4,7,13,1,5,0,15,10,3,9,8,6},
        {4,2,1,11,10,13,7,8,15,9,12,5,6,3,0,14},
        {11,8,12,7,1,14,2,13,6,15,0,9,10,4,5,3}
    },
    { // S6
        {12,1,10,15,9,2,6,8,0,13,3,4,14,7,5,11},
        {10,15,4,2,7,12,9,5,6,1,13,14,0,11,3,8},
        {9,14,15,5,2,8,12,3,7,0,4,10,1,13,11,6},
        {4,3,2,12,9,5,15,10,11,14,1,7,6,0,8,13}
    },
    { // S7
        {4,11,2,14,15,0,8,13,3,12,9,7,5,10,6,1},
        {13,0,11,7,4,9,1,10,14,3,5,12,2,15,8,6},
        {1,4,11,13,12,3,7,14,10,15,6,8,0,5,9,2},
        {6,11,13,8,1,4,10,7,9,5,0,15,14,2,3,12}
    },
    { // S8
        {13,2,8,4,6,15,11,1,10,9,3,14,5,0,12,7},
        {1,15,13,8,10,3,7,4,12,5,6,11,0,14,9,2},
        {7,11,4,1,9,12,14,2,0,6,10,13,15,3,5,8},
        {2,1,14,7,4,10,8,13,15,12,9,0,3,5,6,11}
    }
};

// PC-1 (密钥置换, 64 -> 56)
uint8_t PC1[56] =
{
    57,49,41,33,25,17,9,
    1,58,50,42,34,26,18,
    10,2,59,51,43,35,27,
    19,11,3,60,52,44,36,
    63,55,47,39,31,23,15,
    7,62,54,46,38,30,22,
    14,6,61,53,45,37,29,
    21,13,5,28,20,12,4
};

// PC-2 (密钥压缩, 56 -> 48)
uint8_t PC2[48] =
{
    14,17,11,24,1,5,
    3,28,15,6,21,10,
    23,19,12,4,26,8,
    16,7,27,20,13,2,
    41,52,31,37,47,55,
    30,40,51,45,33,48,
    44,49,39,56,34,53,
    46,42,50,36,29,32
};

// 每轮左移位数
uint8_t SHIFT[16] =
{
    1,1,2,2,2,2,2,2,1,2,2,2,2,2,2,1
};

// ==============================================





// ================= 辅助函数 ====================

// 置换函数：从 input 的 bit 位按 perm 顺序抽取，输出到 output
void permute(uint8_t* input, uint8_t* output, uint8_t* perm, int n)
{
    for (int i = 0; i < n; i++)
    {
        int src_bit = perm[i] - 1;
        output[i] = input[src_bit];
    }
}

// 异或: 对两个字节数组按位异或, 结果放out
void xor_bytes(uint8_t* a, uint8_t* b, uint8_t* out, int len)
{
    for (int i = 0; i < len; i++)
    {
        out[i] = a[i] ^ b[i];
    }
}

// 循环左移：对长度为 len 的位数组左移 n 位
static void left_shift(uint8_t* bits, int len, int n) {
    uint8_t temp[28];   // 最大28位
    n %= len;
    for (int i = 0; i < len; i++)
    {
        temp[i] = bits[(i + n) % len];
    }
    memcpy(bits, temp, len);
}

// 将 8 字节（64位）转换为位数组（每个字节展开为8位，高位在前）
void bytes_to_bits(uint8_t* bytes, uint8_t* bits)
{
    for (int i = 0; i < 8; i++)
    {
        for (int j = 0; j < 8; j++)
        {
            bits[i * 8 + j] = (bytes[i] >> (7 - j)) & 1;
        }
    }
}

// 将位数组（64位）转换回 8 字节
static void bits_to_bytes(const uint8_t* bits, uint8_t* bytes) {
    for (int i = 0; i < 8; i++) {
        uint8_t val = 0;
        for (int j = 0; j < 8; j++) {
            val |= (bits[i * 8 + j] << (7 - j));
        }
        bytes[i] = val;
    }
}

// ==============================================


// =============== 密钥调度 ======================
// 生成16个48位子密钥，存入 subkeys[16][48]
void key_schedule(uint8_t* key64, uint8_t subkeys[16][48])
{
    uint8_t key56[56];
    // PC-1 压缩至56位
    permute(key64, key56, PC1, 56);

    // 分成 C0, D0 (各28位)
    uint8_t C[28], D[28];
    for (int i = 0; i < 28; i++)
    {
        C[i] = key56[i];
        D[i] = key56[i + 28];
    }

    for (int round = 0; round < 16; round++)
    {
        // 循环左移
        left_shift(C, 28, SHIFT[round]);
        left_shift(D, 28, SHIFT[round]);

        // 合并 C+D 位56位
        uint8_t CD[56];
        memcpy(CD, C, 28);
        memcpy(CD + 28, D, 28);

        // PC-2 压缩至48位, 生成子密钥
        permute(CD, subkeys[round], PC2, 48);
    }

}

// 解密密钥生成(逆序子密钥)
void inv_key_schedule(uint8_t* key64, uint8_t subkeys[16][48])
{
    key_schedule(key64, subkeys);

    // 逆序子密钥
    uint8_t temp[48] = { 0 };
    for (int i = 0; i < 8; i++)
    {
        memcpy(temp, subkeys[i], 48);
        memcpy(subkeys[i], subkeys[15 - i], 48);
        memcpy(subkeys[15 - i], temp, 48);
    }

}
// =============================================



// =============== 轮函数f ======================
void f_function(uint8_t* R, uint8_t* K, uint8_t* out)
{
    uint8_t E_R[48];
    // 扩展置换 E
    permute(R, E_R, E, 48);

    // 与子密钥异或
    uint8_t X[48];
    xor_bytes(E_R, K, X, 48);

    // S盒替换 (48 -> 32)
    uint8_t S_out[32];
    for (int i = 0; i < 8; i++)
    {
        // 取6位: 第1位和第6位组成行，中间4位组成列
        int row = (X[i * 6] << 1) | X[i * 6 + 5];
        int col = (X[i * 6 + 1] << 3) | (X[i * 6 + 2] << 2) | (X[i * 6 + 3] << 1) | X[i * 6 + 4];
        uint8_t val = S[i][row][col];

        // 将4位值放入输出
        for (int j = 0; j < 4; j++)
        {
            S_out[i * 4 + j] = (val >> (3 - j)) & 1;
        }

    }

    // P置换
    permute(S_out, out, P, 32);

}
// =============================================



// des 解密
// 
// 解密一个64位块（8字节），key为8字节（64位，含奇偶校验）
void des_decrypt_block(uint8_t* cipher_bits, uint8_t subkeys[16][48], uint8_t* plain_bits)
{
    // 初始置换
    uint8_t IP_block[64];
    permute(cipher_bits, IP_block, IP, 64);

    // 分成 L0, R0 (各32位)
    uint8_t L[32], R[32];
    memcpy(L, IP_block, 32);
    memcpy(R, IP_block + 32, 32);

    // 16轮迭代
    for (int round = 0; round < 16; round++)
    {
        uint8_t f_out[32];
        f_function(R, subkeys[round], f_out);

        uint8_t new_R[32];
        xor_bytes(L, f_out, new_R, 32);

        // 更新 L, R
        memcpy(L, R, 32);
        memcpy(R, new_R, 32);
    }

    // 合并 L32, R32 (注意：最后交换，所以输出是 R32 || L32)
    uint8_t pre_output[64];
    memcpy(pre_output, R, 32);
    memcpy(pre_output + 32, L, 32);

    // 逆初始置换
    permute(pre_output, plain_bits, IP_INV, 64);

}

void des_decrypt_ecb(uint8_t* ciphertext, int input_len, uint8_t* key, int key_len, uint8_t* plaintext)
{
    if (input_len % 8 != 0 || key_len != 8)
    {
        printf("Length Wrong\n");
        return;
    }

    // 将字节转换为位数组
    uint8_t key_bits[64];
    bytes_to_bits(key, key_bits);

    // 16轮48位密钥生成
    uint8_t subkeys[16][48];
    inv_key_schedule(key_bits, subkeys);

    int blockSize = input_len / 8;
    for (int i = 0; i < blockSize; i++)
    {
        uint8_t plain_bits[64], cipher_bits[64];
        bytes_to_bits(&ciphertext[i * 8], cipher_bits);
        des_decrypt_block(cipher_bits, subkeys, plain_bits);
        bits_to_bytes(plain_bits, &plaintext[i * 8]);
    }

}

void des_decrypt_cbc(uint8_t* ciphertext, int input_len, uint8_t* key, int key_len, uint8_t* plaintext, uint8_t* IV)
{
    if (input_len % 8 != 0 || key_len != 8)
    {
        printf("Length Wrong\n");
        return;
    }

    // 将字节转换为位数组
    uint8_t key_bits[64];
    bytes_to_bits(key, key_bits);

    // 16轮48位密钥生成
    uint8_t subkeys[16][48];
    inv_key_schedule(key_bits, subkeys);

    int blockSize = input_len / 8;
    for (int i = 0; i < blockSize; i++)
    {
        uint8_t plain_bits[64], cipher_bits[64];
        bytes_to_bits(&ciphertext[i * 8], cipher_bits);
        des_decrypt_block(cipher_bits, subkeys, plain_bits);
        bits_to_bytes(plain_bits, &plaintext[i * 8]);
        if (i == 0)
        {
            for (int j = 0; j < 8; j++)
            {
                plaintext[j] ^= IV[j];
            }
        }
        else
        {
            for (int j = 0; j < 8; j++)
            {
                plaintext[i * 8 + j] ^= ciphertext[(i - 1) * 8 + j];
            }
        }
    }

}

void Triple_des_decrypt_ecb(uint8_t* ciphertext, int input_len, uint8_t* key, int key_len, uint8_t* plaintext)
{
    if (input_len % 8 != 0)
    {
        printf("Input Length Wrong\n");
        return;
    }

    uint8_t key_bits_first[64], key_bits_second[64], key_bits_third[64];
    if (key_len == 16)
    {
        // 将密钥字节转换为位数组
        bytes_to_bits(key, key_bits_first);
        bytes_to_bits(&key[8], key_bits_second);

        // 16轮48位密钥生成
        uint8_t subkeys_first[16][48], subkeys_second[16][48];
        inv_key_schedule(key_bits_first, subkeys_first);
        key_schedule(key_bits_second, subkeys_second);


        int blockSize = input_len / 8;
        for (int i = 0; i < blockSize; i++)
        {
            uint8_t cipher_bits[64], cipher_bits_1[64], cipher_bits_2[64], cipher_bits_3[64];
            bytes_to_bits(&ciphertext[i * 8], cipher_bits);
            des_decrypt_block(cipher_bits, subkeys_first, cipher_bits_1);
            des_decrypt_block(cipher_bits_1, subkeys_second, cipher_bits_2);
            des_decrypt_block(cipher_bits_2, subkeys_first, cipher_bits_3);
            bits_to_bytes(cipher_bits_3, &plaintext[i * 8]);
        }


    }
    else if (key_len == 24)
    {
        // 将密钥字节转换为位数组
        bytes_to_bits(key, key_bits_first);
        bytes_to_bits(&key[8], key_bits_second);
        bytes_to_bits(&key[16], key_bits_third);

        // 16轮48位密钥生成
        uint8_t subkeys_first[16][48], subkeys_second[16][48], subkeys_third[16][48];
        inv_key_schedule(key_bits_first, subkeys_first);
        key_schedule(key_bits_second, subkeys_second);
        inv_key_schedule(key_bits_third, subkeys_third);

        int blockSize = input_len / 8;
        for (int i = 0; i < blockSize; i++)
        {
            uint8_t cipher_bits[64], cipher_bits_1[64], cipher_bits_2[64], cipher_bits_3[64];
            bytes_to_bits(&ciphertext[i * 8], cipher_bits);
            des_decrypt_block(cipher_bits, subkeys_third, cipher_bits_1);
            des_decrypt_block(cipher_bits_1, subkeys_second, cipher_bits_2);
            des_decrypt_block(cipher_bits_2, subkeys_first, cipher_bits_3);
            bits_to_bytes(cipher_bits_3, &plaintext[i * 8]);
        }
    }
    else
    {
        printf("Key Length Wrong\n");
        return;
    }

}

//blackmanba{D3s_01d_but_funny_vv}
//blackmanba{3D3s_01d_but_funny_v}
int main()
{
    uint8_t ciphertext[32] = { 0xBB,0x9F,0xB3,0xE1,0x41,0xF1,0xD8,0x9C,0x21,0xB4,0x59,0xA6,0x90,0x1F,0x67,0x68,0x4B,0x37,0x09,0xA2,0x69,0xFB,0xCF,0x64,0x0F,0x4E,0xDE,0x31,0x43,0x42,0x02,0x1E };
    uint8_t key[] = { 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x38, 0x39, 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x35, 0x35, 0x34, 0x36, 0x37, 0x38, 0x39, 0x31 };
    //uint8_t IV[] = { 0x61, 0x62, 0x63, 0x64, 0x65, 0x66, 0x67, 0x68 };
    uint8_t plaintext[0xff] = { 0 };

    int cipher_len = sizeof(ciphertext) / sizeof(ciphertext[0]);
    int key_len = sizeof(key) / sizeof(key[0]);

    Triple_des_decrypt_ecb(ciphertext, cipher_len, key, key_len, plaintext);
    printf("%s\n", plaintext);
    return 0;
}
```


## python实现
```python

# 初始置换 IP

IP = [

    58, 50, 42, 34, 26, 18, 10, 2,

    60, 52, 44, 36, 28, 20, 12, 4,

    62, 54, 46, 38, 30, 22, 14, 6,

    64, 56, 48, 40, 32, 24, 16, 8,

    57, 49, 41, 33, 25, 17,  9, 1,

    59, 51, 43, 35, 27, 19, 11, 3,

    61, 53, 45, 37, 29, 21, 13, 5,

    63, 55, 47, 39, 31, 23, 15, 7

]

  

# 逆初始置换 IP^-1

IP_INV = [

    40, 8, 48, 16, 56, 24, 64, 32,

    39, 7, 47, 15, 55, 23, 63, 31,

    38, 6, 46, 14, 54, 22, 62, 30,

    37, 5, 45, 13, 53, 21, 61, 29,

    36, 4, 44, 12, 52, 20, 60, 28,

    35, 3, 43, 11, 51, 19, 59, 27,

    34, 2, 42, 10, 50, 18, 58, 26,

    33, 1, 41,  9, 49, 17, 57, 25

]

  

# 扩展置换 E (32 -> 48)

E = [

    32,  1,  2,  3,  4,  5,

     4,  5,  6,  7,  8,  9,

     8,  9, 10, 11, 12, 13,

    12, 13, 14, 15, 16, 17,

    16, 17, 18, 19, 20, 21,

    20, 21, 22, 23, 24, 25,

    24, 25, 26, 27, 28, 29,

    28, 29, 30, 31, 32,  1

]

  

# P 置换 (32 -> 32)

P = [

    16,  7, 20, 21, 29, 12, 28, 17,

     1, 15, 23, 26,  5, 18, 31, 10,

     2,  8, 24, 14, 32, 27,  3,  9,

    19, 13, 30,  6, 22, 11,  4, 25

]

  

# 8 个 S 盒 (每个 4x16)

S_BOX = [

    # S1

    [

        [14, 4, 13, 1, 2, 15, 11, 8, 3, 10, 6, 12, 5, 9, 0, 7],

        [0, 15, 7, 4, 14, 2, 13, 1, 10, 6, 12, 11, 9, 5, 3, 8],

        [4, 1, 14, 8, 13, 6, 2, 11, 15, 12, 9, 7, 3, 10, 5, 0],

        [15, 12, 8, 2, 4, 9, 1, 7, 5, 11, 3, 14, 10, 0, 6, 13]

    ],

    # S2

    [

        [15, 1, 8, 14, 6, 11, 3, 4, 9, 7, 2, 13, 12, 0, 5, 10],

        [3, 13, 4, 7, 15, 2, 8, 14, 12, 0, 1, 10, 6, 9, 11, 5],

        [0, 14, 7, 11, 10, 4, 13, 1, 5, 8, 12, 6, 9, 3, 2, 15],

        [13, 8, 10, 1, 3, 15, 4, 2, 11, 6, 7, 12, 0, 5, 14, 9]

    ],

    # S3

    [

        [10, 0, 9, 14, 6, 3, 15, 5, 1, 13, 12, 7, 11, 4, 2, 8],

        [13, 7, 0, 9, 3, 4, 6, 10, 2, 8, 5, 14, 12, 11, 15, 1],

        [13, 6, 4, 9, 8, 15, 3, 0, 11, 1, 2, 12, 5, 10, 14, 7],

        [1, 10, 13, 0, 6, 9, 8, 7, 4, 15, 14, 3, 11, 5, 2, 12]

    ],

    # S4

    [

        [7, 13, 14, 3, 0, 6, 9, 10, 1, 2, 8, 5, 11, 12, 4, 15],

        [13, 8, 11, 5, 6, 15, 0, 3, 4, 7, 2, 12, 1, 10, 14, 9],

        [10, 6, 9, 0, 12, 11, 7, 13, 15, 1, 3, 14, 5, 2, 8, 4],

        [3, 15, 0, 6, 10, 1, 13, 8, 9, 4, 5, 11, 12, 7, 2, 14]

    ],

    # S5

    [

        [2, 12, 4, 1, 7, 10, 11, 6, 8, 5, 3, 15, 13, 0, 14, 9],

        [14, 11, 2, 12, 4, 7, 13, 1, 5, 0, 15, 10, 3, 9, 8, 6],

        [4, 2, 1, 11, 10, 13, 7, 8, 15, 9, 12, 5, 6, 3, 0, 14],

        [11, 8, 12, 7, 1, 14, 2, 13, 6, 15, 0, 9, 10, 4, 5, 3]

    ],

    # S6

    [

        [12, 1, 10, 15, 9, 2, 6, 8, 0, 13, 3, 4, 14, 7, 5, 11],

        [10, 15, 4, 2, 7, 12, 9, 5, 6, 1, 13, 14, 0, 11, 3, 8],

        [9, 14, 15, 5, 2, 8, 12, 3, 7, 0, 4, 10, 1, 13, 11, 6],

        [4, 3, 2, 12, 9, 5, 15, 10, 11, 14, 1, 7, 6, 0, 8, 13]

    ],

    # S7

    [

        [4, 11, 2, 14, 15, 0, 8, 13, 3, 12, 9, 7, 5, 10, 6, 1],

        [13, 0, 11, 7, 4, 9, 1, 10, 14, 3, 5, 12, 2, 15, 8, 6],

        [1, 4, 11, 13, 12, 3, 7, 14, 10, 15, 6, 8, 0, 5, 9, 2],

        [6, 11, 13, 8, 1, 4, 10, 7, 9, 5, 0, 15, 14, 2, 3, 12]

    ],

    # S8

    [

        [13, 2, 8, 4, 6, 15, 11, 1, 10, 9, 3, 14, 5, 0, 12, 7],

        [1, 15, 13, 8, 10, 3, 7, 4, 12, 5, 6, 11, 0, 14, 9, 2],

        [7, 11, 4, 1, 9, 12, 14, 2, 0, 6, 10, 13, 15, 3, 5, 8],

        [2, 1, 14, 7, 4, 10, 8, 13, 15, 12, 9, 0, 3, 5, 6, 11]

    ]

]

  

# PC-1 (64 -> 56)

PC1 = [

    57, 49, 41, 33, 25, 17,  9,

     1, 58, 50, 42, 34, 26, 18,

    10,  2, 59, 51, 43, 35, 27,

    19, 11,  3, 60, 52, 44, 36,

    63, 55, 47, 39, 31, 23, 15,

     7, 62, 54, 46, 38, 30, 22,

    14,  6, 61, 53, 45, 37, 29,

    21, 13,  5, 28, 20, 12,  4

]

  

# PC-2 (56 -> 48)

PC2 = [

    14, 17, 11, 24,  1,  5,

     3, 28, 15,  6, 21, 10,

    23, 19, 12,  4, 26,  8,

    16,  7, 27, 20, 13,  2,

    41, 52, 31, 37, 47, 55,

    30, 40, 51, 45, 33, 48,

    44, 49, 39, 56, 34, 53,

    46, 42, 50, 36, 29, 32

]

  

# 每轮左移位数

SHIFT = [1, 1, 2, 2, 2, 2, 2, 2, 1, 2, 2, 2, 2, 2, 2, 1]

  
  

def bytes_to_bits(byte_data):

    """将字节串转换为位列表（每个元素 0/1）"""

    bits = []

    for b in byte_data:

        for i in range(7, -1, -1):

            bits.append((b >> i) & 1)

    return bits

  
  

def bits_to_bytes(bits):

    """将位列表转换为字节串（长度须为8的倍数）"""

    assert len(bits) % 8 == 0

    result = bytearray()

    for i in range(0, len(bits), 8):

        val = 0

        for j in range(8):

            val |= (bits[i + j] << (7 - j))

        result.append(val)

    return bytes(result)

  
  

def permute(bits, table):

    """根据置换表对位列表进行置换"""

    return [bits[i - 1] for i in table]

  
  

def left_shift(bits, n):

    """位列表循环左移 n 位"""

    n %= len(bits)

    return bits[n:] + bits[:n]

  
  

def xor_bits(a, b):

    """两个位列表异或，返回新列表"""

    return [x ^ y for x, y in zip(a, b)]

  
  

def generate_subkeys(key_bytes):

    """生成 DES 的 16 个 48 位子密钥（密钥为 8 字节）"""

    key_bits = bytes_to_bits(key_bytes)

    # PC-1 置换并移除校验位 -> 56 位

    key56 = permute(key_bits, PC1)

    C = key56[:28]

    D = key56[28:]

    subkeys = []

    for round_num in range(16):

        C = left_shift(C, SHIFT[round_num])

        D = left_shift(D, SHIFT[round_num])

        cd = C + D

        k = permute(cd, PC2)

        subkeys.append(k)

    return subkeys

  
  

def feistel(R, subkey):

    """DES 的 Feistel (F) 函数，R 为 32 位列表，subkey 为 48 位列表，返回 32 位列表"""

    # 扩展置换 E

    expanded = permute(R, E)

    # 与子密钥异或

    xored = xor_bits(expanded, subkey)

    # S 盒替换 (48 -> 32)

    s_output = []

    for i in range(8):

        chunk = xored[i * 6:(i + 1) * 6]

        row = (chunk[0] << 1) | chunk[5]

        col = (chunk[1] << 3) | (chunk[2] << 2) | (chunk[3] << 1) | chunk[4]

        val = S_BOX[i][row][col]

        # 将 4 位值转为位列表

        s_output += [(val >> (3 - j)) & 1 for j in range(4)]

    # P 置换

    return permute(s_output, P)

  
  

def des_block_encrypt(plain_bits, subkeys):

    """DES 加密一个 64 位块，plain_bits 为 64 位列表，subkeys 为 16 个 48 位子密钥列表"""

    # 初始置换

    ip = permute(plain_bits, IP)

    L = ip[:32]

    R = ip[32:]

    for i in range(16):

        f_out = feistel(R, subkeys[i])

        new_R = xor_bits(L, f_out)

        L, R = R, new_R

    # 最后交换并逆初始置换

    pre_output = R + L

    cipher_bits = permute(pre_output, IP_INV)

    return cipher_bits

  
  

def des_block_decrypt(cipher_bits, subkeys):

    """DES 解密一个 64 位块，使用逆序子密钥"""

    return des_block_encrypt(cipher_bits, subkeys[::-1])

  
  

def des_ecb_decrypt(ciphertext, key):

    """DES-ECB 解密，ciphertext 和 key 均为字节串，返回明文字节串"""

    if len(ciphertext) % 8 != 0:

        raise ValueError("密文长度必须是8的倍数")

    if len(key) != 8:

        raise ValueError("密钥长度必须为8字节")

    subkeys = generate_subkeys(key)

    plaintext = bytearray()

    for i in range(0, len(ciphertext), 8):

        block = bytes_to_bits(ciphertext[i:i+8])

        dec_bits = des_block_decrypt(block, subkeys)

        plaintext += bits_to_bytes(dec_bits)

    return bytes(plaintext)

  

def des_cbc_decrypt(ciphertext, key, IV):

    """DES-CBC 解密，ciphertext 和 key 均为字节串，返回明文字节串"""

    if len(ciphertext) % 8 != 0:

        raise ValueError("密文长度必须是8的倍数")

    if len(key) != 8:

        raise ValueError("密钥长度必须为8字节")

    subkeys = generate_subkeys(key)

    plaintext = bytearray()

    for i in range(0, len(ciphertext), 8):

        block = bytes_to_bits(ciphertext[i:i+8])

        dec_bits = des_block_decrypt(block, subkeys)

        dec_bytes = bits_to_bytes(dec_bits)

  

        iv_xored = bytearray()

        if (i == 0):

            for j in range(8):

                iv_xored.append(dec_bytes[j] ^ IV[j])

        else:

            for j in range(8):

                iv_xored.append(dec_bytes[j] ^ ciphertext[i - 8 + j])

        plaintext += iv_xored

  

    return bytes(plaintext)

  

def triple_des_ecb_decrypt(ciphertext, key):

    """

    3DES-ECB 解密（密钥选项 3：三个独立密钥，24 字节）

    解密顺序：D(K1) -> E(K2) -> D(K3)

    """

    if len(key) == 16:

        # 2-key 模式：K1 = key[0:8], K2 = key[8:16], K3 = K1

        k1 = key[:8]

        k2 = key[8:16]

        k3 = k1

    elif len(key) == 24:

        k1 = key[:8]

        k2 = key[8:16]

        k3 = key[16:24]

    else:

        raise ValueError("密钥长度必须是 16 或 24 字节")

  

    # 生成每个密钥的子密钥（解密顺序：K1解密，K2加密，K3解密）

    subkeys_k1 = generate_subkeys(k1)   # 用于解密

    subkeys_k2 = generate_subkeys(k2)   # 用于加密

    subkeys_k3 = generate_subkeys(k3)   # 用于解密

  

    plaintext = bytearray()

    for i in range(0, len(ciphertext), 8):

        block = bytes_to_bits(ciphertext[i:i+8])

        # 第一层：用 K1 解密

        temp1 = des_block_decrypt(block, subkeys_k3)

        # 第二层：用 K2 加密

        temp2 = des_block_encrypt(temp1, subkeys_k2)

        # 第三层：用 K3 解密

        final = des_block_decrypt(temp2, subkeys_k1)

        plaintext += bits_to_bytes(final)

    return bytes(plaintext)

  
  
  
  

def main():

  

    ciphertext = [0xBB,0x9F,0xB3,0xE1,0x41,0xF1,0xD8,0x9C,0x21,0xB4,0x59,0xA6,0x90,0x1F,0x67,0x68,0x4B,0x37,0x09,0xA2,0x69,0xFB,0xCF,0x64,0x0F,0x4E,0xDE,0x31,0x43,0x42,0x02,0x1E]

    key = [0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x38, 0x39, 0x31, 0x32, 0x33, 0x34, 0x35, 0x36, 0x37, 0x35, 0x35, 0x34, 0x36, 0x37, 0x38, 0x39, 0x31]

    cipherbytes = bytes(ciphertext)

    keybytes = bytes(key)

    plaintext = triple_des_ecb_decrypt(cipherbytes, keybytes)

    print(plaintext)

  

    cipherbytes_2 = bytes.fromhex("69142E70E25FE107389CAE0DF245EFC03D1F3AA8E3B22BD6CB41D71E49EF453E")

    keybytes_2 = b'12345678'

    IV = b'abcdefgh'

    plaintext_2 = des_cbc_decrypt(cipherbytes_2, keybytes_2, IV)

    print(plaintext_2)

  

    return

  
  

if __name__ == "__main__":

    main()
```




