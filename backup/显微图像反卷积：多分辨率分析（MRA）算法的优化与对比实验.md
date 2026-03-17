## 1. 实验背景与动机

在三维荧光显微成像中，为了提升图像的分辨率和对比度，多分辨率分析（MRA）的反卷积方法展现出了巨大潜力[^1]。本实验在基于最优稀疏表示的 Framelet + Curvelet 多分辨率去卷积方法的基础上尝试引入 UWT（非下采样小波变换）和 Shearlet（剪切波变换）等不同的多尺度、多方向稀疏表示方式。

为了实现这一目标，我修改了基于 FISTA 算法的代码，使其能够自适应不同变换的数据结构，实现了任意两种变换的组合，并支持批量化测试。

## 2. 单一变换的去噪与超分辨效果验证
使用29%光照强度下U2OS细胞内肌动蛋白的SIM图像做对比。保真项+单正则化约束
对比图
<img width="1000" alt="Image" src="https://github.com/user-attachments/assets/8cdd46ee-8822-4bfc-9407-35e33713c2a3" />
局部放大对比图
<img width="1000" alt="Image" src="https://github.com/user-attachments/assets/81ef82ce-84cd-4bbb-80fb-4016cbe41a98" />
各个算法结果沿红线上强度波形图
<img width="600"  alt="Image" src="https://github.com/user-attachments/assets/b29f7b33-808d-4059-b71a-b52a7bccff76" />
根据红蓝框计算的抗噪声评价指标
<img width="356" height="171" alt="Image" src="https://github.com/user-attachments/assets/2b1ab69b-d027-4777-bee1-531a4caf2e1f" />
<img width="600"  alt="Image" src="https://github.com/user-attachments/assets/e2c858e9-f005-4171-9f20-6dfc53222a6e" />


在 20%照明强度的数据集上的初步测试表明，所有的稀疏正则项均能有效地减少噪声。
* **Shearlet**：去噪能力最为显著，但计算复杂度高，运行时间较长。
* **Framelet、Curvelet 与 UWT**：均比原始图像展现出更高的分辨率和对比度（例如，在肌动蛋白的线轮廓分析中，能清晰地将一个波峰分辨为两个）。



## 3. 针对 Shearlet 变换的阈值权重优化

在实验中我发现，当 Shearlet 进行高层数（如 6 层）分解时，高频细节往往会被过度削弱。这与 FISTA 的阈值设计有关。原代码中，Lipschitz 常数 $L$ 仅与系统的点扩散函数 (PSF) 相关：

$$L=2\max(|S_{\text{big}}|^2)$$

算法在所有 Shearlet 子带上使用了统一的软阈值 $\theta=\lambda_2/L$。当层数增加时，变换系数被拆分到更多子带，绝对值变小，导致大量高频系数在统一阈值下被直接抹去。

**改进策略**：我引入了与尺度相关的加权阈值策略（如线性权重或 Log 权重）：

$$D_{\text{th}}=|\text{tmp}|-\frac{\lambda_2}{L}\times\text{scaleWeight}(j)$$

实验结果表明，该改进有效恢复了高频结构信息，使部分方法的定量指标（PSNR、SBR）得到显著提升。

## 4. 混合多分辨率变换对比测试

我进一步在不同信噪比条件下测试了双变换组合的效果：

### 常规条件（20% SIM）
* `Shearlet + Curvelet` 和 `Framelet + Shearlet` 组合表现出较少的背景噪声，高频细节恢复较好。
* 需要注意的是，`Shearlet + Curvelet` 在具备极高分辨能力（如分辨 Y 型分叉结构）的同时，存在较明显的伪影，且运行时间较长（约 34 秒）。

### 弱光低信噪比条件（3% SIM）
在极低照明下，算法的鲁棒性受到挑战：
* **结构分辨力**：`Framelet + Curvelet`、`UWT + Framelet` 以及 `UWT + Curvelet` 在此条件下依然能够成功分辨出紧邻的两条微管结构。
* **噪声控制**：包含 Shearlet 的组合（如 `Shearlet + Curvelet`、`Framelet + Shearlet` 等）虽然噪声抑制较好，但在此信噪比下由于平滑过度，无法分辨这两条微管。同时，相比于 `Framelet + Curvelet`，包含 UWT 的组合背景噪声显得更为明显。

## 5. 总结

本次实验通过改进 FISTA 的阈值策略和引入多种稀疏变换组合，验证了不同 MRA 策略在不同信噪比下的特性。`Framelet + Curvelet` 在弱光下保结构能力突出，而优化后的 `Shearlet` 相关组合在常规信噪比下展现出极佳的去噪与超分辨平衡。后续将继续针对复杂噪声环境下的稳定性进行探索。

---
## 参考资料

[^1]: Mandracchia, B., Liu, W., Hua, X., Forghani, P., Lee, S., Hou, J., Nie, S., Xu, C., & Jia, S. (2023). [Optimal sparsity allows reliable system-aware restoration of fluorescence microscopy images](https://doi.org/10.1126/sciadv.adg9245). *Science Advances*, 9(35), eadg9245.