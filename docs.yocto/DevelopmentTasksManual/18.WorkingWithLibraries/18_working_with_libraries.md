# 18 라이브러리 작업 (Working With Libraries)

Yocto Project에서 라이브러리를 다루는 방법, 특히 정적 라이브러리 포함, 멀티립(Multilib)을 사용한 다중 버전 라이브러리 결합, 그리고 동일 라이브러리의 다중 버전 설치에 대해 설명합니다.

## 18.1 정적 라이브러리 파일 포함 (Including Static Library Files)

기본적으로 Yocto Project 빌드 시스템은 정적 라이브러리(`.a` 파일)를 생성하더라도 최종 이미지나 SDK에 자동으로 포함시키지 않습니다. 이는 정적 라이브러리가 주로 `-staticdev` 패키지로 분리되기 때문입니다.

* **이미지에 포함하기:**
    이미지에 특정 라이브러리의 정적 버전을 포함하려면 `IMAGE_INSTALL` 변수에 해당 라이브러리의 `-staticdev` 패키지를 명시적으로 추가해야 합니다.

    ```bash
    IMAGE_INSTALL:append = " libexample-staticdev"
    ```

* **SDK에 포함하기:**
    SDK 생성 시 정적 라이브러리를 포함하려면 `TOOLCHAIN_TARGET_TASK`에 추가하거나, `local.conf` 파일에서 `SDKIMAGE_FEATURES` 변수를 사용하여 모든 정적 라이브러리를 포함하도록 설정할 수 있습니다.

    ```bash
    # 특정 패키지만 추가
    TOOLCHAIN_TARGET_TASK:append = " libexample-staticdev"

    # 또는 모든 패키지의 정적 라이브러리 포함
    SDKIMAGE_FEATURES:append = " staticdev-pkgs"
    ```

## 18.2 여러 버전의 라이브러리 파일을 하나의 이미지에 결합 (Combining Multiple Versions of Library Files into One Image)

32비트와 64비트 라이브러리를 동시에 지원해야 하는 경우와 같이, 서로 다른 ABI(Application Binary Interface)를 가진 라이브러리를 하나의 이미지에 포함해야 할 때 **Multilib** 기능을 사용합니다.

### 18.2.1 Multilib 사용 준비 (Preparing to Use Multilib)

Multilib을 사용하려면 `conf/local.conf` 파일에 `MULTILIBS` 변수와 해당 멀티립에 대한 `DEFAULTTUNE`을 정의해야 합니다.

예를 들어, 64비트 시스템(`x86-64`)에서 32비트(`x86`) 라이브러리를 지원하려면 다음과 같이 설정합니다:

```bash
# conf/local.conf 예시
MACHINE = "qemux86-64"
require conf/multilib.conf
MULTILIBS = "multilib:lib32"
DEFAULTTUNE:virtclass-multilib-lib32 = "x86"
```

### 18.2.2 Multilib 사용 (Using Multilib)

설정이 완료되면, `IMAGE_INSTALL` 변수에 `lib32-` 접두사를 붙여 패키지를 추가할 수 있습니다.

```bash
IMAGE_INSTALL:append = " lib32-bash lib32-glib-2.0"
```

이렇게 하면 기본 64비트 시스템에 32비트 버전의 bash와 glib 라이브러리가 함께 설치됩니다.

### 18.2.3 추가 구현 세부 사항 (Additional Implementation Details)

* **패키지 관리자:** Multilib 환경에서는 RPM 패키지 관리자가 가장 잘 동작합니다.
* **의존성 해결:** Multilib 패키지의 의존성은 자동으로 `lib32-` 접두사가 붙은 해당 아키텍처의 패키지로 해결됩니다.

## 18.3 동일 라이브러리의 여러 버전 설치 (Installing Multiple Versions of the Same Library)

일반적으로 리눅스 패키지 관리 시스템은 동일한 패키지의 다른 버전을 동시에 설치하는 것을 허용하지 않습니다 (예: `libfoo 1.0`과 `libfoo 2.0`).

하지만 라이브러리의 경우 `SONAME`이 다르다면(예: `libfoo.so.1`, `libfoo.so.2`) 공존이 가능할 수 있습니다. Yocto Project에서 이를 구현하려면, 각 버전을 별도의 레시피 파일로 관리하고 패키지 이름(PN)을 다르게 지정해야 합니다 (예: `libfoo1`, `libfoo2`).

이렇게 패키지 이름이 분리되면 `IMAGE_INSTALL`을 통해 두 버전을 모두 이미지에 설치할 수 있습니다.
