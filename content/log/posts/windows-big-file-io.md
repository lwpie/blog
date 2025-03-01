+++
title = 'Windows 大文件读写问题'
date = 2020-02-22
draft = false
+++

之前写的 [Huffman 编解码器](https://github.com/lwpie/huffman) 读写超过 4G 文件会报 `basic_filebuf::xsgetn error reading the file: iostream error`, 但是 C++ 标准规定大小是 size_t 不应该出问题。后来在群友提醒下发现 [Windows API](https://docs.microsoft.com/windows/win32/api/fileapi/nf-fileapi-writefileex) 接受的参数是 DWORD 类型也即 unsigned int，于是超过 4G 就报错了。

```C++
BOOL WriteFileEx(
  HANDLE                          hFile,
  LPCVOID                         lpBuffer,
  DWORD                           nNumberOfBytesToWrite,
  LPOVERLAPPED                    lpOverlapped,
  LPOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine
);
```

于是根据大小分了一下块进行读取，写入同理

```C++
char *buffer = new char[size];
std::filebuf *ptr = in.rdbuf();
size_t size = ptr->pubseekoff(0, std::ios::end, std::ios::in);
ptr->pubseekpos(0, std::ios::in);
for (size_t i = 0; (i << 10) < size; i++)
	ptr->sgetn(buffer + (i << 10), std::min(size_t(1 << 10), size - (i << 10)));
```

接下来应该会改成在线模式减少内存用量，另外尝试用多线程加速处理过程。
___

## Huffman

Usage

```bash
./encode --infile input --outfile output
./decode --infile input --outfile output
```

Speed

Tested on i5-8265U @ 1.60 GHz, single core.

Use [2018-11-13-raspbian-stretch-lite.img](https://mirrors.tuna.tsinghua.edu.cn/raspberry-pi-os-images/raspbian_lite/images/raspbian_lite-2018-11-15/2018-11-13-raspbian-stretch-lite.zip) as sample file.

```
> encode --infile j --outfile d
0.00	Started
File Size: 1.74G
0.83	File Read
4.37	Char Counted
4.37	Tree Constructed
4.37	Tree Walked
20.25	File Encoded
20.44	Stopped
Overall Speed: 87.08M/s
Compression Rate: 47.68%

> decode --infile d --outfile s
0.18	Started
0.19	Huffman Loaded
File Size: 848.76M
0.61	File Read
20.01	File Decoded
20.01	Stopped
Overall Speed: 88.94M/s
```
