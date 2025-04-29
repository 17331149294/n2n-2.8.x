# 设置 cmake 最低版本
cmake_minimum_required(VERSION 2.6)

# 设置项目名称
project(n2n)

# 引入检查函数模块
include(CheckFunctionExists)

# 引入 ExternalProject 模块，用于自动拉取第三方库
include(ExternalProject)

# 打开详细编译输出
SET(CMAKE_VERBOSE_MAKEFILE ON)

# 设置 n2n 版本信息
set(N2N_VERSION "2.8.x")
set(N2N_OSNAME ${CMAKE_SYSTEM})

# 检测是否存在 git
execute_process(
    COMMAND git --version
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_EXIST
)

# 如果存在 git，则获取 git 版本号信息
if (GIT_EXIST)
    execute_process(
        COMMAND git rev-list --count HEAD
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_REV
    )
    execute_process(
        COMMAND git rev-parse --short HEAD
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_ID
    )
    string(REGEX REPLACE "\n$" "" GIT_REV "${GIT_REV}")
    string(REGEX REPLACE "\n$" "" GIT_ID "${GIT_ID}")
    set(N2N_VERSION "${N2N_VERSION}-多线程")
    MESSAGE(STATUS "Build from git rev: ${N2N_VERSION}")
endif (GIT_EXIST)

# 添加预处理宏
add_definitions(-DCMAKE_BUILD)
add_definitions(-DGIT_RELEASE="${N2N_VERSION}" -DPACKAGE_VERSION="${N2N_VERSION}" -DPACKAGE_OSNAME="${N2N_OSNAME}")
add_definitions(-DN2N_VERSION="${N2N_VERSION}" -DN2N_OSNAME="${N2N_OSNAME}")

# 设置是否生成共享库
OPTION(BUILD_SHARED_LIBS "BUILD Shared Library" OFF)

# N2N 是否启用 AES 加密
OPTION(N2N_OPTION_AES "USE AES" ON)

if(NOT DEFINED N2N_OPTION_AES)
    set(N2N_OPTION_AES ON)
endif(NOT DEFINED N2N_OPTION_AES)

# 如果启用了 AES，则查找 OpenSSL
if(N2N_OPTION_AES)
    find_package(OpenSSL QUIET)
    if(NOT OPENSSL_FOUND)
        message(FATAL_ERROR "错误：未找到 OpenSSL 库，无法启用 AES 加密支持，请先安装 OpenSSL。")
    else()
        include_directories(${OPENSSL_INCLUDE_DIR})
        add_definitions(-DN2N_HAVE_AES)
    endif(NOT OPENSSL_FOUND)
endif(N2N_OPTION_AES)

# 如果未指定构建类型，默认为 None
if(NOT DEFINED CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE None)
endif(NOT DEFINED CMAKE_BUILD_TYPE)

# 针对 Unix 平台的编译器参数
if (DEFINED UNIX)
    set(CMAKE_C_FLAGS "-Wall -Wshadow -Wpointer-arith -Wmissing-declarations -Wnested-externs -w")
    set(CMAKE_CXX_FLAGS "-Wall -Wshadow -Wpointer-arith -Wmissing-declarations -Wnested-externs -w")
    set(CMAKE_C_FLAGS_DEBUG "-g")
    set(CMAKE_CXX_FLAGS_DEBUG "-g")
    set(CMAKE_C_FLAGS_RELEASE "-Os -DNDEBUG -s -w -ffunction-sections -fdata-sections")
    set(CMAKE_CXX_FLAGS_RELEASE "-Os -DNDEBUG -s -w -ffunction-sections -fdata-sections")
    set(CMAKE_EXE_LINKER_FLAGS_RELEASE "-Wl,--gc-sections")
endif(DEFINED UNIX)

# 包含当前目录和 include 目录
INCLUDE_DIRECTORIES(.)
INCLUDE_DIRECTORIES(include)

# 针对 Windows 平台，添加额外头文件路径
if(DEFINED WIN32)
    INCLUDE_DIRECTORIES(win32)
endif(DEFINED WIN32)

# ========== 这里开始加入 c-ares 自动下载编译 ==========

# 配置 c-ares 仓库信息
set(CARES_REPO "https://github.com/c-ares/c-ares.git")  # c-ares 仓库地址
set(CARES_TAG "v1.34.5")                           # 指定 c-ares 版本 tag
set(CARES_INSTALL_DIR ${CMAKE_BINARY_DIR}/thirdparty/c-ares-install)  # 安装目录

# 使用 ExternalProject 自动下载、编译、安装 c-ares
ExternalProject_Add(c-ares
    PREFIX ${CMAKE_BINARY_DIR}/thirdparty/c-ares
    GIT_REPOSITORY ${CARES_REPO}
    GIT_TAG ${CARES_TAG}
    UPDATE_COMMAND ""  # 不自动更新
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -DCMAKE_INSTALL_PREFIX=${CARES_INSTALL_DIR} -DCARES_STATIC=ON -DCARES_SHARED=OFF <SOURCE_DIR>
    BUILD_COMMAND $(MAKE)
    INSTALL_COMMAND $(MAKE) install
    LOG_DOWNLOAD ON
    LOG_CONFIGURE ON
    LOG_BUILD ON
    LOG_INSTALL ON
)

# 包含 c-ares 的头文件目录
include_directories(${CARES_INSTALL_DIR}/include)

# 添加 c-ares 的静态库目录
link_directories(${CARES_INSTALL_DIR}/lib)

# ========== c-ares 配置结束 ==========


# 搜索第三方 liblfds 源码
file(GLOB_RECURSE LFDS_ALL_SRC "thirdparty/liblfds711/src/*.c")

# 添加 n2n 静态库
add_library(n2n STATIC
    src/n2n.c
    src/edge_utils.c
    src/sn_utils.c
    src/wire.c
    src/minilzo.c
    src/twofish.c
    src/transform_null.c
    src/transform_tf.c
    src/transform_aes.c
    src/transform_cc20.c
    src/transform_speck.c
    src/speck.c
    src/random_numbers.c
    src/pearson.c
    src/header_encryption.c
    src/tuntap_freebsd.c
    src/tuntap_netbsd.c
    src/tuntap_linux.c
    src/tuntap_osx.c
    "${LFDS_ALL_SRC}"
)

# 如果是 Windows，额外加 edge_utils_win32
if(DEFINED WIN32)
    add_library(edge_utils_win32 src/edge_utils_win32.c)
    add_subdirectory(win32)
    target_link_libraries(n2n edge_utils_win32 n2n_win32)
endif(DEFINED WIN32)

# 如果启用了 AES，加上 OpenSSL 库
if(N2N_OPTION_AES)
    target_link_libraries(n2n ${OPENSSL_LIBRARIES})
endif(N2N_OPTION_AES)

# 链接 pthread 和 dl
target_link_libraries(n2n pthread dl)

# 添加 c-ares 依赖关系，确保 n2n 编译前 c-ares 已经拉取好
add_dependencies(n2n c-ares)

# 链接 c-ares 静态库
target_link_libraries(n2n cares)

# 创建 edge 可执行文件
add_executable(edge src/edge.c)
target_link_libraries(edge n2n)

# 创建 supernode 可执行文件
add_executable(supernode src/sn.c)
target_link_libraries(supernode n2n)

# 创建示例程序
add_executable(example_edge_embed_quick_edge_init src/example_edge_embed_quick_edge_init.c)
target_link_libraries(example_edge_embed_quick_edge_init n2n)

add_executable(example_edge_embed src/example_edge_embed.c)
target_link_libraries(example_edge_embed n2n)

add_executable(example_sn_embed src/example_sn_embed.c)
target_link_libraries(example_sn_embed n2n)

# 非 Windows 平台，查找 libcap 库
if(NOT DEFINED WIN32)
    find_library(CAP_LIB cap)
    if(CAP_LIB)
        target_link_libraries(edge cap)
        set(CMAKE_REQUIRED_LIBRARIES ${CAP_LIB})
        ADD_DEFINITIONS("-DHAVE_LIBCAP")
    endif()
endif(NOT DEFINED WIN32)

# 安装 edge 和 supernode 到系统目录
install(TARGETS edge supernode
    RUNTIME DESTINATION sbin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

# 构建工具：n2n-benchmark
add_executable(n2n-benchmark tools/benchmark.c)
target_link_libraries(n2n-benchmark n2n)

# 查找 pcap 库
find_library(PCAP_LIB pcap)
if(PCAP_LIB)
    add_executable(n2n-decode tools/n2n_decode.c)
    target_link_libraries(n2n-decode n2n pcap)
    install(TARGETS n2n-decode RUNTIME DESTINATION bin)
    set(CMAKE_REQUIRED_LIBRARIES ${PCAP_LIB})
    check_function_exists(pcap_set_immediate_mode HAVE_PCAP_IMMEDIATE_MODE)
    if(HAVE_PCAP_IMMEDIATE_MODE)
        add_definitions("-DHAVE_PCAP_IMMEDIATE_MODE")
    endif()
endif(PCAP_LIB)

# 安装 n2n-benchmark 工具
install(TARGETS n2n-benchmark RUNTIME DESTINATION bin)
