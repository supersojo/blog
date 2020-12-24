---
title: vs code在c/c++开发中满足需求吗
date: 2020-12-23 22:13:12
tags:
- vs code
- cmake
- boost
categories:
- tools
---

vs code作为一个编辑器，提供不错的外观和出色的性能，大量的扩展插件使其成为出色的IDE。

## Installing vs code

## Install c/c++ extension

## Install cmake/cmake tool extensions

## Using vs code

首先使用c/c++扩展写个测试代码。

```
c_cpp_properties.json
{
    "configurations": [
        {
            "name": "Win32",
            "includePath": [
                "${workspaceFolder}/**",
                "d:\\boost\\include\\boost-1_75"
            ],
            "defines": [
                "_DEBUG",
                "UNICODE",
                "_UNICODE"
            ],
            "windowsSdkVersion": "10.0.18362.0",
            "compilerPath": "C:/Program Files (x86)/Microsoft Visual Studio/2019/BuildTools/VC/Tools/MSVC/14.28.29333/bin/Hostx64/x64/cl.exe",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "msvc-x64"
        }
    ],
    "version": 4
}
```
上面把boost配置includepath，这样vs code可以智能的完成代码补全和代码提示，非常高效。

```
tasks.json
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "cppbuild",
			"label": "C/C++: cl.exe build active file",
			"command": "cl.exe",
			"args": [
				"/Zi",
				"/EHsc",
				"/ID:\\boost\\include\\boost-1_75",
				"/Fe:",
				"${fileDirname}\\${fileBasenameNoExtension}.exe",
				"${file}"
			],
			"options": {
				"cwd": "${workspaceFolder}"
			},
			"problemMatcher": [
				"$msCompile"
			],
			"group": {
				"kind": "build",
				"isDefault": true
			},
			"detail": "compiler: cl.exe"
		}
	]
}
```
上面我们添加boost头文件，或者lib文件，可以方便的生成可执行文件。

```
launch.json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(Windows) 启动",
            "type": "cppvsdbg",
            "request": "launch",
            "program": "${workspaceFolder}/${fileBasenameNoExtension}.exe",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false
        }
    ]
}
```
上面文件我们调试这个可执行文件的配置。

综上c/c++插件可以满足单个c++源文件的构建，在写测试代码时非常方便，但是多个文件的情况难以胜任。

## Using cmake扩展

```
CMakeLists.txt

cmake_minimum_required(VERSION 3.0.0)
project(hello VERSION 0.1.0)

include(CTest)
enable_testing()

# 首先定义target
add_executable(hello foo.cpp main.cpp)

# 针对这个target设置include
target_include_directories(hello PRIVATE "d:\\boost\\include\\boost-1_75")

set(CPACK_PROJECT_NAME ${PROJECT_NAME})
set(CPACK_PROJECT_VERSION ${PROJECT_VERSION})
include(CPack)
```
该扩展可以帮助生成基本的cmake配置文件，在此基础上可配置多个文件的编译生成。

## boost例子
```
#include <iostream>
#include <boost/array.hpp>

extern void foo();

int main() {



    boost::array<int,10> a;

    a[0] = 1;

    int i = std::get<0,int,10>(a);
    
    std::cout<<i<<std::endl;

    std::cout<<"hello from c++!"<<std::endl;

    foo();
    
    return 0;
}
```

## References
