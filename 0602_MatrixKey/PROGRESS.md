# 6.3 矩阵键盘扫描（3×4）实验记录

工程目录：`C:\Users\Lenovo\Desktop\0602 MatrixKey\F407_LED`（内部工程名仍为 F407_LED）
环境：STM32CubeIDE + ARM GCC 命令行 make + OpenOCD(CMSIS-DAP) 烧录；不使用 Keil。
芯片：STM32F407ZETX，168MHz（HSE 8MHz，PLLM=8/N=336/P=2）。

## 一、引脚映射（实测通过）

| 功能 | 引脚 | 方向/模式 | 说明 |
|---|---|---|---|
| 行线 ROW0 | PE4 | 推挽输出、低速、无上下拉、初始高 | 扫描时逐行拉低 |
| 行线 ROW1 | PE5 | 同上 | |
| 行线 ROW2 | PE6 | 同上 | |
| 列线 COL0 | PE0 | 上拉输入 | 按下被拉低 |
| 列线 COL1 | PE1 | 上拉输入 | |
| 列线 COL2 | PE2 | 上拉输入 | |
| 列线 COL3 | PE3 | 上拉输入 | |
| LED1~LED8 | PF0~PF7 | 推挽输出、无上下拉 | **LED 低电平点亮**，显示需取反 |

## 二、核心代码要点

### gpio.c（MX_GPIO_Init）
- 开 GPIOE/GPIOF/GPIOH/GPIOA 时钟；`__HAL_RCC_GPIOE_CLK_ENABLE();`
- PE4/5/6 = GPIO_MODE_OUTPUT_PP / GPIO_NOPULL / SPEED_LOW，初始 GPIO_PIN_SET
- PE0~PE3 = GPIO_MODE_INPUT / GPIO_PULLUP / SPEED_LOW

### main.c
- 变量（USER CODE BEGIN PV）：
  `uint16_t i; uint16_t KeyVal=0; uint8_t PreLine[3]={2,0,1};`
- 扫描逻辑（while 内）：
  - `for(i=0;i<3;i++)`，先判 `(GPIOE->IDR & 0x0F)!=0x0F`，`HAL_Delay(16)` 消抖后再判
  - switch 列值：0x0E→列1、0x0D→列2、0x0B→列3、0x07→列4
  - `KeyVal = 4*PreLine[i] + 列号`；PreLine[0]=2、PreLine[1]=0、PreLine[2]=1
  - 扫描行输出：`GPIOE->ODR = (~(1<<(i+4))) & 0x00FF;`
- BCD 显示（低电平点亮，故取反）：
  `GPIOF->ODR = (~(((KeyVal/10)<<4) + (KeyVal%10))) & 0x00FF;`
  高 4 位=十位（PF7~PF4），低 4 位=个位（PF3~PF0）。

### 键值表（KeyVal）
| 扫描 i / 物理行 | 列1 | 列2 | 列3 | 列4 |
|---|---|---|---|---|
| i=1 行0 | 1 | 2 | 3 | 4 |
| i=2 行1 | 5 | 6 | 7 | 8 |
| i=0 行2 | 9 | 10 | 11 | 12 |

## 三、编译与烧录命令模板

编译（前置 GCC 后在 Debug 目录执行）：
```powershell
$gccbin="D:\Download1\stm32\STM32CubeIDE_2.2.0\STM32CubeIDE\plugins\com.st.stm32cube.ide.mcu.externaltools.gnu-tools-for-stm32.14.3.rel1.win32_1.0.100.202602081740\tools\bin"
$make="D:\Download1\stm32\STM32CubeIDE_2.2.0\STM32CubeIDE\plugins\com.st.stm32cube.ide.mcu.externaltools.make.win32_2.2.200.202604021615\tools\bin\make.exe"
$env:PATH="$gccbin;$env:PATH"
Set-Location "C:\Users\Lenovo\Desktop\0602 MatrixKey\F407_LED\Debug"
& $make all
```

烧录（路径含空格，先把 elf 复制到 %TEMP%；OpenOCD 日志走 stderr，用 `*> 文件` 落盘再读）：
```powershell
Copy-Item ".\F407_LED.elf" "$env:TEMP\F407_LED.elf"
$od="$env:TEMP\F407_LED.elf" -replace '\\','/'
& $ocd -s $scr -f "interface/cmsis-dap.cfg" -f "target/stm32f4x.cfg" `
  -c "adapter speed 1000" -c "init" -c "reset halt" `
  -c "program $od verify reset" -c "shutdown" *> "$env:TEMP\ocd_flash.log"
```
成功标志：`Programming Finished` / `Verified OK` / `Resetting Target`，退出码 0。

## 四、踩坑记录

1. makefile 链接脚本路径含空格：依赖项里裸写绝对路径会报 `No rule to make target 'C:\Users\Lenovo\Desktop\0602'`；删除该行依赖声明、保留 `-T"..."` 引号路径即可。
2. 清理 Debug 产物时不能按 `.list` 扩展名删（会误删 objects.list 链接输入表）。
3. OpenOCD 日志走 stderr，PowerShell 管道 `Select-String` 会吞输出；必须 `*> 日志文件` 再 `Get-Content`。
4. 教材直送 BCD 到 ODR，本板 LED 低有效导致显示反码（全亮/灭灯），取反后显示正常。
5. 旧 0601 独立按键实验实物为 PA6(低有效)/PA7(高有效)，与本矩阵实验 PE 口无冲突，勿混用配置。

## 五、实测结论
1~12 键按下亮灯组合与 8421 BCD 示例相符，松开保持最后键值（教材逻辑；如需松手即灭可在无键时 KeyVal=0）。
