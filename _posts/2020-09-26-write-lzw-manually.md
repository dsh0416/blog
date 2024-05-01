---
title: 手写一个 LZW 压缩算法
date: 2020-09-26 16:05:10 +0900
tags: [LZW, C语言, 算法, 压缩]
---

## 起因

之前需要在某个单片机下塞点阵字库，为了能多覆盖一些字，准备在字库上做一点压缩。由于常用字相邻编码通常是按形码编码的，所以形状上有很多相似性，因此应该是比较可以压缩的。读取的时候，把一整块相邻编码解压塞到内存里，在内存里做个 LRU 缓存。由于常用字的编码也比较靠近，所以可以一定程度上在覆盖生僻字的同时，达到比较好的读取性能。

不过单片机上写个压缩算法比较麻烦。单片机本身的性能就很差，压缩算法本身约简单越好。最好实现的压缩算法恐怕就是 LZW 了。由于 LZW 的字典是自解释的，也不需要单独构建霍夫曼树，one-pass 一遍读完就解决。于是就考虑写个 LZW。

## LZW 基本原理

LZW 压缩需要两件东西，一个是字符集一个是字典。最常见的字符集就是 `0x00` 到 `0xff` 的 256 个字符当作字符集，即我们把所以 8-bit 数据看成一个单独字符。字典是这个字符集的组合。字典构成一个更大的空间，通常教科书上的例子的是 14-bit 的字典空间。也就是 `0x0000-0x1fff`，其中 `0x0000-0x00ff` 是基础字符集，`0x0100-0x1fff` 7935 个编号作为可编码的字典。这对于压缩率是比较好的，不过 14-bit 的读取实在太麻烦了，因为它不是 byte 的整倍数。于是我把可编码的字典空间放达到 2 个 bytes 也就是 `0x0100-0xffff` 65279 个字符，和字符集一起构成一个 65536 字符的空间。

## 函数签名

头文件和函数签名非常简单：
```c
#include <stdlib.h>
#include <assert.h>
#include <string.h>
#include <stdio.h>
#include <stdbool.h>

struct dictionary {
    size_t count;
    size_t entries_sizes[65536];
    char* entries[65536];
};

struct dictionary* dictionary_init();
void dictionary_free(struct dictionary* dict);
size_t dictionary_insert(struct dictionary* dict, const char* word, size_t size); // The string would be copied, be sure to free the word.
size_t dictionary_find(struct dictionary* dict, const char* word, size_t size);
void compress(const char* source, size_t size, const char* filepath);
char* decompress(size_t* size, const char* filepath);
```

维护字典大小、插入字典、根据字符查找在字典的位置、执行压缩和解压缩。

其中字典的创建、释放和插入是基本的 C 语言常识：

```c
struct dictionary* dictionary_init() {
    struct dictionary* dict = (struct dictionary*) malloc(sizeof(struct dictionary));
    dict->count = 256;

    for (size_t i = 0; i < 256; i++) {
        dict->entries[i] = (char*) malloc(sizeof(char));
        dict->entries[i][0] = (char) i;
        dict->entries_sizes[i] = 1;
    }

    return dict;
}

void dictionary_free(struct dictionary* dict) {
    for (size_t i = 0; i < dict->count; i++) {
        free(dict->entries[i]);
    }
    free(dict);
}

size_t dictionary_insert(struct dictionary* dict, const char* word, size_t size) {
    char* w = (char*) malloc(sizeof(char) * size);
    size_t idx = dict->count;

    assert(idx < 65536);
    memcpy(w, word, size);

    dict->entries[idx] = w;
    dict->entries_sizes[idx] = size;
    dict->count++;
    return idx;
}
```

查找的一个比较好的实现方法是使用哈希（`unordered_map`）。不过由于我在使用 C 语言，没有炫酷的 C++ STL 标准库。考虑到字典最大就 65536 个，而且用 size 大小就能很好做初步的筛选，我还是整个遍历一遍算了。

```c
size_t dictionary_find(struct dictionary* dict, const char* word, size_t size) {
    size_t i;
    for (i = 0; i < dict->count; i++) {
        if (dict->entries_sizes[i] == size) {
            bool found = true;
            for (size_t j = 0; j < size; j++) {
                if (dict->entries[i][j] != word[j]) {
                    found = false;
                    break;
                }
            }
            if (found) { return i; }
        }
    }
    return i;
}
```

## 压缩过程

LZW 之所以不需要单独维护字典是因为 LZW 对于如何建立字典这件事情是 **隐含** 在算法中的。对于当前压缩过程，如果前一个字典字和当前的字符的组合没有出现在字典中，那么就插入到字典中。为了理解这个概念，我们先简化一下模型。我们假设基本字符集只有 `0` 和 `1` 两个字符，然后我们编码这个序列：`01001101`

| 前一个字典字 | 当前字符 | 构成的组合 | 是否出现在字典中？        | 输出 |
| ------------ | -------- | ---------- | ------------------------- | ---- |
| -            | 0        | 0          | 出现（基本字符 0 -> 0）   | -    |
| 0            | 1        | 01         | 没有（插入字典 2 -> 01）  | 0    |
| 1            | 0        | 10         | 没有（插入字典 3 -> 10）  | 1    |
| 0            | 0        | 00         | 没有（插入字典 4 -> 00）  | 0    |
| 0            | 1        | 01         | 出现（2 -> 01）           | -    |
| 01           | 1        | 011        | 没有（插入字典 5 -> 011） | 2    |
| 1            | 1        | 11         | 没有（插入字典 6 -> 11）  | 1    |
| 1            | 0        | 10         | 出现（3 -> 10）           | -    |
| 10           | 1        | 101        | 没有（插入字典 7 -> 101） | 3    |
| 1            | -        | 1          |                           | 1    |

于是我们把 `01001101` 编码成了 `0102131`，这个序列的压缩率是非常糟糕的，因为 01 可以用一个 bit 表示，`01001101` 只有 1 byte，但压缩后的 `0102131` 至少需要 2 个 bit 来表示一个字符，虽然整体数量减少到了 7 个字符，但是实际上需要 14-bit（1.75 bytes）才能存储。但随着字符变长，字典覆盖会越来越好，压缩率也会越来越低。

我在实际实现的时候还标注了一个元信息，就是用第一个 `size_t` 来标注源文件的大小，以便于之后解压的时候申请内存。

```c
void compress(const char* source, size_t size, const char* filepath) {
    FILE* destination = fopen(filepath, "wb+");
    struct dictionary* dict = dictionary_init();

    // First size_t indicates the file size
    fwrite(&size, sizeof(size_t), 1, destination);

    char* p = NULL;
    size_t p_size = 0;
    char c;

    for (size_t i = 0; i < size; i++) {
        char* ppc = (char*) malloc(p_size + 1); // p + c

        c = source[i];
        memcpy(ppc, p, p_size);
        ppc[p_size] = c;
        size_t idx = dictionary_find(dict, ppc, p_size+1);
        if (idx < dict->count) {
            // Found
            if (p != NULL) { free(p); }
            p_size = p_size + 1;
            p = malloc(sizeof(char) * p_size);
            memcpy(p, ppc, p_size);
        } else {
            // Not found
            assert(p != NULL);
            unsigned short p_res = dictionary_find(dict, p, p_size);
            fwrite(&p_res, sizeof(unsigned short), 1, destination);
            free(p); p = NULL;
            dictionary_insert(dict, ppc, p_size + 1);
            p = (char*) malloc(sizeof(char));
            p[0] = c;
            p_size = 1;
            if (i == size - 1) {
                unsigned short c_res = dictionary_find(dict, p, 1);
                fwrite(&c_res, sizeof(unsigned short), 1, destination);
            }
        }

        free(ppc);
    }

    if (p != NULL) {
        free(p);
    }

    dictionary_free(dict);
    fclose(destination);
}
```

## 解压过程

LZW 的压缩是比较简单的，但是解压却是有点 tricky 的。如果我们顺着压缩的思路来解压，我们会认为，顺着我们解压的过程，我们会慢慢构建出我们需要的字典。我们考虑下面一个序列 `10101` 的压缩过程：

| 前一个字典字 | 当前字符 | 构成的组合 | 是否出现在字典中？        | 输出 |
| ------------ | -------- | ---------- | ------------------------- | ---- |
| -            | 1        | 1          | 出现（基本字符 1 -> 1）   | -    |
| 1            | 0        | 10         | 没有（插入字典 3 -> 10）  | 1    |
| 0            | 1        | 01         | 没有（插入字典 4 -> 01）  | 0    |
| 1            | 0        | 10         | 出现（3 -> 10）           | -    |
| 10           | 1        | 101        | 没有（插入字典 5 -> 101） | 3    |
| 1            | -        | 1          |                           | 1    |

输出的结果是 `1031`。如果我们解压的话过程如下：

| 前一个字典字 | 当前字符 | 构成的组合 | 是否出现在字典中？       | 输出 |
| ------------ | -------- | ---------- | ------------------------ | ---- |
| -            | 1        | 1          | 出现（基本字符 1 -> 1）  | 1    |
| 1            | 0        | 10         | 没有（插入字典 3 -> 10） | 0    |
| 0            | 3 (10)   | 010        | **没有（4 -> 010）**     | 10   |

显然我们把第四个字典的字符值插入错了，应该是 `01` 而我们却组合出了 `010`。这会进一步导致之后的解压出错。而且我们很容易思考到一个问题，在压缩过程中我们组合的都是前一个字典字和一个 **单一字符**，这里我们让 0 和 10 相加，后者显然不是单一字符。进一步思考我们会发现，在压缩过程中如果一个字典被创建，那么这个步骤的前一个插入字典的尾部，必然是这个匹配到的前缀，也就是这个字典值的第一个字符。所以这里的 `4` 应该是前一个字典字 `0` 和 `10` 的第一个字符 1 的组合，也就是 `01`。

于是我们以此正确构建我们的解压代码：

```c
char* decompress(size_t* size, const char* filepath) {
    FILE* source = fopen(filepath, "rb");
    struct dictionary* dict = dictionary_init();

    // First size_t indicates the file size
    fread(size, sizeof(size_t), 1, source);

    char* result = malloc(sizeof(char) * (*size));
    size_t counter = 0;

    if (size == 0) {
        dictionary_free(dict);
        return result;
    }

    unsigned short p_index;
    unsigned short c_index;
    char* c_word;
    size_t c_word_size;
    char* p_word = NULL;
    size_t p_word_size = 0;

    fread(&p_index, sizeof(unsigned short), 1, source);
    p_word = dict->entries[p_index];
    p_word_size = dict->entries_sizes[p_index];
    memcpy(result+counter, p_word, p_word_size);
    counter += p_word_size;


    while (counter < *size) {
        fread(&c_index, sizeof(unsigned short), 1, source);
        if (c_index < dict->count) {
            // Found
            c_word = dict->entries[c_index];
            c_word_size = dict->entries_sizes[c_index];
            memcpy(result+counter, c_word, c_word_size);
            counter += c_word_size;
            char* ppc = malloc(sizeof(char) * (p_word_size + 1));
            memcpy(ppc, p_word, p_word_size);
            memcpy(ppc+p_word_size, c_word, 1);
            dictionary_insert(dict, ppc, p_word_size + 1);
            free(ppc);
        } else {
            char* ppc = malloc(sizeof(char) * (p_word_size + 1));
            memcpy(ppc, p_word, p_word_size);
            memcpy(ppc+p_word_size, p_word, 1);
            c_index = dictionary_insert(dict, ppc, p_word_size + 1);
            memcpy(result+counter, ppc, p_word_size + 1);
            c_word = dict->entries[c_index];
            c_word_size = p_word_size + 1;
            counter += p_word_size + 1;
            free(ppc);
        }
        p_word = c_word;
        p_word_size = c_word_size;
    }

    dictionary_free(dict);
    fclose(source);

    return result;
}
```

## 实验

我尝试用马丁路德金的《I have a dream》作为例子实验了一下这个实现：

```c
int main() {
    const char path[] = "/tmp/compressed.lzw";
    char test[] = "I am happy to ...";
    compress(test, sizeof(test), path);

    size_t s = 0;

    FILE* f = fopen(path, "rb");
    fseek(f, 0, SEEK_END); // seek to end of file
    size_t file_size = ftell(f);
    fclose(f);

    char* res = decompress(&s, path);

    assert(sizeof(test) == s);
    assert(strcmp(test, res) == 0);

    printf("         Raw Text: %s\n", test);
    printf("Decompressed Text: %s\n", res);
    printf(" Compression Rate: %d%%\n",  (unsigned int)(file_size * 1.0 / sizeof(test) * 100));
    return 0;
}
```

最后的输出如下：

```
/Users/delton/CLionProjects/playgound/cmake-build-debug/playgound
         Raw Text: I am happy to ...
Decompressed Text: I am happy to ...
 Compression Rate: 73%
```

整体压缩率有 73%，对比了一下 zip 42% 的压缩率，实在是望尘莫及。

如果我们把文本重复 5 遍，压缩率能提升到 49%。不过这条件下，zip 能提高到 9% 的压缩率。果然还是不能和 DEFLATE 这种 LZW + 霍夫曼树这样的怪物比啊。不过简简单单 200 行代码就能实现一个性能上还不错的压缩、解压缩算法我已经比较满意了。
