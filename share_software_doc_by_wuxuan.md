# 简介

本文档介绍如何在合肥超算上使用共享目录下的`` Smilei ``程序和 ``FaTiDo ``程序。



# 前置小修小补

有的用户出于某些特殊原因可能会把系统自带的文件和目录删除。为了后续操作能正常进行，首先检查`~/`目录下是否有以下文件/目录：

```bash
# 直接执行以下命令即可，它们不会对已存在的文件/目录产生影响
$ mkdir -p ~/.local/bin
$ touch ~/.bashrc
$ touch ~/.bash_profile
```



# 环境变量

通过编写``.bashrc``和``.bash_profile``文件可以在每次登录系统时自动执行一些操作，本文提供的``.bashrc``和``.bash_profile``文件主要实现以下功能：

- 卸载超算自动加载的库，加载所需版本的库（见``set modules``部分）
- 设置环境变量（见``set env variables``部分），包括：
  - 将``$HOME/.local/bin``链接到路径中（换言之，执行该目录下的文件时，``$HOME/.local/bin``可以省略）
  - 将共享目录中``python``和``hdf5``的动态链接库路径链接到系统的动态链接库路径中（听起来很绕，但bash script看起来还是很好懂的）
- 加载系统自带的全局定义（见``Source global definitions``部分）

详细解释可参考任意AI给出的说明或者吴同的文档。基于吴同的文档，笔者给出功能相同、但仅使用共享目录下软件的另一版本：

```bash
# .bash_profile

# set modules
module purge
module load compiler/gcc/12.3.0
module load apps/cmake/3.21.0
module load mpi/intelmpi/2021.3.0

# set env variables
export PATH=$HOME/.local/bin:$PATH
export LD_LIBRARY_PATH=/public/share/ac58qn21ek/share_local/programs/python/lib:/public/share/ac58qn21ek/share_local/programs/hdf5/lib:$LD_LIBRARY_PATH



# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

```

```bash
# .bashrc

# Source global definitions
if [ -f /etc/bashrc ]; then
        . /etc/bashrc
fi
```





# 软链接

为了让提交脚本里的可执行文件路径更具可读性，可选择将共享目录下的文件软链接至用户目录下。

进行软链接：

```bash
$ ln -s /public/share/ac58qn21ek/share_local/programs/python/bin/python3.9 ~/.local/bin/python3
$ ln -s /public/share/ac58qn21ek/share_local/programs/python/bin/pip3.9 ~/.local/bin/pip3
$ ln -s /public/share/ac58qn21ek/share_local/programs/Smilei-5.1/smilei ~/.local/bin/smilei
```

这样就可以仅通过可执行文件名称定位到共享目录下的文件。以python为例，可通过以下方式检查链接是否正确：

```bash
$ which python3
~/.local/bin/python3
$ ls -l ~/.local/bin/python3
lrwxr-xr-x 1 xuanwu ac58qn21ek 66 2月  22 10:09 /public/home/xuanwu/.local/bin/python3 -> /public/share/ac58qn21ek/share_local/programs/python/bin/python3.9
```



# Smilei

上述步骤执行完毕后，就可以通过编写输入脚本运行``Smilei``和``FaTiDo``。

``Smilei``的一个提交脚本示例如下：

```bash
#!/bin/sh
#SBATCH -J test_v1
#SBATCH -p hfacnormal02
#SBATCH --nodes=2              #number of nodes
#SBATCH --ntasks-per-node=8    #MPI task per node
#SBATCH --cpus-per-task=16     #threads per task
#SBATCH -o out%j.log
#SBATCH -e err%j.log
#SBATCH -t 12:00:00



export OMP_NUM_THREADS=16    #theards per processes
export OMP_SCHEDULE=dynamic  #balance between threads
export OMP_PROC_BIND=true    #bind process to CPU
export I_MPI_PIN_DOMAIN=omp  #start the OMP in MPIs



mpirun -n 16 smilei /path/to/input/file
```

需要注意：

- ``mpi -n``之后的数字为核数（``tasks = nodes * ntasks-per-node = 2 * 8 = 16``）
- 修改``/path/to/input/file``为``Smilei``的输入文件的路径

在超算上通过高性能计算-作业提交-模板，进入如下界面：

![image][tmp]

下面是对一些选项的解释：

- 提交方式：选择调度脚本
- 资源选择：选择空闲节点较多、排队作业较少的队列可以使程序尽快运行（ps：早上七点到九点之间大家跑了一晚上的任务基本都结束了，早起的同学可以一个人爽用几十个节点）
- 工作目录：存放运行日志和输出文件的目录，建议在用户目录下自行选择
- 调度脚本：略



# FaTiDo

``FaTiDo``的一个提交脚本示例如下：

```bash
#!/bin/sh
#SBATCH -J test
#SBATCH -p hfacnormal02
#SBATCH --nodes=2              #number of nodes
#SBATCH --ntasks-per-node=4    #MPI task per node
#SBATCH --cpus-per-task=16     #threads per task
#SBATCH -o out%j.log
#SBATCH -e err%j.log
#SBATCH -t 06:00:00


export OMP_NUM_THREADS=16    # theards per processes
export OMP_SCHEDULE=dynamic  # balance between threads
export OMP_PROC_BIND=true    # bind process to CPU
export I_MPI_PIN_DOMAIN=omp  # start the OMP in MPIs
export PYTHONUNBUFFERED=true # set this to not buffering for python or use python -u ***.py


PARAMETERS_FILE="/path/to/input/file"  # replace with the actual path

mpirun -np 8 python3 /public/share/ac58qn21ek/share_local/programs/Fatido_Smilei/FaTiDo_mpi.py "$PARAMETERS_FILE"
```

和``Smilei``类似，将``PARAMETERS_FILE``改为输入文件的路径即可。输入文件的一个示例如下：

```python
# inputParameters.py

import numpy as np 
from scipy.constants import c, e, m_e, epsilon_0

#############################################################################
R_obs                     = 0.1         # distance from origin to far field detector plane*/
species                   = 'eon'      # specie name to process, same as that defined in FBPIC input script
N_coord                   = 3

############## Detector theta-phi plane #######################
N_phi                     = 1           # number of observation direction along azimuthal direction
N_theta                   = 1        # number of observation direction along radial direction        
theta_min                 = np.pi / 2.
theta_max                 = np.pi / 2.       
phi_min                   = 0           #  minimum of phi in rad */
phi_max                   = 0     # maximum of phi in rad */
 
############## Detector time axis ##############################
t_propagation             = R_obs/c    # time needed for the light to propagate from origin to far field detector plane
t_obs_start               = t_propagation;      # Start of the retarded time Unit:[s].*/
dt_obs                    = 0.1109e-16;       # Timestep to record electrical field, Unit:[s]. This is associated with PI/omega_max */
Nt_obs                    = 9000       # number of record timesteps. Notice users need to make sure  t_start <= >t_part_start-vec{n}*vec{r}/c, t_end >= t_part_end-vec{n}*vec{r}/c, where t_end= t_start+Nt_obs*dt_obs, t_part_start and t_part_end are start time and end time of input trace .

############## Input and output #################################
path_to_data = '/public/home/xuanwu/usr/smilei_tasks/test/TrackParticlesDisordered_eon.h5'             
outputFileName = 'Te_1e-1_Ti_2e-2_Z_1_ne_1e17_alpha_2.h5'     # output far field electric field file

################### units of Smilei ouput properties ############
nteon                     = 1
n0                        = 1.114855e+27
weight_unit               = 1
l0                        = 4.18879e-6      # um, unit[m]
x0                        = 0.5*l0  # laser center and electron collides at 6.4um 
y0                        = 0.5*l0   # 
z0                        = 0.5*l0
t_track_start             = 0    #6.5*lambda0/c  Actually, it doesn’t make sense. The numerical calculation starts from the start time of diag_particle.


#############################################################################
```

此处``path_to_data``需改为电子轨迹文件的路径。

