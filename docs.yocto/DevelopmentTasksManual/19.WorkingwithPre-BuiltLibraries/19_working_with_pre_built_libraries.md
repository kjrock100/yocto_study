# 19 사전 빌드 라이브러리 작업 (Working with Pre-Built Libraries)

## 19.1 소개 (Introduction)

일부 라이브러리 공급업체는 소프트웨어의 소스 코드를 공개하지 않지만 사전 빌드된 바이너리를 공개합니다. 공유 라이브러리가 빌드될 때 버전이 지정되어야 하지만(배경은 [이 기사](https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)를 참조), 때때로 그렇지 않습니다.

요약하면, 버전이 지정된 라이브러리는 두 가지 조건을 충족해야 합니다:

1. 파일 이름에 버전이 추가되어야 합니다, 예: `libfoo.so.1.2.3`.
2. 라이브러리에 ELF 태그 `SONAME`이 라이브러리의 메이저 버전으로 설정되어야 합니다, 예: `libfoo.so.1`. `readelf -d filename | grep SONAME` 명령으로 이를 확인할 수 있습니다.

이 섹션에서는 버전이 지정된 사전 빌드 라이브러리와 버전이 지정되지 않은 사전 빌드 라이브러리를 다루는 방법을 보여줍니다.

## 19.2 버전이 지정된 라이브러리 (Versioned Libraries)

이 예제에서는 FT4222H USB I/O 칩을 위한 사전 빌드 라이브러리로 작업합니다. 라이브러리는 여러 대상 아키텍처 변형을 위해 빌드되고 다음과 같이 아카이브에 패키징됩니다:

```
├── build-arm-hisiv300
│   └── libft4222.so.1.4.4.44
├── build-arm-v5-sf
│   └── libft4222.so.1.4.4.44
├── build-arm-v6-hf
│   └── libft4222.so.1.4.4.44
├── build-arm-v7-hf
│   └── libft4222.so.1.4.4.44
├── build-arm-v8
│   └── libft4222.so.1.4.4.44
├── build-i386
│   └── libft4222.so.1.4.4.44
├── build-i486
│   └── libft4222.so.1.4.4.44
├── build-mips-eglibc-hf
│   └── libft4222.so.1.4.4.44
├── build-pentium
│   └── libft4222.so.1.4.4.44
├── build-x86_64
│   └── libft4222.so.1.4.4.44
├── examples
│   ├── get-version.c
│   ├── i2cm.c
│   ├── spim.c
│   └── spis.c
├── ftd2xx.h
├── install4222.sh
├── libft4222.h
├── ReadMe.txt
└── WinTypes.h
```

이러한 라이브러리를 시스템에서 사용하기 위한 레시피를 작성하려면:

- 공급업체는 독점 라이선스를 가질 가능성이 높으므로 레시피에서 `LICENSE_FLAGS`를 설정합니다.
- 공급업체는 라이브러리가 포함된 tarball을 제공하므로 `SRC_URI`를 적절히 설정합니다.
- `COMPATIBLE_HOST`를 설정하여 지원되지 않는 아키텍처에서는 레시피를 사용할 수 없도록 합니다. 다음 예제에서는 `x86` 아키텍처의 32비트 및 64비트 변형만 지원합니다.
- 공급업체가 버전이 지정된 라이브러리를 제공하므로 `utils`에서 `oe_soinstall`을 사용하여 공유 라이브러리를 설치하고 심볼릭 링크를 생성할 수 있습니다. 공급업체가 이를 수행하지 않으면 다음 섹션의 버전이 지정되지 않은 라이브러리 지침을 따라야 합니다.
- 공급업체가 Yocto Project 빌드의 `LDFLAGS`와 다른 `LDFLAGS`를 사용했을 가능성이 높으므로 `INSANE_SKIP`에 `ldflags`를 추가하여 해당 검사를 비활성화합니다.
- 공급업체는 일반적으로 디버깅 심볼 없이 릴리스 빌드를 제공합니다. 패키징 작업이 심볼을 제거하고 별도의 디버그 패키지에 추가하는 것을 방지하여 오류를 피합니다. 이는 아래에 표시된 `INHIBIT_` 플래그를 설정하여 수행됩니다.

완전한 레시피는 다음과 같습니다:

```
SUMMARY = "FTDI FT4222H Library"
SECTION = "libs"
LICENSE_FLAGS = "ftdi"
LICENSE = "CLOSED"

COMPATIBLE_HOST = "(i.86|x86_64).*-linux"

# Sources available in a .tgz file in .zip archive
# at https://ftdichip.com/wp-content/uploads/2021/01/libft4222-linux-1.4.4.44.zip
# Found on https://ftdichip.com/software-examples/ft4222h-software-examples/
# Since dealing with this particular type of archive is out of topic here,
# we use a local link.
SRC_URI = "file://libft4222-linux-${PV}.tgz"

S = "${UNPACKDIR}"

ARCH_DIR:x86-64 = "build-x86_64"
ARCH_DIR:i586 = "build-i386"
ARCH_DIR:i686 = "build-i386"

INSANE_SKIP:${PN} = "ldflags"
INHIBIT_PACKAGE_STRIP = "1"
INHIBIT_SYSROOT_STRIP = "1"
INHIBIT_PACKAGE_DEBUG_SPLIT = "1"

do_install () {
    install -m 0755 -d ${D}${libdir}
    oe_soinstall ${S}/${ARCH_DIR}/libft4222.so.${PV} ${D}${libdir}
    install -d ${D}${includedir}
    install -m 0755 ${S}/*.h ${D}${includedir}
}
```

사전 컴파일된 바이너리가 정적으로 링크되지 않고 다른 라이브러리에 대한 종속성을 가지고 있다면, `DEPENDS`에 해당 라이브러리를 추가하여 링킹을 검사하고 적절한 `RDEPENDS`를 자동으로 추가할 수 있습니다.

## 19.3 버전이 지정되지 않은 라이브러리 (Non-Versioned Libraries)

### 19.3.1 배경 설명 (Some Background)

리눅스 시스템의 라이브러리는 일반적으로 버전이 지정되어 동일한 라이브러리의 여러 버전을 설치할 수 있도록 하여 업그레이드와 이전 소프트웨어 지원을 용이하게 합니다. 예를 들어, 버전이 지정된 라이브러리에서 실제 라이브러리가 `libfoo.so.1.2`라고 불리고, `libfoo.so.1`이라는 심볼릭 링크가 `libfoo.so.1.2`를 가리키며, `libfoo.so`라는 심볼릭 링크가 `libfoo.so.1.2`를 가리킨다고 가정합니다. 이러한 조건이 주어지면 바이너리를 라이브러리에 연결할 때 일반적으로 버전이 지정되지 않은 파일 이름(`-lfoo`를 링커에)을 제공합니다. 그러나 링커는 심볼릭 링크를 따라 실제로 버전이 지정된 파일 이름에 연결합니다. 버전이 지정되지 않은 심볼릭 링크는 개발 시에만 사용됩니다. 결과적으로, 라이브러리는 헤더와 함께 개발 패키지 `${PN}-dev`에 패키징되며 실제 라이브러리와 버전이 지정된 심볼릭 링크는 `${PN}`에 있습니다. 버전이 지정된 라이브러리가 버전이 지정되지 않은 라이브러리보다 훨씬 더 일반적이기 때문에 기본 패키징 규칙은 버전이 지정된 라이브러리를 가정합니다.

### 19.3.2 Yocto 라이브러리 패키징 개요 (Yocto Library Packaging Overview)

따라서 버전이 지정되지 않은 라이브러리를 패키징하려면 레시피에서 약간의 작업이 필요합니다. 기본적으로 `libfoo.so`는 `${PN}-dev`에 패키징되며, 이는 `-dev` 패키지에 심볼릭 링크가 아닌 라이브러리가 있다는 QA 경고를 트리거하고, 동일한 레시피의 바이너리가 `${PN}-dev`의 라이브러리에 연결되어 더 많은 QA 경고를 트리거합니다. 이 문제를 해결하려면 버전이 지정되지 않은 라이브러리를 속한 `${PN}`에 패키징해야 합니다. `bitbake.conf`의 축약된 기본 `FILES` 변수는 다음과 같습니다:

```
SOLIBS = ".so.*"
SOLIBSDEV = ".so"
FILES:${PN} = "... ${libdir}/lib*${SOLIBS} ..."
FILES_SOLIBSDEV ?= "... ${libdir}/lib*${SOLIBSDEV} ..."
FILES:${PN}-dev = "... ${FILES_SOLIBSDEV} ..."
```

`SOLIBS`는 실제 공유 객체 라이브러리와 일치하는 패턴을 정의합니다. `SOLIBSDEV`는 개발 형식(버전이 지정되지 않은 심볼릭 링크)과 일치합니다. 이 두 변수는 `FILES:${PN}`과 `FILES:${PN}-dev`에 사용되어 실제 라이브러리를 `${PN}`에, 버전이 지정되지 않은 심볼릭 링크를 `${PN}-dev`에 넣습니다. 버전이 지정되지 않은 라이브러리를 패키징하려면 레시피에서 변수를 다음과 같이 수정해야 합니다:

```
SOLIBS = ".so"
FILES_SOLIBSDEV = ""
```

수정 사항으로 인해 `.so` 파일이 실제 라이브러리가 되고 `FILES_SOLIBSDEV`가 설정 해제되어 `${PN}-dev`에 라이브러리가 패키징되지 않습니다. `PACKAGES`가 변경되지 않는 한 `${PN}-dev`가 `${PN}`보다 먼저 파일을 수집하므로 이러한 변경이 필요합니다. `${PN}-dev`는 `${PN}`에 원하는 파일을 수집해서는 안 됩니다.

마지막으로, `dlopen()`을 사용하여 빌드 시가 아닌 런타임에 연결되는 로드 가능한 모듈인 본질적으로 버전이 지정되지 않은 라이브러리는 일반적으로 개인 디렉토리에 설치되어야 합니다. 그러나 `${libdir}`에 설치되면 모듈을 버전이 지정되지 않은 라이브러리로 취급할 수 있습니다.

### 19.3.3 예제 (Example)

아래 예제는 `libfoo.so`라는 버전이 지정되지 않은 x86-64 사전 빌드 라이브러리를 설치합니다. `COMPATIBLE_HOST` 변수는 레시피를 x86-64 아키텍처로 제한하고 `INSANE_SKIP`, `INHIBIT_PACKAGE_STRIP` 및 `INHIBIT_SYSROOT_STRIP` 변수는 위의 버전이 지정된 라이브러리 예제와 같이 모두 설정됩니다. "마법"은 위에서 설명한 대로 `SOLIBS` 및 `FILES_SOLIBSDEV` 변수를 설정하는 것입니다:

```
SUMMARY = "libfoo sample recipe"
SECTION = "libs"
LICENSE = "CLOSED"

SRC_URI = "file://libfoo.so"

COMPATIBLE_HOST = "x86_64.*-linux"

INSANE_SKIP:${PN} = "ldflags"
INHIBIT_PACKAGE_STRIP = "1"
INHIBIT_SYSROOT_STRIP = "1"
SOLIBS = ".so"
FILES_SOLIBSDEV = ""

do_install () {
    install -d ${D}${libdir}
    install -m 0755 ${UNPACKDIR}/libfoo.so ${D}${libdir}
}
```
