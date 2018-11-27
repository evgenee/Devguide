# 控制器图解

本节包括PX4主要控制器的图解。

图解使用标准的 [PX4 符号](../contribute/notation.md) (附有详细图例注解)。

## 多旋翼位置控制器

![MC Position Controller Diagram](../../assets/diagrams/px4_mc_position_controller_diagram.png)

<!-- The drawing is on draw.io: https://drive.google.com/open?id=13Mzjks1KqBiZZQs15nDN0r0Y9gM_EjtX
Request access from dev team. -->

* 状态估计来自[EKF2](../tutorials/tuning_the_ecl_ekf.md)模块。
* 这是一个标准的位置-速度级联控制回路。
* 在某些模式，外环(位置回路) 可能会被绕过 (图中在外环之后增加一个多路开关来表示)。 只有在位置保持模式或某轴无速度请求时，位置回路才会发挥作用。
* 内环 (速度回路) 控制器使用箝位法对积分器做了抗饱和处理 (ARW)。

## 固定翼姿态控制器

![FW Attitude Controller Diagram](../../assets/diagrams/px4_fw_attitude_controller_diagram.png)

<!-- The drawing is on draw.io: https://drive.google.com/file/d/1ibxekmtc6Ljq60DvNMplgnnU-JOvKYLQ/view?usp=sharing
Request access from dev team. -->

姿态控制器由回路级联的方法实现。 外环计算姿态设定值和姿态估计值的误差，然后乘以增益 (比例控制器)，得到角速率设定值。 内环计算角度率误差并使用 PI (比例+积分) 控制器计算角加速度。

然后可以根据期望的角加速度和系统先验信息，通过控制分配 (又叫混控)，计算出执行机构 (副翼，水平尾翼，垂直尾翼，等) 的角偏移量。 另外，由于气动控制面的效率与速度正相关，因此控制率 - 一般在巡航速度下调参 - 按照空速测量值缩放刻度因子 (如果使用了空速传感器的话)。

> **Note** 如果没有安装空速传感器，固定翼姿态控制的增益调整将被禁用 (它是开环的)；您将无法在 TECS (全能量控制系统) 中使用空速反馈。

前馈增益用于补偿空气动力阻尼。 基本上，绕机体轴的两个主要力矩分量分别来自：控制翼面 (副翼，水平尾翼，垂直尾翼 - 驱动机体转动) 和 空气动力阻尼 (与机体角速率成正比 - 阻止机体转动) 。 为了保持恒定的角速率, 可以在速率回路中使用前馈来补偿这种气动阻尼。

滚转和俯仰控制器具有相同的结构，并且假定纵向和侧向气动力相互独立，没有耦合。 但是，为了将飞机侧滑产生的侧向加速度最小化，偏航控制器利用转向协调约束产生偏航速率设定值。 偏航速率控制器同样有助于抵消偏航效应带来的负面影响 (https://youtu.be/sNV_SDDxuWk) 并且可以提供额外的方向阻尼以减小 [荷兰滚效应 (十年没见过这个词了，好激动)](https://en.wikipedia.org/wiki/Dutch_roll)。

### 空速刻度化

本节的目的是：通过公式来解释怎样根据空速调整角速率回路 (PI) 和前馈控制器 (FF) 的输出，以及为何如此。 我们首先给出简化的滚转轴线性力矩方程，然后说明空速对力矩产生的直接影响，最后是空速对匀速滚转运动的影响。

如上所示的固定翼姿态控制器，角速率控制器输出角加速度设定值，传递给控制分配器 (这里叫“混控”)。 为了达到期望的角加速度，混控必须利用气动控制面 (例如：典型的飞机有两个副翼，两个水平尾翼和一个垂直尾翼) 产生力矩。 气动控制面产生的力矩受以下因素影响最大：飞机的相对空速和空气密度，更准确的说，是气动压力。 如果没有针对空速的刻度化处理，在某一特定巡航速度下调参的控制器，将会使飞机在高速下发生振荡，或者在在低速下达不到理想的随动效果。

读者们首先必须明白 [真实空速 (TAS)](https://en.wikipedia.org/wiki/True_airspeed) 和 [指示空速 (IAS)](https://en.wikipedia.org/wiki/Indicated_airspeed) 这二者的读数差别很大，除非你在海平面上飞行。

气动压力的定义是

$$\bar{q} = \frac{1}{2} \rho V_T^2$$,

$$\rho$$ 代表空气密度，$$V_T$$ 代表真实空速 (TAS)。

以滚转轴为例，带量纲的滚转轴力矩可以表示为：

$$\ell = \frac{1}{2}\rho V_T^2 S b C_\ell = \bar{q} S b C_\ell$$,

$$\ell$$ 代表滚转力矩，$$b$$ 代表飞机翼展，$$S$$ 代表参考面。

无量纲的滚转力矩系数 $$C_\ell$$ 可以通过通过以下几个系数建模得到：副翼效率系数 $$C_{\ell_{\delta_a}}$$，滚转阻尼系数 $$C_{\ell_p}$$ 和二面角系数 $$C_{\ell_\beta}$$。

$$C_\ell = C_{\ell_0} + C_{\ell_\beta}\:\beta + C_{\ell_p}\:\frac{b}{2V_T}\:p + C_{\ell_{\delta_a}} \:\delta_a$$,

$$\beta$$ 代表侧滑角，$$p$$ 代表滚转角速率，$$\delta_a$$ 代表副翼偏转角。

假设一架飞机对称 ($$C_{\ell_0} = 0$$) 且无侧滑 ($$\beta = 0$$) ，上面的方程就可以简化到只有滚转率阻尼和副翼产生的滚转力矩。

$$\ell = \frac{1}{2}\rho V_T^2 S b \left [C_{\ell_{\delta_a}} \:\delta_a + C_{\ell_p}\:\frac{b}{2V_T} \: p \right ]$$.

刚才推导出的这个最终方程，将会作为后面两个小节的基线。

#### 静态力矩 (PI) 刻度化

At a zero rates condition ($$p = 0$$), the damping term vanishes and a constant - instantaneous - torque can be generated using

$$\ell = \frac{1}{2}\rho V_T^2 S b \: C_{\ell_{\delta_a}} \:\delta_a = \bar{q} S b \: C_{\ell_{\delta_a}} \:\delta_a$$.

Extracting $$\delta_a$$ gives

$$\delta_a = \frac{2bS}{C_{\ell_{\delta_a}}} \frac{1}{\rho V_T^2} \ell = \frac{bS}{C_{\ell_{\delta_a}}} \frac{1}{\bar{q}} \ell$$,

where the first fraction is constant and the second one depends on the air density and the true airspeed squared.

Furthermore, instead of scaling with the air density and the TAS, it can be shown that the indicated airspeed (IAS, $$V_I$$) is inherently adjusted by the air density since at low altitude and speed, IAS can be converted to TAS using a simple density error factor

$$V_T = V_I \sqrt{\frac{\rho_0}{\rho}}$$,

where $$\rho_o$$ is the air density as sea level, 15°C.

Squaring, rearranging and adding a 1/2 factor to both sides makes the dynamic pressure $$\bar{q}$$ expression appear

$$\bar{q} = \frac{1}{2} \rho V_T^2 = \frac{1}{2} V_I^2 \rho_0$$.

We can now easily see that the dynamic pressure is proportional to the IAS squared

$$\bar{q} \propto V_I^2$$.

The scaler previously containing TAS and the air density can finally be written using IAS only

$$\delta_a = \frac{2bS}{C_{\ell_{\delta_a}}\rho_0} \frac{1}{V_I^2} \ell$$.

#### Rate (FF) scaling

The main use of the feedforward of the rate controller is to compensate for the natural rate damping. Starting again from the baseline dimensional equation but this time, during a roll at constant speed, the torque produced by the ailerons should exactly compensate for the damping such as

$$- C_{\ell_{\delta_a}} \:\delta_a = C_{\ell_p} \frac{b}{2 V_T} \: p$$.

Rearranging to extract the ideal ailerons deflection gives

$$\delta_a = -\frac{b \: C_{\ell_p}}{2 \: C_{\ell_{\delta_a}}} \frac{1}{V_T} \: p$$.

The first fraction gives the value of the ideal feedforward and we can see that the scaling is linear to the TAS. Note that the negative sign is then absorbed by the roll damping derivative which is also negative.

#### Conclusion

The output of the rate PI controller has to be scaled with the indicated airspeed (IAS) squared and the output of the rate feedforward (FF) has to be scaled with the true airspeed (TAS)

$$\delta_{a} = \frac{V_{I_0}^2}{V_I^2} \delta_{a_{PI}} + \frac{V_{T_0}}{V_T} \delta_{a_{FF}}$$,

where $$V_{I_0}$$ and $$V_{T_0}$$ are the IAS and TAS at trim conditions.

Finally, since the actuator outputs are normalized and that the mixer and the servo blocks are assumed to be linear, we can rewrite this last equation as follows

$$\dot{\mathbf{\omega}}*{sp}^b = \frac{V*{I_0}^2}{V_I^2} \dot{\mathbf{\omega}}*{sp*{PI}}^b + \frac{V_{T_0}}{V_T} \dot{\mathbf{\omega}}*{sp*{FF}}^b$$,

and implement it directly in the rollrate, pitchrate and yawrate controllers.

#### Tuning recommendations

The beauty of this airspeed scaling algorithm is that it does not require any specific tuning. However, the quality of the airspeed measurements directly influences its performance.

Furthermore, to get the largest stable flight envelope, one should tune the attitude controllers at an airspeed value centered between the stall speed and the maximum airspeed of the vehicle (e.g.: an airplane that can fly between 15 and 25m/s should be tuned at 20m/s). This "tuning" airspeed should be set in the [FW_AIRSPD_TRIM](../advanced/parameter_reference.md#FW_AIRSPD_TRIM) parameter.