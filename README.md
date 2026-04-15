## Mars编译

### 1. 环境

使用`wsl2`+`Ubuntu`+`python3`+`ndk27d`编译 mars (版本: `6aa5b56`)

#### 1.1 安装依赖

```bash
sudo apt update
sudo apt install -y git python3 make cmake unzip
```

#### 1.2 设置ndk

1. 下载

```bash
# 进入一个文件夹下载
wget https://dl.google.com/android/repository/android-ndk-r27d-linux.zip
```

2. 解压

```bash
unzip android-ndk-r27d-linux.zip
```

3. 设置环境变量

```bash
# 添加至`~/.bashrc`文件末尾
NDK_ROOT=~/ws/android-ndk-r27d
PATH=$NDK_ROOT:$PATH
```

或者使用命令方式:

```bash
printf 'export NDK_ROOT=~/ws/android-ndk-r27d\nexport PATH=$NDK_ROOT:$PATH\n' >> ~/.bashrc
```

4. 并刷新

```bash
source ~/.bashrc
```

5. 验证

```bash
echo $NDK_ROOT
```

### 2. 修改源码

#### 2.1 下载源码

一般使用主分支即可;

#### 2.2 适配NDK27

修改`build_android.py`的`ANDROID_BUILD_CMD`语句:

```python
ANDROID_BUILD_CMD = 'cmake "%s" %s -DANDROID_ABI="%s" ' \
                    '-DCMAKE_BUILD_TYPE=Release ' \
                    '-DCMAKE_TOOLCHAIN_FILE=%s/build/cmake/android.toolchain.cmake ' \
                    '-DANDROID_NDK=%s -DANDROID_PLATFORM=android-21 ' \
                    '-DANDROID_STL="c++_shared" ' \
                    # 增加, 禁用因 NDK 版本过高导致累积错误过多而导致的编译失败
                    '-DCMAKE_CXX_FLAGS="-w" ' \
                    # 增加, 设置16k页面对齐, --undefined-version允许链接器引用未定义的符号版本而不报错
                    '-DCMAKE_SHARED_LINKER_FLAGS="-Wl,-z,max-page-size=16384 -Wl,--undefined-version" ' \
                    '&& cmake --build . %s --config Release -- -j8'

# 修改为NDK27的地址 
ANDROID_STRIP_FILE = {
    'armeabi': NDK_ROOT + '/toolchains/llvm/prebuilt/%s/bin/llvm-strip',
    'armeabi-v7a': NDK_ROOT + '/toolchains/llvm/prebuilt/%s/bin/llvm-strip',
    'x86': NDK_ROOT + '/toolchains/llvm/prebuilt/%s/bin/llvm-strip',
    'arm64-v8a': NDK_ROOT + '/toolchains/llvm/prebuilt/%s/bin/llvm-strip',
    'x86_64': NDK_ROOT + '/toolchains/llvm/prebuilt/%s/bin/llvm-strip',
}

# 修改为ndk27的地址， 固定为linux-x86_64， 如果使用不同的系统需要修改，
ANDROID_STL_FILE = {
    'armeabi-v7a': NDK_ROOT + '/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/arm-linux-androideabi/libc++_shared.so',
    'arm64-v8a': NDK_ROOT + '/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android/libc++_shared.so',
    'x86': NDK_ROOT + '/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/i686-linux-android/libc++_shared.so',
    'x86_64': NDK_ROOT + '/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/x86_64-linux-android/libc++_shared.so',
}
```

#### 3. 编译
进入`mars/mars`目录下, 执行:

```bash
python3 build_android.py
# 选择3编译xlog
```
等待完成. 出现`Output`即编译成功, 单个abi会编译多次.
#### 4. 使用
仅以`xlog`为例:
1. 复制`mars/libraries/mars_xlog_sdk/libs`下文件到`src/main/jniLibs`文件夹中;
2. 将`mars/libraries/mars_xlog_sdk/src/main/java`下的文件复制到`src/main/java`中, 要包括完整的包名`com.tencent.mars.xlog`;
3. 实际使用参考官方: [https://github.com/Tencent/mars](https://github.com/Tencent/mars?tab=readme-ov-file#xlog-init-3)
4. 如果你有其它库也使用`libc++_shared.so`, 会因为冲突而导致androidstudio编译失败, 其中一个解决办法是只使用第一个版本:
```kts
// app/build.gradle.kts
android {
    // 低版本使用packagingOptions
    packaging { 
        jniLibs {
            pickFirsts.add("**/libc++_shared.so")
        }
    }
}
```

#### 5. 其它

1. 如果有网络问题, 可以在windows中下载然后移动到linux的文件系统中; windows可以直接打开/复制wsl中的文件, windows的文件在linux中加载在`/mnt/{windows盘符}`下;
2. 修改源码直接在该文件夹中使用`code .`即可(自动安装插件并)在windows中使用`vscode`打开
