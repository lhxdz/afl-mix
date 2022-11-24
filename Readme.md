# Mix Schedule

#以下为AFLFast原公告(中文为添加)

<a href="https://mboehme.github.io/paper/CCS16.pdf"><img src="https://mboehme.github.io/paper/CCS16.png" align="right" width="250"></a>
Power schedules implemented by Marcel Böhme \<marcel.boehme@acm.org\>. 
AFLFast is an extension of AFL which is written and maintained by 
Michal Zalewski \<lcamtuf@google.com\>.

**Update**: Checkout [AFL++](https://github.com/vanhauser-thc/AFLplusplus) which is actively maintained and implements AFLFast power schedules!

AFLFast is a fork of AFL that has been shown to outperform AFL 1.96b by an **order of magnitude**! It helped in the success of Team Codejitsu at the finals of the DARPA Cyber Grand Challenge where their bot Galactica took **2nd place** in terms of #POVs proven (see red bar at https://www.cybergrandchallenge.com/event#results). AFLFast exposed several previously unreported CVEs that could not be exposed by AFL in 24 hours and otherwise exposed vulnerabilities significantly faster than AFL while generating orders of magnitude more unique crashes. 

Essentially, we observed that most generated inputs exercise the same few "high-frequency" paths and developed strategies to gravitate towards low-frequency paths, to stress significantly more program behavior in the same amount of time. We devised several **search strategies** that decide in which order the seeds should be fuzzed and **power schedules** that smartly regulate the number of inputs generated from a seed (i.e., the time spent fuzzing a seed). We call the number of inputs generated from a seed, the seed's **energy**. 

We find that AFL's exploitation-based constant schedule assigns **too much energy to seeds exercising high-frequency paths** (e.g., paths that reject invalid inputs) and not enough energy to seeds exercising low-frequency paths (e.g., paths that stress interesting behaviors). Technically, we modified the computation of a seed's performance score (`calculate_score`), which seed is marked as favourite (`update_bitmap_score`), and which seed is chosen next from the circular queue (`main`). We implemented the following schedules (in the order of their effectiveness, best first   其中mix为fast,explore,quad三种策略的结合,是本人实验结果,且默认为mix能量分配策略(Mix Schedule)):

| AFL flag | Power Schedule             | 
| ------------- | -------------------------- |
| `-p fast` (default)| ![FAST](http://latex.codecogs.com/gif.latex?p(i)=\\min\\left(\\frac{\\alpha(i)}{\\beta}\\cdot\\frac{2^{s(i)}}{f(i)},M\\right))  |
| `-p coe` | ![COE](http://latex.codecogs.com/gif.latex?p%28i%29%3D%5Cbegin%7Bcases%7D%200%20%26%20%5Ctext%7B%20if%20%7D%20f%28i%29%20%3E%20%5Cmu%5C%5C%20%5Cmin%5Cleft%28%5Cfrac%7B%5Calpha%28i%29%7D%7B%5Cbeta%7D%5Ccdot%202%5E%7Bs%28i%29%7D%2C%20M%5Cright%29%20%26%20%5Ctext%7B%20otherwise.%7D%20%5Cend%7Bcases%7D) |
| `-p explore` | ![EXPLORE](http://latex.codecogs.com/gif.latex?p%28i%29%3D%5Cfrac%7B%5Calpha%28i%29%7D%7B%5Cbeta%7D) |
| `-p quad` | ![QUAD](http://latex.codecogs.com/gif.latex?p%28i%29%20%3D%20%5Cmin%5Cleft%28%5Cfrac%7B%5Calpha%28i%29%7D%7B%5Cbeta%7D%5Ccdot%5Cfrac%7Bs%28i%29%5E2%7D%7Bf%28i%29%7D%2CM%5Cright%29) |
| `-p lin` | ![LIN](http://latex.codecogs.com/gif.latex?p%28i%29%20%3D%20%5Cmin%5Cleft%28%5Cfrac%7B%5Calpha%28i%29%7D%7B%5Cbeta%7D%5Ccdot%5Cfrac%7Bs%28i%29%7D%7Bf%28i%29%7D%2CM%5Cright%29) |
| `-p exploit` (AFL) | ![LIN](http://latex.codecogs.com/gif.latex?p%28i%29%20%3D%20%5Calpha%28i%29) |
where *α(i)* is the performance score that AFL uses to compute for the seed input *i*, *β(i)>1* is a constant, *s(i)* is the number of times that seed *i* has been chosen from the queue, *f(i)* is the number of generated inputs that exercise the same path as seed *i*, and *μ* is the average number of generated inputs exercising a path.
  
More details can be found in our paper that was recently accepted at the [23rd ACM Conference on Computer and Communications Security (CCS'16)](https://www.sigsac.org/ccs/CCS2016/accepted-papers/).

PS: The most recent version of AFL (2.33b) implements the explore schedule which yielded a significance performance boost. We are currently conducting experiments with a hybrid version between AFLFast and 2.33b and report back soon.

PPS: In parallel mode (several instances with shared queue), we suggest to run the master using the exploit schedule (-p exploit) and the slaves with a combination of cut-off-exponential (-p coe), exponential (-p fast; default), and explore (-p explore) schedules. In single mode, the default settings will do. **EDIT:** In parallel mode, AFLFast seems to perform poorly because the path probability estimates are incorrect for the imported seeds. Pull requests to fix this issue by syncing the estimates accross instances are appreciated :)

Copyright 2013, 2014, 2015, 2016 Google Inc. All rights reserved.
Released under terms and conditions of Apache License, Version 2.0.

# mix策略(Mix Schedule)要述
## 背景：
经本人测试各个算法(EXPLOIT/AFL、EXPLORE、COE、LINEAR、QUAD、FAST)的结果进行分析发现各个策略有各个的优势，根据之前的实验，以及结合各能量分配策略的能量分配方式，产生了一些想法，或许不同的策略适合不同的被测试对象？那么是否可以让每个能量分配策略都有机会进行实践呢？于是设计了一款能量分配方式：设计一个转换器，根据一定的策略改变能量分配策略，我将这样的策略称之为Mix Schedule，混合策略。

## Mix Schedule中的核心策略选择：
根据测试，在我的测试结果中前三位的是FAST、COE、QUAD，我试过将每个策略都轮转的方式，反而降低了效率，所以仅采用这三个策略。

## 现有策略分析：
1. Exponential Schedule (FAST)

`p(i)=min⁡((α(i)/β)*(2^s(i) /f(i) ),M)`

其中α(i)是算法中assignEnergy的实现。s(i)表示种子ti之前从队列T中选到的次数。f(i)表示执行状态为i的生成的输入的数量。M则是能量的上限值。其中β>1。这其实是对COE的扩展,即当f(i)>μ时不再完全不对ti进行Fuzz处理。s (i)放在指数部分：期望的种子队列T本质上需要一个维护一个探索低密度区的输入序列，所以如果s(i)越大，直接含义上表示从输入队列中选择ti输入的次数越多，也就是说状态i达到的路径数越少，状态i处于低密度区，所以放在指数上，ti选取越多，就给它高能量值。

2. Cut-Off Exponential (COE)

当 f(i)>μ     `P(i)=0`

其他情况 `p(i)=min⁡((α(i)/β)*2^s(i) ,M)`

其中μ=∑i∈S+f(i)/∣S+∣，也就是说说μ是探索到的所有路径后生成数目数量的均值。其中α(i)是算法中assignEnergy的实现。s(i)表示种子ti之前从队列T中选到的次数。M则是能量的上限值。其中β>1。

3. Quadratic Schedule (QUAD)

`p(i)=min⁡((α(i)/β)*((s(i)^2)/f(i) ),M)`

其中α(i)是算法中assignEnergy的实现。s(i)表示种子ti之前从队列T中选到的次数。f(i)表示执行状态为i的生成的输入的数量。M则是能量的上限值。其中β>1。以二次方式增加状态i的能量，其中ti已从T中选择ti的次数s（i），与路径的模糊能量f（i）成正比。

## mix策略(Mix Schedule)
经过一系列的实验探索，mix策略最终定稿：
先运行4个小时的EXPLORE策略，一旦转换器监测到4个小时过去了，这立即切换到FAST策略，同时转换器实时监测，一旦发现FAST策略期间新路径的寻找效率低于0.05 paths/min 则切换能量分配策略为QUAD，当转换器实时监测到在QUAD策略期间新路径的寻找效率低于0.066 paths/min 则切换能量分配策略为FAST，如此反复。

## 结果
经测试，效率有提升：
FAST策略在运行1400分钟后新路径探索数大概在1572左右，Mix Schedule策略在运行1400分钟后新路径探索数大概在1750左右，Mix Schedule策略的新路径寻找比例比FAST策略高出11.3%。

## 原因：
mix策略选择这样的组合方式的根本原因是：
前四个小时使用EXPLORE对种子能量实现均匀分配，以增加路径探索的广度，四个小时候直接切换FAST策略，能量分配以s（i）的指数型增长（2^s(i)），但当转换器监测到新路径寻找速率低于0.05 paths/min时则切换到QUAD策略，让种子的能量分配呈现二次方级别的增长（s(i)^2），以寻求突破，一旦在QUAD策略期间监测到新路径寻找速率低于0.066 paths/min则又切换回FAST策略。
经测试，这样的组合方式暂时是我设计的测试对比模型中效果最好的，且在`-d`模式(Skip Deterministic模式)下和原本的Fast策略的`-d`模式相比没有劣化

## 待改进：
在`-d`模式(Skip Deterministic模式)下和原本的Fast策略的`-d`模式相比，效率相差无几，并没有效率的提升

## 代码指引
mix策略器主要在 afl-fuzz.c 中的 `void *change()`