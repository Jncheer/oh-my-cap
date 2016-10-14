# CAP 反演

## 权重文件

运行 `example/` 下的脚本 `weight.pl` 即可生成相应的权重文件 `example/20080418093658/weight.dat`：

    perl weight.pl 20080418093658

文件的内容是：

      IU_WCI      140 1 1 1 1 1  20.2   0.0
      NM_BLO      140 1 1 1 1 1  20.4   0.0
     NM_SIUC      145 1 1 1 1 1  21.2   0.0
      NM_SLM      210 1 1 1 1 1  29.0   0.0
      NM_FVM      235 1 1 1 1 1  31.8   0.0
      IU_WVT      260 1 1 1 1 1  35.0   0.0
     NM_PVMO      280 1 1 1 1 1  37.7   0.0
      IU_CCM      300 1 1 1 1 1  40.3   0.0
      NM_MPH      415 1 1 1 1 1  54.2   0.0

也就是说格式为：

    station_name dist w1 w2 w3 w4 w5 tp ts

第一列是台站的名称，需要和 SAC 文件名(后缀除外)一致。
第二列是震中距，gCAP 在反演时，会依据震中距去找对应的格林函数文件(dist.grn.?)。真实的震中距显然不会都是恰好 5 公里的倍数。
这里是做了简化，好处是事先算好格林函数，避免每一个地震都要算一次格林函数的麻烦。
官方例子的权重文件，也做了这样的简化，说明在 gCAP 的作者 Prof. Zhu 看来地震定位不准对震源机制的计算没有太大影响。
后面五列的 1 是 PnlZ、PnlR、Z、R、T 五个震相的权重，这里默认全部为1。
如果 tp 是正数，tp 表示 Pnl 波的到时，这里用的是 taup 标记的理论到时 T0。如果你没有安装 taup，这里会显示 0。即使这样，请继续阅读下去，到时并不是必须的。
ts 是 S 波实际的到时减理论的到时。
如果 w2 是 -1，就表示对应的台站是远震台，并且只使用 P (PnlZ) 和 SH (T)，这时候 ts 表示 S 波的到时。

## 时窗截取

CAP 会把完整的记录分成两个时间窗口：Pnl 和面波窗口。窗口的定义方式如下：

1. 用 SAC 文件 Z 分量的 t1 和 t2，以及 T 分量的 t3 和 t4 依次表示 Pnl 的起点、终点、面波的起点和终点。
2. 如果 1 中的 SAC 头段没有标记，又在反演时用 -V 选项给了视速度，则会用下面的公式计算 t1、t2、t3 和 t4：

    t1 = dist/vp - 0.2*m1, t2 = ts

    t3 = dist/vLove - 0.3*m2, t4 = dist/vRayleigh + 0.7*m2

3. 如果 1 和 2 都不行，则使用下面的公式计算(m1 和 m2 决定了 Pnl 和面波时间窗口的最大长度，用 -T 选项输入)：

    t1 = tp - 0.2*m1,  t2 = ts

    t3 = ts - 0.3*m2,  t4 = t3+m2

上述公式内的 dist 是震源位置到台站的直线距离，程序在实际计算时是等于 sqrt( distance * distance + depSqr)，其中 distance 是震中距，而depSqr 猜测应该是深度的平方，但是程序实际是把 depSqr 固定为 25。tp 和 ts 是用的格林函数文件中的值，而不是权重文件中的值。

这里，使用的是方法 3。

## 反演

在 `example/` 下执行命令：

    cap.pl -H0.2 -P0.3 -S5/10/0 -T35/70 -D2/1/0.5 -C0.05/0.3/0.02/0.1 -W1 -X10 -Mhk_15/5.0 20080418093658

### 结果解释

照上面的命令执行后，会看到这样的输出：

    20080418093658 15 1
    Warning: flag=21 => the minimum 295.5/90.0/  1.4 is at boundary
    inversion done

第二行的警告是因为 NM_MPH 台的位置很特殊，刚好在节面上，可以打开 sac，看看这个台的波形可以看到其 P 波非常弱。
可以将权重文件中 NM_MPH 台的五个权重全部改为 0，警告就会消失。
如果没有看到第三行 inversion done 这样的提示，说明并没有进行反演，多半是因为忘记生成了权重文件。

打开 `example/20080418093658/hk_15.ps`，可以看到反演的结果。
机制解右旁的文字是说，事件为 20080418093658，模型和深度是hk、15公里。
机制解(FM)的参数是295 90 1，即走向是 295 度，倾角是 90 度，滑移角是 1 度，说明这是一个走滑断层。震级为 5.21 Mw。
残差为 2.211e-2。下面的每一排代表一个台站的拟合情况，台站名下方的两个数字是震中距和偏移时间，如 IU_WCI 下面的 137.5/-1.89 表示震中距是 137.5 公里，波形向前移动了 1.89 秒，每一列震相下面的数字是拟合度和该震相的波形偏移。波形图中红色的理论波形，黑色的是实际的波形。

### 反演多个深度的结果

在 `example/` 下执行：

    perl inversion.pl 20080418093658

这个脚本会反演多个深度的结果。然后可以求出最佳深度和误差估计：

    perl get_depth.pl 20080418093658