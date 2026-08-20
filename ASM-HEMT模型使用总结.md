# ASM-HEMT 模型使用完整总结

## 一、这个模型是什么

你在本地下载的 `ASM-HEMT-main` 是 **GaN HEMT（氮化镓高电子迁移率晶体管）的紧凑模型**，行业标准型号，用于 RF 射频和功率器件的电路仿真。

它不是能直接双击启动的软件，而是用 **Verilog-A 语言**写成的 SPICE 模型代码，必须装进支持 Verilog-A 的电路仿真器中编译后，才能在电路网表里调用。

**仓库结构**：
```
ASM-HEMT-main/
├── ASMHEMT101.0.0_Manual.pdf   ← 技术手册（53页，已读完）
├── vacode/
│   └── asmhemt.va              ← 模型核心代码（Verilog-A，1232行）
├── research_papers/            ← 相关论文
└── videos/                     ← 视频资料
```

**模型关键信息**（来自 `asmhemt.va`）：
- 器件名：`asmhemt`（第289行 `module asmhemt(d,g,s,b,dt)`）
- 5 个端口：`d`(漏极) `g`(栅极) `s`(源极) `b`(衬底) `dt`(热节点，可悬空)
- 版本 101.0.0，所有模型参数都有默认值，可直接跑通

## 二、怎么使用（三步流程）

**第1步：装仿真器**（二选一）
- **免费开源**：ngspice + OpenVAF
- **商业**（学校有正版 license 可用）：Cadence Spectre / Keysight ADS / Synopsys HSPICE / Silvaco SmartSpice（直接编译 .va，无需 OpenVAF）

**第2步：编译模型**（以免费方案为例，只需做一次）
```
openvaf asmhemt.va    # 生成 asmhemt.osdi
```

**第3步：写网表仿真**（跑 Id-Vg 特性曲线示例）
```
.load asmhemt.osdi        # ngspice 里加载编译产物
x1 d g s b dt asmhemt     # 实例化器件
vg g 0 dc -5
vd d 0 dc 10
.dc vg -6 2 0.1
.plot i(vd)
```

## 三、工具下载地址

**ngspice**（仿真器主程序）
- 官网: https://ngspice.sourceforge.io/
- Windows 安装包: https://sourceforge.net/projects/ngspice/files/

**OpenVAF**（Verilog-A 编译器，把 .va 编译成 ngspice 可加载的 .osdi）
- 官网: https://openvaf.semimod.de/
- GitHub Releases: https://github.com/pascalkuthe/OpenVAF/releases

**两者都要装**：OpenVAF 只用于一次编译（类似编译器），ngspice 用于日常仿真（主程序）。编译完 `.osdi` 后，之后只用 ngspice。

## 四、手册核心要点

**模型原理**：物理基（表面势）紧凑模型，分三部分：
1. 核心 2-DEG 电荷/表面势计算（基于薛定谔-泊松方程的解析解）
2. 端电流与电荷模型（Ward-Dutton 划分，满足电荷守恒）
3. 真实器件效应模型

**内置物理效应**：速度饱和、迁移率退化、DIBL、亚阈斜率、沟道长度调制、自热、温度依赖、非线性源漏串联电阻、栅电流、陷阱模型、场板（最多4个FP）、flicker/热噪声

**关键参数**（第4章参数表）：
- 几何/工艺：`L, W, NF, TBAR, LSG, LDG, EPSILON`
- 模型开关：`RDSMOD, GATEMOD, SHMOD, TRAPMOD, FNMOD, TNMOD, FP1MOD~FP4MOD, RGATEMOD`（新手保持默认）
- 核心电学：`VOFF`(默认-2V 截止电压), `NFACTOR`(亚阈斜率), `U0/UA/UB`(迁移率), `VSAT`(饱和速度), `LAMBDA`(CLM), `ETA0`(DIBL), `VDSCALE`

**参数提取顺序**（第5.1节实操指南）：
1. 先定几何参数 L/W/NF/TBAR/LSG/LDG
2. 线性区（Vd≈50-100mV）Id-Vg 提取 **VOFF、NFACTOR**
3. 同区提取迁移率 **U0、UA、UB**（配合 RSC、NS0ACCS/D、MEXPACCS/D）
4. 高 Vd 区提取 **ETA0、VDSCALE**（DIBL）、**CDSCD**（亚阈退化）
5. 高 Vd 线性区提取 **VSAT、LAMBDA** 及串联电阻参数
6. Id-Vd 输出特性微调
7. 之后扩展：温度参数 → S参数（CGSO/CGDO/CGDL/VDSATCV、RSHG/XGW）→ 大信号 RF

**模型验证**（第5.2节）：已通过 Gummel 对称性、谐波平衡、AC 对称性、互易性测试，模型质量有保障。

## 五、你的电脑配置

- 16GB 内存 + i5-12450H + 64位 Windows
- 完全带得动：OpenVAF 编译峰值约几百 MB，ngspice 仿真几十到几百 MB，即便复杂 RF 仿真也远够用

## 六、当前状态

已读完全部手册，工具暂未下载安装。下一步随时可以做：帮你下载安装 ngspice + OpenVAF，并搭建第一个能跑的仿真环境（含示例网表）。