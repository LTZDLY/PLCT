# 初步学习mugen

## 尝试编写测试用例

### 编写gcc测试用例

> - 开发测试需要编写的文件
>   - testcase文件
>   - common文件（可选）
>   - suite2cases文件
> - testcase常见测试编写方法
>   - 合理运用mugen的库
>     - 全局common_lib
>       - LOG_INFO/WARN/DEBUG/ERROR() 用于打印信息
>       - CHECK_RESULT() 对比结果
>         - `CHECK_RESULT $?` 判断上一语句返回值是否为0，否则判错
>         - `CHECK_RESULT $? 0 0 "log content"` 判断上一语句返回值是否为0，否则> 判错并输出信息log content
>       - DNF_INSTALL/REMOVE() 用于安装和卸载需要测试的软件包
>       - SLEEP_WAIT() 等待命令执行时长，超时判错
>     - cli-test中的systemd单元（详见 testcases/cli-test/common/common_lib.sh）

首先进入mugen的根目录。

创建testcase和common文件：

```shell
[root@openEuler-riscv64 mugen]# mkdir testcases/cli-test/gcc
[root@openEuler-riscv64 mugen]# cd testcases/cli-test/gcc
[root@openEuler-riscv64 gcc]# nano oe_test_gcc.sh
[root@openEuler-riscv64 gcc]# mkdir common
[root@openEuler-riscv64 gcc]# nano test.c
```

testcase主体：oe_test_gcc.sh

```bash
# testcases/cli-test/gcc/oe_test_gcc.sh

function pre_test() {
    LOG_INFO "Start to prepare the test environment."
    DNF_INSTALL gcc
    cp -r ../common ./tmp
    cd ./tmp
    LOG_INFO "End to prepare the test environment."
}
function run_test() {
    LOG_INFO "Start to run test."
    gcc test.c -o test
    CHECK_RESULT $?
    ./test | grep "Hello world!" > /dev/null
    CHECK_RESULT $?
    objdump -D ./test | grep "<main>" > /dev/null
    CHECK_RESULT $?
    LOG_INFO "End to run test."
}
function post_test() {
    LOG_INFO "Start to restore the test environment."
    rm -rf ./tmp
    DNF_REMOVE
    LOG_INFO "End to restore the test environment."
}
```

common/test.c

```c
/// testcases//cli-test//gcc//common//test.c

#include<stdio.h>

int main() {
    printf("Hello world!\n");
    return 0;
}
```

然后回到根目录创建testsuite文件

```bash
[root@openEuler-riscv64 gcc]# cd ../../../
[root@openEuler-riscv64 mugen]# nano suite2cases/gcc.json
```

testsuite：gcc.json

```json
{
    "path": "$OET_PATH/testcases/cli-test/gcc",
    "cases": [
        {
            "name": "oe_test_gcc"
        }
    ]
}
```

测试运行结果

```bash
[root@openEuler-riscv64 mugen]# bash mugen.sh -f gcc
Thu Aug 18 13:54:29 2022 - INFO  - start to run testcase:oe_test_gcc.
Thu Aug 18 13:54:31 2022 - INFO  - The case exit by code 0.
Thu Aug 18 13:54:31 2022 - INFO  - End to run testcase:oe_test_gcc.
Thu Aug 18 13:54:32 2022 - INFO  - A total of 1 use cases were executed, with 1 successes and 0 failures.
```





