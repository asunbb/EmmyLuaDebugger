cmake_minimum_required(VERSION 3.2)

# 是否静态链接到编译单元 lua 库
# 1 若静态链接 lua 库，目标编译单元生成为 static
# 2 若动态加载 lua 库函数，目标编译单元生成为 shared
# 3 ios 和 android 平台，目标编译单元生成为 static
option(EMMY_USE_LUA_SOURCE "Build with lua source" ON)

set(EMMY_CORE_NAME "emmy_core")
set(EMMY_HOOK_NAME "emmy_hook")

project(${EMMY_CORE_NAME})

set(LINK_LIBRARIES uv_a)

# 源码
file(GLOB SRC_LIST
    *.cpp proto/*.cpp
    api/*.cpp
)
file(GLOB HOOK_LIST
    dllmain.cpp hook/*.h hook/*.cpp
    hook/libpe/*.h hook/libpe/*.cpp
    api/*.cpp
)

# 头文件
file(GLOB HEADER_LIST
    *.h proto/*.h
    api/lua_api.h
)
file(GLOB SHARED_LIST
    ../Shared/*.h
    ../Shared/*.cpp
)

# rapid json
add_definitions(-DRAPIDJSON_HAS_STDSTRING)

# 源文件配置，各平台差异
if (ANDROID OR IOS OR (WIN32 AND EMMY_USE_LUA_SOURCE))
    # 静态链接 lua 库的情况下不再需要动态加载函数
    remove_item_from_var(SRC_LIST ".+lua_api_loader.cpp")
endif ()

include_directories(
    ../Shared
    ../third-party/libuv-1.29.0/include
    ../third-party/rapidjson-1.1.0/include
    ../third-party/EasyHook/EasyHookDll
)

source_group_by_dir(${CMAKE_CURRENT_SOURCE_DIR} SRC_LIST)
source_group_by_dir(${CMAKE_CURRENT_SOURCE_DIR} HEADER_LIST)
source_group_by_dir(${CMAKE_CURRENT_SOURCE_DIR} HOOK_LIST)

# 各平台生成差异
if (WIN32)
    # emmy_core
    if (EMMY_USE_LUA_SOURCE)
        # 若是静态链接 lua 库则只能生成静态 lib，lua 库最终的 link 依赖由上层调用代码维护
        add_library(${EMMY_CORE_NAME} STATIC ${SRC_LIST} ${HEADER_LIST})
        target_compile_definitions(${EMMY_CORE_NAME} PRIVATE EMMY_USE_LUA_SOURCE)
    else ()
        add_library(${EMMY_CORE_NAME} SHARED ${SRC_LIST} ${HEADER_LIST})
    endif ()
    add_dependencies(${EMMY_CORE_NAME} ${LINK_LIBRARIES})
    target_link_libraries(${EMMY_CORE_NAME} ${LINK_LIBRARIES})

    # emmy_hook
    add_library(${EMMY_HOOK_NAME} SHARED ${SRC_LIST} ${HEADER_LIST} ${HOOK_LIST} ${SHARED_LIST})
    # emmy_hook 动态加载 lua 库进行使用
    list(APPEND LINK_LIBRARIES EasyHook)
    target_compile_definitions(${EMMY_HOOK_NAME} PRIVATE EMMY_BUILD_AS_HOOK)
    add_dependencies(${EMMY_HOOK_NAME} ${LINK_LIBRARIES})
    target_link_libraries(${EMMY_HOOK_NAME} ${LINK_LIBRARIES})

    install(
        TARGETS ${EMMY_CORE_NAME} ${EMMY_HOOK_NAME}
        LIBRARY DESTINATION bin
        RUNTIME DESTINATION bin
    )
elseif (IOS)
    # IOS 环境下 emmy_core 只编译成静态库
    add_library(${EMMY_CORE_NAME} STATIC ${SRC_LIST} ${HEADER_LIST})
    target_compile_definitions(${EMMY_CORE_NAME} PRIVATE EMMY_USE_LUA_SOURCE)
    add_dependencies(${EMMY_CORE_NAME} ${LINK_LIBRARIES})
    target_link_libraries(${EMMY_CORE_NAME} ${LINK_LIBRARIES})
elseif (ANDROID)
    # ANDROID 环境下 emmy_core 只编译成静态库
    add_library(${EMMY_CORE_NAME} STATIC ${SRC_LIST} ${HEADER_LIST})
    target_compile_definitions(${EMMY_CORE_NAME} PRIVATE EMMY_USE_LUA_SOURCE)
    add_dependencies(${EMMY_CORE_NAME} ${LINK_LIBRARIES})
    target_link_libraries(${EMMY_CORE_NAME} ${LINK_LIBRARIES})
    install(
        TARGETS ${EMMY_CORE_NAME}
        LIBRARY DESTINATION bin
        RUNTIME DESTINATION bin
    )
endif ()

# 各平台属性设置
if (IOS)
    set_xcode_property(${EMMY_CORE_NAME} IPHONEOS_DEPLOYMENT_TARGET 9.0)
endif ()
