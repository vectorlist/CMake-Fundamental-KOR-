# FetchContent
C++ 라이브러리가 아닌 C++ 실행 프로젝트를 작업할경우 패키지를 가져오는 것이 무거울수 있음
그냥 라이브러리 소스를 가져와 기존 프로젝트에 병합하여 컴파일만 하는경우
FetchContent 모듈이 좋습니다


FetchContent는 다운로드 또는 "페치" 종속성을 매우 간단하게 만드는 CMake 모듈입니다. FetchContent_Declare()를 호출하여 
소스가 어디에 있는지 CMake에 알려준 다음 FetchContent_MakeAvailable()을 사용하여 하위 프로젝트로 포함하기만 하면 됩니다.
그러면 프로젝트가 자동으로 다운로드되고 타깃을 사용할 수 있게 되어 필요에 따라 링크하고 빌드할 수 있습니다.

FetchContent는 git 리포지토리를 복제할 수 있습니다,

라이브러리 작성자인 경우 테스트와 같은 개발자 대상을 숨기고, 다운스트림과 관련된 소스 파일만 포함하는 zip 아카이브를 제공하고,
GitHub 작업을 사용하여 자동으로 생성하는 등 FetchContent를 사용하는 최종 사용자의 경험을 개선하도록 CMake 프로젝트를 구성할 수 있는 방법이 있습니다.

```cmake
include(FetchContent) # once in the project to include the module

FetchContent_Declare(googletest
                     GIT_REPOSITORY https://github.com/google/googletest.git
                     GIT_TAG        703bd9caab50b139428cea1aaff9974ebee5742e # release-1.10.0)
FetchContent_MakeAvailable(googletest)
```
or
```cmake
FetchContent_Declare(lexy URL https://lexy.foonathan.net/download/lexy-src.zip)
FetchContent_MakeAvailable(lexy)
```

# FetchContent 셋업 디자인

FetchContent를 통해 프로젝트를 사용하는 경우 CMake는 자동으로 add_subdirectory()를 호출합니다. 이렇게 하면 프로젝트의 모든 대상을 부모에서 사용할 수 있으므로 해당 대상에 링크하여 사용할 수 있습니다.

그러나 여기에는 유닛 테스트, 문서 빌더 등과 같이 다운스트림 소비자에게 유용하지 않은 타깃도 포함됩니다. 결정적으로, 여기에는 이러한 대상의 종속성이 포함됩니다. 라이브러리를 사용할 때 CMake가 해당 라이브러리 테스트 프레임워크를 다운로드하지 않기를 원합니다! 따라서 하위 디렉터리로 사용되지 않을 때만 해당 헬퍼 대상을 노출하여 이를 방지하는 것이 좋습니다.

라이브러리의 루트 CMakeLists.txt에서 CMAKE_CURRENT_SOURCE_DIR과 CMAKE_SOURCE_DIR을 비교하면 소스 트리의 실제 루트인 경우에만 동일하다는 것을 감지할 수 있습니다. 따라서 테스트 대상은 그렇지 않은 경우에만 정의합니다.

```cmake
project(my_project LANGUAGES CXX)

# define build options useful for all use
…

# define the library targets
add_subdirectory(src)

if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    # We're in the root, define additional targets for developers.
    option(MY_PROJECT_BUILD_EXAMPLES   "whether or not examples should be built" ON)
    option(MY_PROJECT_BUILD_TESTS      "whether or not tests should be built" ON)

    if(MY_PROJECT_BUILD_EXAMPLES)
        add_subdirectory(examples)
    endif()
    if(MY_PROJECT_BUILD_TESTS)
        enable_testing()
        add_subdirectory(tests)
    endif()

    …
endif()
```

이러한 방식으로 CMakeLists.txt를 분기하면 다운스트림 소비자와 라이브러리 개발자를 위해 서로 다른 CMake 버전을 사용할 수도 있습니다. 예를 들어, lexy를 사용하려면 3.8 버전이 필요하지만 개발하려면 3.18 버전이 필요합니다. 이 작업은 if() 블록 내에서 cmake_minimum_required(버전 3.18)를 호출하면 됩니다.

# 어떤것을 다운로드 해야할까

FetchContent_Declare는 다양한 소스에서 프로젝트를 다운로드할 수 있지만 모든 소스가 같은 시간이 걸리는 것은 아닙니다. 적어도 GitHub에서는 압축된 소스를 다운로드하고 압축을 푸는 것보다 git 리포지토리를 복제하는 것이 훨씬 더 오래 걸립니다.
```cmake
# slow
FetchContent_Declare(lexy GIT_REPOSITORY https://github.com/foonathan/lexy)
FetchContent_MakeAvailable(lexy)
# fast
FetchContent_Declare(lexy URL https://github.com/foonathan/lexy/archive/refs/heads/main.zip)
FetchContent_MakeAvailable(lexy)
```
하지만 모든 소스를 다운로드하는 것은 너무 많은 양일 수 있습니다. 예를 들어, lexy의 경우 많은 테스트, 예제, 벤치마크가 포함되어 있지만 실제로 다운스트림 사용자로서 프로젝트를 사용하는 데는 필요하지 않습니다. 특히 위에서 설명한 대로 하위 프로젝트로 사용하면 대부분의 기능이 비활성화되기 때문에 더욱 그렇습니다.

따라서 대신 lexy의 경우 헤더 파일, 라이브러리의 소스 파일, 최상위 CMakeLists.txt 등 필요한 파일만 포함된 사전 패키징된 zip 파일을 다운로드해야 합니다. 이렇게 하면 불필요한 파일에 대역폭이나 디스크 공간을 낭비하지 않습니다.

```cmake
# really fast
FetchContent_Declare(lexy URL https://lexy.foonathan.net/download/lexy-src.zip)
FetchContent_MakeAvailable(lexy)
```
특히 이 프로세스를 완전히 자동화할 수 있으므로 FetchContent와 함께 사용하기 위한 라이브러리를 유지 관리하고 있다면 그렇게 하는 것이 좋습니다.

# 패키지 소스 파일 자동 생성 및 퍼블리쉬
이를 위해서는 먼저 패키지를 생성할 사용자 정의 CMake 대상을 정의해야 합니다.

```cmake
set(package_files include/ src/ CMakeLists.txt LICENSE)
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-src.zip
    COMMAND ${CMAKE_COMMAND} -E tar c ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-src.zip --format=zip -- ${package_files}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${package_files})
add_custom_target(${PROJECT_NAME}_package DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-src.zip)
```
이 작업은 세 단계로 이루어집니다.

* 패키지에 포함해야 하는 모든 파일과 폴더의 목록을 정의합니다. 여기에는 항상 루트 CMakeLists.txt와 라이브러리의 포함 및 소스 파일이 포함되어야 합니다.

* zip 파일을 생성하는 사용자 지정 명령을 정의합니다. 이 명령은 아카이브를 생성하기 위해 cmake -E tar를 호출해야 합니다. 이 명령은 패키지 파일 목록에 대한 종속성을 가지므로 파일이 변경되면 CMake가 zip 아카이브를 다시 빌드해야 한다는 것을 알 수 있습니다.

* 사용자 지정 대상을 정의합니다. 이를 빌드하기 위해(그 자체로는 아무 작업도 수행하지 않음) CMake에 zip 파일이 필요하다고 지시했습니다. 따라서 타깃을 빌드하면 사용자 지정 명령이 실행되고 아카이브가 생성됩니다.

**!물론 이 대상은 프로젝트가 하위 디렉터리로 사용되지 않는 경우에만 정의됩니다!**

FetchContent는 종속성을 관리하는 매우 편리한 방법입니다. 하지만 라이브러리 작성자는 최종 사용자가 더 쉽게 사용할 수 있도록 몇 가지 작업을 수행할 수 있습니다:

프로젝트가 하위 디렉터리로 포함될 때 최소한의 대상만 정의하세요.
사용자가 전체 리포지토리 대신 다운로드할 수 있는 최소한의 압축된 소스 아카이브를 제공하세요.
각 릴리스에 대한 아카이브를 자동으로 만들려면 GitHub 작업을 사용하세요.
