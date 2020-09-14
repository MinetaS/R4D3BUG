---
description: Address Sanitizer
---

# ASAN

## Introduction

Google에서 memory corruption bug를 탐지하기 위해 만든 open source programming tool이다.

## What It Looks Like

아래의 코드를 컴파일해보며 ASAN을 사용했을 때, 결과가 어떻게 달라지는 살펴보자.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define SENT_LEN 27

int main() {
	char s[] = "This is popo from popoland.";
	char buf[SENT_LEN];

	strcpy(buf,s);

	printf("'%s' is what you got.", buf);
	return EXIT_SUCCESS;
}
```

별다른 기능 없이, 문자열 `s`를 `buf`로 옮기고 이를 출력하는 코드이다.

### Without ASAN

ASAN 없이 GCC로 컴파일하고 실행해보면 아무런 문제가 생기지 않는다.

```bash
$ gcc -o popo po.c <input>
$ ./popo
'This is popo from popoland.' is what you got.
```

### With ASAN

그렇다면 ASAN을 넣으면 어떻게 될까? ASAN은 `-fsanitize=address` 플래그를 달아주어 사용할 수 있다.

```bash
$ gcc -o popo po.c -fsanitize=address
$ ./popo
=================================================================
==97160==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7ffd292c492b at pc 0x7f982ab303a6 bp 0x7ffd292c48e0 sp 0x7ffd292c4088
WRITE of size 28 at 0x7ffd292c492b thread T0
    #0 0x7f982ab303a5  (/usr/lib/x86_64-linux-gnu/libasan.so.4+0x663a5)
    #1 0x556f26e00c1f in main (/home/cereal/asan/popo+0xc1f)
    #2 0x7f982a6fab96 in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x21b96)
    #3 0x556f26e009e9 in _start (/home/cereal/asan/popo+0x9e9)

Address 0x7ffd292c492b is located in stack of thread T0 at offset 59 in frame
    #0 0x556f26e00ad9 in main (/home/cereal/asan/popo+0xad9)

  This frame has 2 object(s):
    [32, 59) 'buf' <== Memory access at offset 59 overflows this variable
    [96, 124) 's'
HINT: this may be a false positive if your program uses some custom stack unwind mechanism or swapcontext
      (longjmp and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-buffer-overflow (/usr/lib/x86_64-linux-gnu/libasan.so.4+0x663a5) 
Shadow bytes around the buggy address:
  0x1000252508d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1000252508e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x1000252508f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x100025250900: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x100025250910: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 f1 f1
=>0x100025250920: f1 f1 00 00 00[03]f2 f2 f2 f2 00 00 00 04 f3 f3
  0x100025250930: f3 f3 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x100025250940: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x100025250950: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x100025250960: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x100025250970: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==97160==ABORTING
```

잘만되던 코드가 엄청나게 긴 에러를 출력하며 작동하지 않는다.

`WRITE of size 28 at 0x7ffd292c492b thread T0`라고 되어 있는 것으로 보아, `s`의 길이가 널바이트를 포함했을 때, 28이 되어 `buf`의 길이를 넘어간 것이 문제가 된 듯하다.

그 뒤에 백트레이스 및 오류 발생 위치 주변의 메모리를 출력해준다. 메모리 출력 부분에서 `=>` 되어 있는 곳을 보면 `00 00 00 [03]`으로 되어있는 부분이 있는데 여기가 `buf`이다. 코드 상으로는 길이가 27이어야 하는데 어째서 4바이트로 표현되었는지 의아해할 수 있는데, ASAN은 shadow byte를 사용하여 한 바이트가 실제로는 8바이트를 나타낸다.

아래의 Shadow byte legend를 참고하면 각각의 바이트가 무엇을 의미하는지 알 수 있는데, `00`은 8바이트가 전부 할당 가능함을 의미하며, `01 02 03 ... 07`은 부분적으로 할당 가능함을 의미한다. 이외에 `f`로 시작하는 부분은 할당이 되면 안되는 부분이라고 보면 된다. 

그래서 다시 `00 00 00 [03]`으로 돌아와서 보면 `00`은 각각 8바이트를, `03`은 3바이트를 나타내어 총 27 바이트가 되는 것을 알 수 있다.

`f`로 시작하는 `redzone`들은 어떻게 생긴 것인지는 ASAN이 어떻게 작동하는지에서 보자.

## How It Works

AddressSanitizer는 기본적으로 `malloc`, `free`와 관련된 함수들을 수정하여  할당될 영역 주변에 `redzone`을 설정한다. 이후 프로그램에서 메모리에 접근할 때, `redzone`와 겹치는지 확인을 하고 에러를 출력하는 방식이다.

자세한 내용은 일단 Reference의 pdf 파일 읽어주세



#### References

[https://github.com/google/sanitizers/wiki/AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer)

[https://www.usenix.org/system/files/conference/atc12/atc12-final39.pdf](https://www.usenix.org/system/files/conference/atc12/atc12-final39.pdf)

