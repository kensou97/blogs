# 1 背景

有一个客户的POC，客户使用Debezium做cdc，将数据库数据序列化以avro的格式写入到kafka，而当处理decimal类型时，发现我们的系统无法将其正确的反序列化。

# 2 解决

首先看了avro相关的文档，[decimal](https://avro.apache.org/docs/1.8.2/spec.html#Logical+Types)属于logical type，其表示是一个无符号字节数组表示的大整数unscaled，以及一个scale属性。

其计算方式为：unscaled*10^(-scale)。

我们依赖的是[avro](https://github.com/apache/avro)库中的[avro-c](https://github.com/apache/avro/tree/main/lang/c)，也就是c语言版本的实现，好像c语言版本已经没有活跃的维护者了，查看了代码和文档，发现并没有支持logical type，有一个[pull request](https://github.com/apache/avro/pull/843)，不过没有人review。

先把这个pull request拉到本地看了看代码，测试了一下，发现似乎没有问题。本地build的时候，还发现了两个问题，顺便帮忙提[pull request](https://github.com/apache/avro/pulls?q=is%3Apr+author%3Akensou97+is%3Aclosed)修复了，看起来确实是年久失修了...

不过这个支持的只是识别logical type，也就是可以拿到decimal相关的属性如scale了，不过值还是需要自己计算。

因为数据库是可以识别280000e-2这种科学技术法的字符串的，因此我们的任务就是求出这样科学技术法的char*返回就可以了。

"e-2"这个部分是简单的，实际上就是把scale拼接上就可以了。



那么问题就是求前边的unscaled部分了，由于其数值可能非常大，自己实现的话很有挑战，于是去网上找一些库...

发现有个[libgmp](https://gmplib.org/)是解决大数计算问题的，而且幸运的是，几乎主流linux发行版都自带了这个库。



看起来我们可以直接使用[mpz_import](https://gmplib.org/manual/Integer-Import-and-Export)将数组buf导入就可以了，但是其实对于负数的情况要特殊处理一下，

因为mpz_import总是将数组buf存储的大数当成无符号数来处理，负数在buf中其实是补码表示。

因此如果是负数，我们需要先将其按位取反，import之后再+1，再取负号即可。

```c
#include "gmp.h"
/**
* calculate the decimal literal according to buf and scale
* result = unscaled * 10^(-scale)
* see: https://avro.apache.org/docs/1.8.2/spec.html#Decimal
* @param buf stores the unscaled number with byte array format
* @param buf_size the size of buf array
* @param scale
* @return literal representation of the decimal value
**/
static char* calculate_decimal_literal(char *buf, size_t buf_size, int scale){
  char *scale_s = int_to_string(scale);
  
  int negative = buf[0] < 0 ? 1 : 0;
  mpz_t unscaled;
  mpz_init(unscaled);
  /*
  if unscaled is negative, it is stored with two's complement, 
  because mpz_import always treat buf as an unsigned number, 
  so we can't import with mpz_import directly, 
  we should calculate its complement(which is the absolute value), then multiply -1.
  
  how to calculate its complement? there's no direct gmp api, 
  so we can flip the buf first, then add 1.
  
  e.g. -280000
  buf:				1111 1011 1011 1010 0100 0000 (-280000)
  flip buf:		0000 0100 0100 0101 1011 1111	(279999)
  flip buf+1:	0000 0100 0100 0101 1100 0000 (280000)
  */
  if(negative){
      size_t i=0;
      for(; i < buf_size; i++){
        	// flip buf
        	buf[i] = ~buf[i];
      }
  }
  // https://gmplib.org/manual/Integer-Import-and-Export
  mpz_import(unscaled, buf_size, 1, sizeof(char), 0, 0, buf);
  if(negative){
    	mpz_add_ui(unscaled, unscaled, 1);
    	mpz_neg(unscaled, unscaled);
  }
  
  char *unscaled_s = mpz_get_str(NULL, 10, unscaled);
  // e.g. -280000e-2\0
  int length = strlen(unscaled_s) + 2 + strlen(scaled_s) + 1;
  char *s = malloc(length);
  snprintf(s, length, "%se-%s", unscaled_s, scale_s);
  
  // release resources
  mpz_clear(unscaled);
  free(unscaled_s);
  free(scale_s);
  return s;
}
```