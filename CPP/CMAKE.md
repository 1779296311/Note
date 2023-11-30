# 最底层逻辑
1.  头文件的作用，其实是函数的声明， 直接在比的文件里面写声明也可， 但是那样代码重复的太多。头文件不仅可以减少重复代码， 还可以在修改时，该得少；

2. 减轻依赖 - morden cmake

    1. target_include_directories(add PRIVATE ../common/) 
    private 代表 后面的文件只对当前目标生效， 否则还会对依赖当前目标的文件生效；

    1. target_include_directories(common INTERFACE ./)
        -- INTERFACE 代表 ./目录是我提供给 别的依赖 commom 的文件使用的； - 别人 cmakefile 就不用 添加 
            target_include_directories(add PRIVATE ../commom/)

3. find_package(name REQUIRED) - 从环境变量i找 - nameConfig.cmake

4. set(name value) - 设置变量 - 全是字符串 - ${name} 使用
    set(CMAKE_CXX_STANDARD 14)
    set(EXECUTEABLE_OUTPUT_PATH path) - 设置输出路径

5. 搜索文件
    1. file(GLOB/GLOB_RECURSE files path/*.cpp) - GLOB_RECURSE - 递归搜索

6. add_library(common OBJECT/STATIC commom.cpp)
    1. 之前用的都是 STATIC 不能单独使用；
    2. 用OBJECT ，可以单独拿出去使用；

所以可以改成
```cpp
add_library(add STATIC add.cpp)
target_include_directories(add INTERFACE ./)
target_link_libraries(add PRIVATE common)

//修改后
//先生成 object 
add_library(add_object OBJECT add.cpp)
target_link_libraries(add_object PRIVATE common)
//生成库文件
add_library(add STASTIC $<TARGET_OBJECT:add_object> $<TARGET_OBJECT:common_object>)
target_include_directories(add PUBLIC ./)
```