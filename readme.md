# LAMMPS 脚本逐行解析

## 文件用途
模拟含球形孔洞的高熵合金在 y 方向单轴拉伸下的应力-应变行为。

---

## 1. 初始环境设置

```lammps
units metal
dimension 3
boundary p p p
atom_style atomic
read_data 4.dat
```

- `units metal` → 金属单位：能量 eV，长度 Å，时间 ps，压力 bars  
- `dimension 3` → 三维模拟  
- `boundary p p p` → x、y、z 三个方向均为周期性边界  
- `atom_style atomic` → 原子类型为简单原子（无键、角等）  
- `read_data 4.dat` → 读入包含原子坐标与盒子信息的初始数据文件  

---

## 2. 孔洞几何与温度变量

```lammps
variable diam equal 8.0
variable radius equal ${diam}/2
variable cx equal 30.0
variable cy equal 30.0
variable cz equal 30.0
variable T equal 298.0
```

- 定义球体直径 `8.0 Å`，半径 `4.0 Å`  
- 球心坐标 `(30,30,30)`  
- 系统温度设定为 `298 K`

---

## 3. 创建孔洞并标记原子

```lammps
region ball sphere ${cx} ${cy} ${cz} ${radius} units box
delete_atoms region ball compress yes
```

- `region ball` → 定义球形区域，名为 `ball`  
- `delete_atoms` → 删除该区域内所有原子，`compress yes` 表示删除后重新压缩盒子  

```lammps
group inner region ball
group outer subtract all inner
```

- `inner` 组 → 包含原先球内区域的原子（目前已无原子）  
- `outer` 组 → 所有原子减去 `inner`，即全部基体原子  

```lammps
fix aprop all property/atom i_a
set group inner i_a 1
set group outer i_a 2
```

- `fix aprop` → 为每个原子添加一个自定义整数属性 `i_a`  
- `inner` 组设为 `1`（实际上无原子），`outer` 组（所有基体原子）设为 `2`  

---

## 4. 势函数与能量最小化

```lammps
pair_style eam/alloy
pair_coeff * * FeNiCrCoAl-heaweight.setfl.txt Ni Al Fe Cr
```

- 使用 EAM（嵌入原子方法）合金势  
- 势文件：`FeNiCrCoAl-heaweight.setfl.txt`  
- 元素映射：原子类型 1→Ni，2→Al，3→Fe，4→Cr  

```lammps
minimize 1e-12 1e-12 10000 100000
```

- 能量最小化：力/能量收敛容差 `1e-12`，最大迭代 100000 步，最大计算 100000 次力  

> 以下三行计算了中心对称参数、原子势能等，但在后续并未使用，属于冗余代码：
> ```lammps
> compute csym all centro/atom bcc
> compute peratom all pe/atom
> variable zz atom c_peratom
> ```

---

## 5. 弛豫阶段（NPT 平衡）

```lammps
reset_timestep 0
timestep 0.001
velocity all create ${T} 12345 mom yes rot no
```

- `reset_timestep 0` → 重置步数为 0  
- `timestep 0.001` → 时间步长 0.001 ps（1 fs）  
- `velocity` → 根据温度 298 K 初始化原子速度，随机数种子 12345，总动量归零，不赋予角速度  

```lammps
fix 1 all npt temp ${T} ${T} 0.1 iso 0 0 1 drag 1.0
thermo 1000
thermo_style custom step lx ly lz press pxx pyy pzz pe temp
dump 1 all custom 5000 *_1tensile.cfg id type x y z
run 10000
```

- `fix npt` → NPT 系综：温度 298 K，控温耦合常数 0.1，目标压力 0 bar，各向同性控压，控压阻尼 1.0  
- `thermo 1000` → 每 1000 步输出一次热力学量  
- 输出内容：步数、盒子长宽高、总压力、各向应力分量（pxx, pyy, pzz）、势能、温度  
- `dump 1` → 每 5000 步输出原子坐标（id, type, x, y, z）到 `*_1tensile.cfg` 文件  
- `run 10000` → 运行 10000 步（10 ps）用于弛豫  

```lammps
unfix 1
undump 1
variable tmp equal "ly"
variable L0 equal ${tmp}
print "Initial Length, L0: ${L0}"
```

- 移除 fix 1 和 dump 1  
- 将当前 y 方向盒子长度赋值给临时变量 `tmp`，再赋给 `L0` 作为初始长度  
- 打印 `L0` 值，供后续应变计算使用  

---

## 6. 拉伸变形设置

```lammps
reset_timestep 0
fix 2 all npt temp ${T} ${T} 1 x 0 0 1 z 0 0 1 drag 1.0
variable srate equal 1.0e10
variable srate1 equal "v_srate / 1.0e14"
fix 3 all deform 1 y erate ${srate1} units box remap x
```

- `reset_timestep 0` → 再次重置步数  
- `fix 2` → NPT 控温 298 K，x 和 z 方向压力保持 0 bar，y 方向不控压（以允许拉伸自由变形）  
- `srate` → 目标应变率 `1.0×10¹⁰ s⁻¹`  
- `srate1` → 单位换算为 ps⁻¹，`1e10 / 1e14 = 1e-4 ps⁻¹`  
- `fix deform` → 沿 y 方向施加工程应变率 `1e-4 ps⁻¹`，盒子变形，原子坐标重映射（`remap x` 实际是 remap 坐标，关键词 `x` 可理解为坐标重映射）

---

## 7. 应力-应变输出与运行

```lammps
variable strain equal "(ly - v_L0)/v_L0"
variable p2 equal "-pxx/10000"
variable p3 equal "-pyy/10000"
variable p4 equal "-pzz/10000"
fix def1 all print 1000 "${strain} ${p2} ${p3} ${p4}" file Al_SC_100.def1.txt screen no
```

- `strain` → 工程应变，用当前 `ly` 和弛豫长度 `L0` 计算  
- `p2, p3, p4` → 将三向应力分量转换为 GPa（1 bar = 1e-4 GPa，`pxx` 单位为 bar，除以 10000 并取反）  
- `fix def1` → 每 1000 步将应变和三个应力值写入文件 `Al_SC_100.def1.txt`，不在屏幕输出  

```lammps
dump 2 all custom 10000 dump.tensile_*.cfg id type x y z i_a
thermo 10000
thermo_style custom step v_strain temp v_p2 v_p3 v_p4 ke pe press
run 1500000
```

- `dump 2` → 每 10000 步输出一次配置，包含原子坐标和自定义属性 `i_a`  
- `thermo 10000` → 每 10000 步屏幕输出热力学信息  
- 输出内容：步数、应变、温度、三个应力分量（GPa）、动能、势能、总压力  
- `run 1500000` → 运行 1,500,000 步，时间步长 0.001 ps，总拉伸时间 1500 ps，总应变 0.15  

```lammps
print "All done"
```

- 模拟结束标志  

---

## 关键参数速查

| 参数          | 值                  | 说明                |
|---------------|---------------------|---------------------|
| 孔洞直径      | 8 Å                 | 球心 (30,30,30)     |
| 温度          | 298 K               |                     |
| 势函数        | EAM/alloy           | FeNiCrCoAl 高熵合金 |
| 弛豫时间      | 10 ps               | NPT, 0 bar          |
| 应变率        | 1×10¹⁰ s⁻¹          | 换算为 1×10⁻⁴ ps⁻¹ |
| 拉伸时间      | 1500 ps             | 总应变 15%          |
| 应力输出文件  | Al_SC_100.def1.txt  | 单位 GPa            |
| 轨迹输出      | dump.tensile_*.cfg  | 包含属性 i_a        |
```

这样，脚本的每一行都被分解并解释，逻辑清晰，可直接用于文档或演示。
