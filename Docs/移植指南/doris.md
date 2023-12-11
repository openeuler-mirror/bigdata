# Doris 移植

---

硬件环境：

1. 系统架构：ARM X64
2. CPU：至少4 C
3. 内存：至少16 GB
4. 硬盘：至少40GB（SSD）、至少100GB（SSD）

---

1. 创建软件下载安装包根目录和软件安装根目录
   
   ```
   # 创建软件下载安装包根目录
   mkdir /opt/tools
   # 创建软件安装根目录
   mkdir /opt/software
   ```

2. 安装 git
   
   `yum install -y git`

3. 安装 java 环境
   
   ```
   cd /opt/tools
   wget https://doris-thirdparty-repo.bj.bcebos.com/thirdparty/jdk-8u291-linux-aarch64.tar.gz && \
       tar -zxvf jdk-8u291-linux-aarch64.tar.gz && \
       mv jdk1.8.0_291 /opt/software/jdk8
   ```

4. 安装 maven
   
   ```
   cd /opt/tools
   # wget 工具下载后，直接解压缩配置环境变量使用
   wget https://dlcdn.apache.org/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz && \
       tar -zxvf apache-maven-3.6.3-bin.tar.gz && \
       mv apache-maven-3.6.3 /opt/software/maven
   ```

5. 安装 nodejs
   
   ```
   cd /opt/tools
   # 下载 arm64 架构的安装包
   wget https://doris-thirdparty-repo.bj.bcebos.com/thirdparty/node-v16.3.0-linux-arm64.tar.xz && \
       tar -xvf node-v16.3.0-linux-arm64.tar.xz && \
       mv node-v16.3.0-linux-arm64 /opt/software/nodejs
   ```

6. 安装 LDB_Toolchain
   
   ```
   cd /opt/tools
   # 下载 LDB-Toolchain ARM 版本
   wget https://github.com/amosbird/ldb_toolchain_gen/releases/download/v0.9.1/ldb_toolchain_gen.aarch64.sh && \
      sh ldb_toolchain_gen.aarch64.sh /opt/software/ldb_toolchain/
   ```

7. 配置环境变量
   
   ```
   # 配置环境变量
   vim /etc/profile.d/doris.sh
   export JAVA_HOME=/opt/software/jdk8
   export MAVEN_HOME=/opt/software/maven
   export NODE_JS_HOME=/opt/software/nodejs
   export LDB_HOME=/opt/software/ldb_toolchain
   export PATH=$JAVA_HOME/bin:$MAVEN_HOME/bin:$NODE_JS_HOME/bin:$LDB_HOME/bin:$PATH
   
   # 保存退出并刷新环境变量
   source /etc/profile.d/doris.sh
   ```

8. 安装其他额外环境和组件
   
   ```
   sudo yum install -y byacc patch automake libtool make which file ncurses-devel gettext-devel unzip bzip2 bison zip util-linux wget git python3
   
   # install autoconf-2.69
   cd /opt/tools
   wget http://ftp.gnu.org/gnu/autoconf/autoconf-2.69.tar.gz && \
       tar zxf autoconf-2.69.tar.gz && \
       mv autoconf-2.69 /opt/software/autoconf && \
       cd /opt/software/autoconf && \
       ./configure && \
       make && \
       make install
   ```

9. 下载源码
   
   ```
   cd /opt
   git clone https://github.com/apache/doris.git
   ```

10. 打开 doris 根目录下的 env.sh 脚本，将 115 行设置为 ` DORIS_TOOLCHAIN=gcc`, 更换为 gcc 编译器
    
    ```
    if [[ -z "${DORIS_TOOLCHAIN}" ]]; then
        if [[ "$(uname -s)" == 'Darwin' ]]; then
            DORIS_TOOLCHAIN=clang
        else
            DORIS_TOOLCHAIN=gcc
        fi
    fi
    ```

11. 编辑 /thirdparty下的 `build-thirdparty.sh` 文件，将 1656 行的 `build_hadoop_libs()` 函数替换为以下内容
    
    ```
    build_hadoop_libs() {
        check_if_source_exist "${HADOOP_LIBS_SOURCE}"
        cd "${TP_SOURCE_DIR}/${HADOOP_LIBS_SOURCE}"
        echo "THIRDPARTY_INSTALLED=${TP_INSTALL_DIR}" >env.sh
    
        ASAN_SYMBOLIZER_PATH_TEMP=ASAN_SYMBOLIZER_PATH
        BUILD_SYSTEM_TEMP=BUILD_SYSTEM
        BUILD_THIRDPARTY_WIP_TEMP=BUILD_THIRDPARTY_WIP
        CC_TEMP=CC
        CCACHE_COMPILERCHECK_TEMP=CCACHE_COMPILERCHECK
        CCACHE_PCH_EXTSUM_TEMP=CCACHE_PCH_EXTSUM
        CCACHE_SLOPPINESS_TEMP=CCACHE_SLOPPINESS
        CLANG_COMPATIBLE_FLAGS_TEMP=CLANG_COMPATIBLE_FLAGS
        CMAKE_CMD_TEMP=CMAKE_CMD
        CMAKE_PREFIX_PATH_TEMP=CMAKE_PREFIX_PATH
        CXX_TEMP=CXX
        DORIS_BIN_UTILS_TEMP=DORIS_BIN_UTILS
        DORIS_CLANG_HOME_TEMP=DORIS_CLANG_HOME
        DORIS_GCC_HOME_TEMP=DORIS_GCC_HOME
        DORIS_HOME_TEMP=DORIS_HOME
        DORIS_THIRDPARTY_TEMP=DORIS_THIRDPARTY
        GENERATOR_TEMP=GENERATOR
        LC_ALL_TEMP=LC_ALL
        LD_LIBRARY_PATH_TEMP=LD_LIBRARY_PATH
        PKG_CONFIG_PATH_TEMP=PKG_CONFIG_PATH
        PYTHON_TEMP=PYTHON
        REPOSITORY_URL_TEMP=REPOSITORY_URL
        TP_DIR_TEMP=TP_DIR
        TP_INCLUDE_DIR_TEMP=TP_INCLUDE_DIR
        TP_INSTALL_DIR_TEMP=TP_INSTALL_DIR
        TP_JAR_DIR_TEMP=TP_JAR_DIR
        TP_LIB_DIR_TEMP=TP_LIB_DIR
        TP_PATCH_DIR_TEMP=TP_PATCH_DIR
        TP_SOURCE_DIR_TEMP=TP_SOURCE_DIR
    
        unset ASAN_SYMBOLIZER_PATH
        unset BUILD_SYSTEM
        unset BUILD_THIRDPARTY_WIP
        unset CC
        unset CCACHE_COMPILERCHECK
        unset CCACHE_PCH_EXTSUM
        unset CCACHE_SLOPPINESS
        unset CLANG_COMPATIBLE_FLAGS
        unset CMAKE_CMD
        unset CMAKE_PREFIX_PATH
        unset CXX
        unset DORIS_BIN_UTILS
        unset DORIS_CLANG_HOME
        unset DORIS_GCC_HOME
        unset DORIS_HOME
        unset DORIS_THIRDPARTY
        unset GENERATOR
        unset LC_ALL
        unset LD_LIBRARY_PATH
        unset PKG_CONFIG_PATH
        unset PYTHON
        unset REPOSITORY_URL
        unset TP_DIR
        unset TP_INCLUDE_DIR
        unset TP_INSTALL_DIR
        unset TP_JAR_DIR
        unset TP_LIB_DIR
        unset TP_PATCH_DIR
        unset TP_SOURCE_DIR
    
        ./build.sh
    
        ASAN_SYMBOLIZER_PATH=ASAN_SYMBOLIZER_PATH_TEMP
        BUILD_SYSTEM=BUILD_SYSTEM_TEMP
        BUILD_THIRDPARTY_WIP=BUILD_THIRDPARTY_WIP_TEMP
        CC=CC_TEMP
        CCACHE_COMPILERCHECK=CCACHE_COMPILERCHECK_TEMP
        CCACHE_PCH_EXTSUM=CCACHE_PCH_EXTSUM_TEMP
        CLANG_COMPATIBLE_FLAGS=CLANG_COMPATIBLE_FLAGS_TEMP
        CMAKE_CMD=CMAKE_CMD_TEMP
        CMAKE_PREFIX_PATH=CMAKE_PREFIX_PATH_TEMP
        CXX=CXX_TEMP
        DORIS_BIN_UTILS=DORIS_BIN_UTILS_TEMP
        DORIS_CLANG_HOME=DORIS_CLANG_HOME_TEMP
        DORIS_GCC_HOME=DORIS_GCC_HOME_TEMP
        DORIS_HOME=DORIS_HOME_TEMP
        DORIS_THIRDPARTY=DORIS_THIRDPARTY_TEMP
        GENERATOR=GENERATOR_TEMP
        LC_ALL=LC_ALL_TEMP
        LD_LIBRARY_PATH=LD_LIBRARY_PATH_TEMP
        PKG_CONFIG_PATH=PKG_CONFIG_PATH_TEMP
        PYTHON=PYTHON_TEMP
        REPOSITORY_URL=REPOSITORY_URL_TEMP
        TP_DIR=TP_DIR_TEMP
        TP_INCLUDE_DIR=TP_INCLUDE_DIR_TEMP
        TP_INSTALL_DIR=TP_INSTALL_DIR_TEMP
        TP_JAR_DIR=TP_JAR_DIR_TEMP
        TP_LIB_DIR=TP_LIB_DIR_TEMP
        TP_PATCH_DIR=TP_PATCH_DIR_TEMP
        TP_SOURCE_DIR=TP_SOURCE_DIR_TEMP
    
        mkdir -p "${TP_INSTALL_DIR}/include/hadoop_hdfs/"
        mkdir -p "${TP_INSTALL_DIR}/lib/hadoop_hdfs/"
        cp -r ./hadoop-dist/target/hadoop-libhdfs-3.3.4/* "${TP_INSTALL_DIR}/lib/hadoop_hdfs/"
        cp -r ./hadoop-dist/target/hadoop-libhdfs-3.3.4/include/hdfs.h "${TP_INSTALL_DIR}/include/hadoop_hdfs/"
        rm -rf "${TP_INSTALL_DIR}/lib/hadoop_hdfs/native/*.a"
        find ./hadoop-dist/target/hadoop-3.3.4/lib/native/ -type f ! -name '*.a' -exec cp {} "${TP_INSTALL_DIR}/lib/hadoop_hdfs/native/" \;
        find ./hadoop-dist/target/hadoop-3.3.4/lib/native/ -type l -exec cp -P {} "${TP_INSTALL_DIR}/lib/hadoop_hdfs/native/" \;
    
    }
    ```

12. 在根目录下执行编译脚本（由于网络原因下载第三方库时可能会失败，需要多尝试几次）
    
    ```
    export REPOSITORY_URL=https://doris-thirdparty-repo.bj.bcebos.com/thirdparty
    USE_AVX2=OFF sh build.sh
    ```
