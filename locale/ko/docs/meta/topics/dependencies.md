---
title: 의존성
layout: docs.hbs
---

# 의존성

Node.js가 의존하고 있는 여러 의존성

- [라이브러리](#libraries)
  - [V8](#v8)
  - [libuv](#libuv)
  - [http-parser](#http-parser)
  - [c-ares](#c-ares)
  - [OpenSSL](#openssl)
  - [zlib](#zlib)
- [도구](#tools)
  - [npm](#npm)
  - [gyp](#gyp)
  - [gtest](#gtest)

## <!--libraries-->라이브러리

### V8

V8 라이브러리는 Node.js가 V8 C++ API로 제어하는 JavaScript 엔진을 제공합니다.
V8은 구글이 관리하고 크롬에서 사용 중인 엔진입니다.

- [문서](https://v8docs.nodesource.com/)

### libuv

또 하나의 중요한 의존성은 libuv입니다. libuv는 C 라이브러리로 논블로킹 I/O 작업을 지원하는
모든 플랫폼에서 일관된 인터페이스로 추상화하는 데 사용됩니다. libuv는 파일 시스템, DNS, 네트워크,
자식 프로세스, 파이프, 신호 처리, 폴링, 스트리밍을 다루는 메커니즘을 제공하고 운영체제 수준에서
비동기로 처리될 수 없는 작업을 위한 스레드 풀도 포함하고 있습니다.

- [문서](http://docs.libuv.org/)

### http-parser

HTTP 파싱은 http-parser라는 경량 C 라이브러리가 처리합니다. 이는 시스템 호출이나 할당을 하려고
만들어진 것이 아니므로 요청당 아주 작은 메모리 공간만 차지합니다.

- [문서](https://github.com/joyent/http-parser/)

### c-ares

일부 비동기 DNS 요청을 위해서 Node.js는 c-ares라는 C 라이브러리를 사용합니다. c-ares는
JavaScript DNS 모듈로 `resolve()` 류의 함수들을 노출합니다. 코어의 다른 부분에서 사용하는
`lookup()` 함수는 libuv에서 스레드로 관리되는 `getaddrinfo(3)` 호출을 사용합니다. 이렇게 사용하는
이유는 c-ares가 /etc/hosts, /etc/resolv.conf, /etc/svc.conf는 지원하지만
mDNS 같은 것은 지원하지 않기 때문입니다.

- [문서](https://c-ares.haxx.se/docs.html)

### OpenSSL

OpenSSL은 `tls`와 `crypto` 모듈에서 광범위하게 사용되고 있습니다. OpenSSL은 현대 웹이
보안에서 사용하는 수많은 암호화 함수에 대한 검증된 구현체를 제공합니다.

- [문서](https://www.openssl.org/docs/)

### zlib

빠른 압축과 압축 해제를 하기 위해 Node.js는 산업 표준인 zlib 라이브러리를 사용하고 zlib은
gzip과 libpng를 사용한다고 알려져 있습니다. Node.js는 동기, 비동기, 스트리밍 압축과
압축 해제 인터페이스에 zlib을 사용합니다.

- [문서](https://www.zlib.net/manual.html)

## <!--tools-->도구

### npm

Node.js는 모든 것이 모듈화되어 있으므로 질 좋은 패키지 매니저가 필요해졌습니다. 이 목적 때문에
npm이 만들어졌습니다. npm이 모든 프로그래밍 생태계에서 커뮤니티가 만든 커다란 패키지 선택권을
제공해 주므로 Node.js를 빠르고 쉽게 만들 수 있습니다.

- [문서](https://docs.npmjs.com/)

### gyp

V8에서 파생된 Python 기반의 프로젝트 제너레이터인 gyp가 빌드 시스템을 다룹니다. gyp는 다수의
플랫폼에서 빌드 시스템을 사용하기 위한 프로젝트 파일을 생성할 수 있습니다. Node.js는 컴파일이
필요한 언어로 작성된 부분이 많이 있으므로(혹은 의존성 라이브러리에서) 빌드 시스템이 필요합니다.

- [문서](https://gyp.gsrc.io/docs/UserDocumentation.md)

### gtest

네이티브 코드는 Chromium의 gtest로 테스트할 수 있습니다. gtest로 C/C++를 시작하는
기존의 node 실행 파일 없이 C/C++를 테스트할 수 있습니다.

- [문서](https://code.google.com/p/googletest/wiki/V1_7_Documentation)
