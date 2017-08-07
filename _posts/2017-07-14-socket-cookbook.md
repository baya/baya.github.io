---
layout: post
title: Socket Cookbook
---

所有例子都来源于自己的实际编码中，并将不断补充。

1. 只记录和 socket 编程相关的东西

2. 应该记录会经常用到的东西, 并且至少用过一次

3. 记录的例子尽量精简，并且能够正确执行


## inet_pton 和 inet_ntop

> inet_pton - convert IPv4 and IPv6 addresses from text to binary form

> inet_ntop - convert IPv4 and IPv6 addresses from binary to text form

```c
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void)
{
    int domain, s;
    unsigned char buf[sizeof(struct in6_addr)];
    char *src = "::ffff:127.0.0.1";
    char str[INET6_ADDRSTRLEN];

    domain = AF_INET6;
    s = inet_pton(domain, src, buf);
    if (s <= 0) {
	if (s == 0)
	    fprintf(stderr, "Not in presentation format");
	else
	    perror("inet_pton");
	exit(EXIT_FAILURE);
    }

    if (inet_ntop(domain, buf, str, INET6_ADDRSTRLEN) == NULL) {
	perror("inet_ntop");
	exit(EXIT_FAILURE);
    }

    printf("%s\n", str);
    exit(EXIT_SUCCESS);
}

```
