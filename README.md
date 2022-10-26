# MEGAN3.2模型相关代码和备忘录

编者：王浩帆

邮箱：wanghf58@mail2.sysu.edu.cn

本教程所需的所有数据均可在百度网盘进行下载：

链接：https://pan.baidu.com/s/19GVkILp2psCdOqSTYuewUg?pwd=8888 

## 1. Intel编译器编译MEGAN3.2 (这部分还存在问题)

解压文件
```shell
unzip MEGANv3.2_Aug_2022.zip
cd MEGANv3.21_Oct_2022/MEGANv3.2_Dec_2021/src
```
修改makefile文件
```shell
cd TXT2IOAPI
rm makefile
cp makefile.intel makefile
vi makefile  # 添加IOAPI和NETCDF路径
```
在此以前，你需要将netcdf的lib文件放入ioapi的lib文件中。
```shell
cp ~/wanghf/CMAQ-work/libraries/netcdf/lib/*.a ~/wanghf/CMAQ-work/libraries/ioapi32/Linux2_x86_64ifort/
```
修改以下部分：
```shell
SHELL=/bin/sh
FC = ifort
#FFLAGS = -O3 -fixed -132 -traceback -xHost -Bstatic
#FFLAGS = -check all -fp-stack-check -O3 -g -fixed -132 -traceback -qopenmp -xHost -Bstatic
FFLAGS = -O2 -fp-model precise -fpp -132 -mcmodel=large -convert big_endian
IOAPI = /WORK/sysu_fq_1/wanghf/CMAQ-work/libraries/ioapi32
IOAPI_LIB = ${IOAPI}/Linux2_x86_64ifort
NETCDF = /WORK/sysu_fq_1/wanghf/CMAQ-work/libraries/netcdf
NETCDF_LIB = ${NETCDF}/lib
NETCDF_INC = ${NETCDF}/include
CURDIR = /WORK/sysu_fq_1/wanghf/MEGAN-work/v3.2/MEGANv3.21_Oct_2022/MEGANv3.2_Dec_2021/src/TXT2IOAPI

LIBS =   -L$(IOAPI_LIB) -lioapi \
         -L$(NETCDF_LIB) -lnetcdff -lnetcdf
INCLUDE = -I$(IOAPI)/ioapi/fixed_src \
          -I$(NETCDF_INC) \
          -I$(CURDIR)/INCLDIR
PROGRAM = txt2ioapi
```
保存后直接make，生成txt2ioapi则说明编译成功。

## 2. 如何将MEGAN3.2运用到CMAQ5.4中进行online计算？

主要步骤就是准备输入数据。

### 2.1 Emission Factor Processor (Python code)

先创建一个新的环境
```shell
conda create -n megan32 python=3.8
```
进入MEGEFP32安装必备的第三方库
```shell
pip install -r requirements.txt
```
### 2.2 gfortran编译MEGAN32_Prep_Code_Jan_2022

解压源代码：

```shell
tar zxvf MEGAN32_Prep_Code_Jan_2022.tar.gz
cd MEGAN32_Prep_Code_Jan_2022
vim Makefile
```

修改部分内容为：

```shell
F90    = gfortran #$(FC)
NETCDF_DIR = /usr
LIBS   = -L$(NETCDF_DIR)/lib -lnetcdf -lnetcdff
```
保存文件以后，即可以开始编译。

在CMAQ5.4中必需的三个文件是通过`prepmegan4cmaq_grwform.f90`,`prepmegan4cmaq_ecotype.f90`以及`prepmegan4cmaq_cantype.f90`生成。此外，如果你还需要自己准备LAI文件，则选择性编译`prepmegan4cmaq_lai.f90`。

在终端中输入以下命令既可以开始编译：

```shell
./make_util prepmegan4cmaq_grwform.x  # 编译prepmegan4cmaq_grwform.f90
./make_util prepmegan4cmaq_ecotype.x  # 编译prepmegan4cmaq_ecotype.f90
./make_util prepmegan4cmaq_cantype.x  # 编译prepmegan4cmaq_cantype.f90
./make_util prepmegan4cmaq_lai.x  # 编译prepmegan4cmaq_lai.f90
```

如果出现以下类似错误：

```shell
NETCDF_DIR = /usr

arg PREPMEGAN4CMAQ_LAI.X
gfortran  -O3 -Bstatic -c -I/usr/include misc_definitions_module.f90
gfortran  -O3 -Bstatic -c -I/usr/include constants_module.f90
gfortran  -O3 -Bstatic -c -I/usr/include bio_types.f90
gfortran  -O3 -Bstatic -c -I/usr/include area_mapper.f90
gfortran  -O3 -Bstatic -c -I/usr/include prepmegan4cmaq_lai.f90
prepmegan4cmaq_lai.f90:831:30:

  831 |                write (99, '(3(I,","), 48(F10.4,","))')   &
      |                              1
Error: Nonnegative width required in format string at (1)
make: *** [Makefile:77: prepmegan4cmaq_lai.o] Error 1
failed to build  prepmegan4cmaq_lai.x
```

打开`prepmegan4cmaq_lai.f90`文件，跳转到831行，将此行改为：

```fortran
              write (99, '(3(I0,","), 48(F10.4,","))')   &
```

然后重新编译即可，每个程序如果编译成功，会在屏幕末尾打印类似如下字符串：

```shell
++++++++++++++++++++++++
prepmegan4cmaq_lai.x build successful
++++++++++++++++++++++++
```

### 2.3 准备CMAQ5.4所需的相关数据

CMAQ5.4所需的数据为NC格式文件，但是在此以前，我们需要通过`MEGAN32_Prep_Code_Jan_2022`和`Emission Factor Processor (Python code)`将官网提供的全球数据处理map到我们自己的CMAQ网格（此处文件以csv形式输出），然后通过MEGAN3.2中的`txt2ioapi`将csv文件转换为NC格式。因此，此小节将通过两个部分来进行说明，第一部分为介绍如何将官网提供的全球数据map到我们自己的CMAQ网格；第二部分为介绍如何通过txt2ioapi将csv文件转换为NC格式。

所有测试文件都可以在MEGAN官网下载到，但是由于谷歌硬盘的下载速度太慢，因此所有的数据会在百度网盘提供，如有需要可联系作者（邮箱见文首）获取。

#### 2.3.1 准备csv文件

* **Growth form**

`prepmegan4cmaq.growthform.inp`为运行`prepmegan4cmaq_grwform.x`的输入参数模板。

```shell
cp prepmegan4cmaq.growthform.inp prepmegan4cmaq.growthform_mycase.inp
vim prepmegan4cmaq.growthform_mycase.inp
```

检查此文件中的参数是否正确，确认后保存。运行以下代码，生成关于Growth form的csv文件。

```shell
./prepmegan4cmaq_grwform.x < prepmegan4cmaq.growthform_mycase.inp
```

**注意：**可能会出现以下错误。

```fortran
At line 215 of file prepmegan4cmaq_grwform.f90
Fortran runtime error: Bad value during floating point read
```

打开`prepmegan4cmaq_grwform.f90`文件，将第215行前后的代码修改为：

```fortran
!-----read growth form----------------------------------------------
      scale_factor  = 1.
      inpname       = trim(treefilename)
      varname       = trim(treevarname)
      missing_value = -1.
      !read(treefillvalue,"(f7.0)") missing_value
      CALL  megan2_bioemiss
      inpname       = trim(shrubfilename)
      varname       = trim(shrubvarname)
      !read(shrubfillvalue,"(f7.0)") missing_value
      CALL  megan2_bioemiss
      inpname       = trim(cropfilename)
      varname       = trim(cropvarname)
      !read(cropfillvalue,"(f7.0)") missing_value
      CALL  megan2_bioemiss
      inpname       = trim(grassfilename)
      varname       = trim(grassvarname)
      !read(grassfillvalue,"(f7.0)") missing_value
      CALL  megan2_bioemiss

!     endif
```

然后执行以下代码重新编译后再运行：

```shell
./make_util prepmegan4cmaq_grwform.x  # 编译prepmegan4cmaq_grwform.f90
./prepmegan4cmaq_grwform.x < prepmegan4cmaq.growthform_mycase.inp
```

* **Ecotype**

`prepmegan4cmaq.ecotype.inp`为运行`prepmegan4cmaq_ecotype.x`的输入参数模板。

```shell
cp prepmegan4cmaq.ecotype.inp prepmegan4cmaq.ecotype_mycase.inp
vim prepmegan4cmaq.ecotype_mycase.inp
```

检查此文件中的参数是否正确，确认后保存。运行以下代码，生成关于Ecotype的csv文件。

```shell
./prepmegan4cmaq_ecotype.x < prepmegan4cmaq.ecotype_mycase.inp
```

* **Cantype**

`prepmegan4cmaq.cantype.inp`为运行`prepmegan4cmaq_cantype.x`的输入参数模板。

```shell
cp prepmegan4cmaq.cantype.inp prepmegan4cmaq.cantype_mycase.inp
vim prepmegan4cmaq.cantype_mycase.inp
```

检查此文件中的参数是否正确，确认后保存。运行以下代码，生成关于Cantype的csv文件。

```shell
./prepmegan4cmaq_cantype.x < prepmegan4cmaq.cantype_mycase.inp
```

* **Emission factor**

**注意：Growth form和Ecotype是无法被直接输入到MEGAN3.2的，因此需要通过**`Emission Factor Processor (Python code)`**来进行转换，将其转换为Emission Factor文件**。

将`MEGAN3.2/output/grid_ecotype.csv`和`MEGAN3.2/output/grid_growth_form.csv`分别重命名为`MEGAN3.2/output/grid_ecotype.PRD.csv`和`MEGAN3.2/output/grid_growth_form.PRD.csv`,然后将重命名后的文件放入`MEGAN3.2/MEGEFP32/inputs/EFP`中，然后修改`MEGAN3.2/MEGEFP32/MEGAN_EFP.py`的第**39**行。

```python
scen_name = "PRD"
```

然后运行，如果在屏幕最后输出如下所示内容，则说明运行成功。

```
grid EF Output: ./outputs/OutputGridEF.PRD.csv
```

#### 2.3.1 csv转换为NETCDF

进入MEGAN3.21目录，确定PRD网格的相关文件在此目录下。

```shell
cd ~/wanghf/MEGAN-work/v3.2/MEGANv3.21_Oct_2022/MEGANv3.2_Dec_2021/Input/MAP
ls *.PRD.csv
```

如果屏幕中能够出现以下文件则说明输入文件以准备好。

```
CT3.PRD.csv  OutputGridEF.PRD.csv
```

进入到工作目录准备开始运行`txt2ioapi`程序。

```shell
cd ~/wanghf/MEGAN-work/v3.2/MEGANv3.21_Oct_2022/MEGANv3.2_Dec_2021/work
vim run.txt2ioapi.v32.csh
```

修改以下部分：

```shell
foreach dom ( PRD_BASE )

## Run control
setenv RUN_EFS      T  # [T|F]
setenv RUN_LAI      F  # [T|F]
setenv RUN_CANTYP   T  # [T|F]
setenv RUN_W126     F  # [T|F]
setenv RUN_LDF      T  # [T|F]
setenv RUN_ARID     F  # [T|F] 
setenv RUN_NONARID  F  # [T|F]
setenv RUN_FERT     F  # [T|F] fertilizer data
setenv RUN_NITROGEN F  # [T|F] total nitrogen deposition data (ng/m2/s)
setenv RUN_LANDTYPE F  # [T|F] biome type
```

以及增加以下内容到如下位置：

```shell
################# Outputs #######################
setenv EFMAPS  $MGNINP/MAP/EFMAP.2019b.${GDNAM3D}.ncf
setenv CANTYP  $MGNINP/MAP/CT3_${GDNAM3D}.ncf
setenv LAIFILE  $MGNINP/MAP/LAI3_${GDNAM3D}.ncf
setenv W126FILE $MGNINP/MAP/W126_${GDNAM3D}.ncf
setenv LDFILE $MGNINP/MAP/LDF_${GDNAM3D}.2019b.ncf
setenv ARIDFILE $MGNINP/MAP/ARID_${GDNAM3D}.ncf
setenv NONARIDFILE $MGNINP/MAP/NONARID_${GDNAM3D}.ncf
setenv LANDTYPEFILE $MGNINP/MAP/LANDTYPE_${GDNAM3D}.ncf
setenv FERTRESFILE $MGNINP/MAP/FERT_${GDNAM3D}.ncf
setenv NDEPFILE $MGNINP/MAP/NDEP_${GDNAM3D}.ncf
setenv CANTYPFILE $MGNINP/MAP/CT3_${GDNAM3D}.ncf # 这部分是增加的内容，其余可以不变
```

















