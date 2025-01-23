# CPA_MIA_analysis
## 说明：
* 本实验需要提前安装MATLAB R2024a - academic use（或其它可用版本）
* 本实验所需数据存储于``data.mat``中
> ***注意：请确保``data.mat``文件位于MATLAB的Current Folder下***
* 本实验结果存储于``output_CPA.mat``，``output_MIA_other.mat``，``output_MIA_p_1.mat``及``output_MIA_p_2.mat``中
> * 由于``output_MIA.mat``文件过大，故将该文件拆为三份
> * 原文件过大本质上是因为其中的``p``太大，所以将``p``拆为``p_1``和``p_2``
> * 若要恢复``p``，可以使用如下语句：
> ```matlab
> p = cat(4, p_1, p_2);
> ```
> * 若要检测``p``是否恢复为原大小（``256*2000*9*8``），可以使用如下语句：
> ```matlab
> size(p);
> ```

## 一、对无防护的AES算法进行CPA攻击
*实验代码请见``experiment_CPA.mlx``*
<div id="target-section01"></div>

### 1.1 实验目的、条件与环境
* 实验目的：获得AES算法密钥的第一个字节``ck``
* 实验条件：无防护AES算法。共使用1000条能量迹，每条2000点，一条能量迹对应一个明文第一字节。能量迹存储在``wave``（``1000*2000 int32``）中。
* 实验环境：MATLAB R2024a - academic use
### 1.2 实验思路
<div id="target-section02"></div>

* 从``data.mat``文件导入数据，并转换类型为``int32``（便于后续求汉明重）
> 我上传的数据被事先转换成了``int32``类型  
```matlab
load("data.mat");
d = int32(d);
s = int32(s);
wave = int32(wave);
```
* 将1000条明文第一字节``d``（``1000*1 int32``）与256种可能的密钥第一字节``k``（即0~255，亦即``00~FF``）送入S盒运算，得到中间结果``v``（``1000*256 int32``）
```matlab
for k = 0:255
    for i = 1:1000
        v(i, k+1) = s(bitxor(d(i), k)+1);
    end
end
```
* 求出S盒运算出的中间结果``v``的汉明重``hw``（``1000*256 int32``）
> 之前已将数据转换为``int32``类型，故这里可以使用``mod``与``idivide``函数
```matlab
hw = zeros(1000, 256, 'int32');
%1000: D
%256: K
for i = 1:8
    hw = hw + mod(v, 2);
    v = idivide(v, 2);
end
```
* 求出各密钥第一字节对应的hw（``hw``中的一个列向量）与各时刻实测能量迹的值（``wave``中的一个列向量）的相关系数（correlation coefficient），存于``r``（``256*2000 double``）中
>  * 注意：``corrcoef``函数的参数应当为``double``类型
>  * ``r``中第``k+1``行对应于密钥第一字节``k``，第``t+1``列对应于时刻``t``
```matlab
for i = 1:256 %k+1
    for j = 1:2000 %t+1
        R = corrcoef(double(hw(:, i)), double(wave(:, j)));
        r(i, j) = R(1, 2);
    end
end
```
* 求出``r``的绝对值``r_abs``，便于后续求最大值、作图
```matlab
r_abs = abs(r);
```
* 找出``r_abs``中的最大值(``M``)及对应的行（``row``）与列（``col``），进而算出正确的密钥第一字节（``ck``）与对应时刻（``ct``）
```matlab
[M, I] = max(r_abs, [], "all");
[row, col] = ind2sub(size(r_abs), I);
ck = row-1;
ct = col-1;
```
* 把实验结果保存在``output_CPA.mat``文件中
```matlab
save("output_CPA.mat");
```
* 绘图，用曲面形式展现``r_abs``矩阵，并标出最大值位置（即``ck``与``ct``对应的位置）
```matlab
[X, Y] = meshgrid(0:1999, 0:255);
surf(X, Y, r_abs);
hold on;
xlabel('time');
ylabel('key');
zlabel('correlation coefficient(abs)');
title('Result CPA');
hold on;
scatter3(ct, ck, M);
hold off;
```
* ***注意：在MATLAB中，索引从1开始!*** 但在求``ck``与``ct``及绘图时，密钥第一字节和时刻是从0开始的。
### 1.3 实验结果
* ``ck = 101``
* ``result_CPA.png``
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/16e99a0a1a094d69a90a45f6854061f3.png#pic_center)

## 二、对无防护的AES算法进行CPA攻击
*实验代码请见``experiment_MIA.mlx``*
### 2.1 实验目的、条件与环境
[同上](#target-section01)
### 2.2 实验思路
[前三步同上](#target-section02)，即完成求中间结果``v``的汉明重``hw``
* 求出``hw``与``wave``的联合概率分布``p``（``256*2000*9*8 double``）及各自的边际概率分布``phw``（``256*9 double``）和``po``（``2000*8 double``）
> * 对于联合概率分布，要分密钥第一字节``k``与时刻``t``讨论，在确定密钥第一字节与时刻的前提下求出``hw``与	``wave``的联合概率分布。**因此，``p``前两个维度代表``k``与``t``，后两个维度代表``hw``与``wave``（``o``)**
> * 对于``hw``和``wave``各自的边际概率分布，则需要相应地分密钥第一字节或时刻讨论。**因此，``phw``两个维度代表``k``和``hw``，``po``两个维度代表``t``和``wave``（``o``）**
> 
> ***
> ***重点——将连续的``wave``切分为离散的``o``***：需要在``wave``的取值范围上人为分出多个区间。本实验通过参照``wave``的图像确定区间的划分（如下图所示）
> ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c37b1581af6b4813af6f40ab2e3600b1.png#pic_center)
> * 在本实验中，将``wave``元素取值在$[-3.5 \times 10^4,0.5 \times 10^4]$区间上等分为8段。
> * 实现对``wave``的切分，要依靠下方代码中关于``o``的循环与条件控制（通过寻找合适的``o``找到``wave``中元素所属的区间）
```matlab
p = zeros(256, 2000, 9, 8);
phw = zeros(256, 9);
po = zeros(2000, 8);

for k = 0:255
    for t = 0:1999
        for n = 1:1000
            for o = 1:8
                if ((o-8)*5000 <= wave(n,t+1)) && ((o-7)*5000 > wave(n,t+1))
                    p(k+1,t+1,(hw(n,k+1)+1),o) = p(k+1,t+1,(hw(n,k+1)+1),o) + 1;
                end
            end
        end
    end
end
p = p/1000;

for k = 0:255
    for n = 1:1000
        phw(k+1,(hw(n,k+1)+1)) = phw(k+1, (hw(n,k+1)+1)) + 1;
    end
end
phw = phw/1000;

for t = 0:1999
    for n = 1:1000
        for o = 1:8
            if ((o-8)*5000 <= wave(n,t+1)) && ((o-7)*5000 > wave(n,t+1))
                po(t+1,o) = po(t+1,o) + 1;
            end
        end
    end
end
po = po/1000;
```
* 计算互信息``I``（``256*2000 double``）
> 互信息计算公式：$I(X;Y)= \sum \limits _{x} \sum \limits _{y} p(x,y) \log {\frac {p(x,y)} {p(x)p(y)}}$
> * $\log$底数的选择主要影响单位，而不改变互信息的相对大小
> * 本实验选择以bit为单位，故**底数选择2**
> 
> ***
> ***注意：要考虑``p``，``phw``和``po``的值是否为0！***
> * 在``p``，``phw``或``po``为0的情况下，对应项无意义，不参与求和（即跳过该项）
> * 由于``phw``或``po``为0时``p``必然为0，故只需判断``p``是否为0
```matlab
I = zeros(256, 2000);
for k = 0:255
    for t = 0:1999
        for lhw = 0:8
            for o = 1:8
                if p(k+1,t+1,lhw+1,o) ~= 0
                    temp = p(k+1,t+1,lhw+1,o) / (phw(k+1,lhw+1) * po(t+1,o));
                    I(k+1,t+1) = I(k+1,t+1) + p(k+1,t+1,lhw+1,o) * log(temp) / log(2);
                end
            end
        end
    end
end
```
* 找出``I``中的最大值(``M``)及对应的行（``row``）与列（``col``），进而算出正确的密钥第一字节（``ck``）与对应时刻（``ct``）
```matlab
[M, IND] = max(I, [], "all");
[row, col] = ind2sub(size(I), IND);
ck = row-1;
ct = col-1;
```
* 把实验结果保存在``output_MIA.mat``文件中
```matlab
save("output_MIA.mat");
```
* 绘图，用曲面形式展现``I``矩阵，并标出最大值位置（即``ck``与``ct``对应的位置）
```matlab
[X, Y] = meshgrid(0:1999, 0:255);
surf(X, Y, I);
hold on;
xlabel('time');
ylabel('key');
zlabel('mutual info');
title('Result MIA');
hold on;
scatter3(ct, ck, M);
hold off;
```
### 2.3 实验结果
* ``ck = 101``
* ``result_MIA.png``
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/778e9743de444775a9c2007f18fd47a1.png#pic_center)

