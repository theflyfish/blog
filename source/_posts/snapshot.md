title: 使用带权计数算法对快照机制优化

date: 2016-1-15 20:10:33 

tags: 

---
## 1. 背景

当前快照机制使用世代引用计数算法(generational referencecounting)，来判定对象是否需要删除。
但该算法无法实现legder对象空间复用，就导致了如果快照个数达到最大值，即使删除已创建快照，也无法创建新快照。
当前每次删除快照时，每删除一个对象都需要操作一个ledger记账对象，导致快照删除的性能很差
现在考虑使用带权计数算法进行优化，旨在解决ledger对象复用的问题，并尝试根据新的算法解决快照性能的不足。

## 2. 基于带权算法的快照机制

1. 每个数据对象创建后都有一个固定的总权重TW(total weight)，原始权重OW(original weight).最初时TW=OW=2^63，TW与OW值由ledger对象来管理
2. 每个引用该对象的VDI都会维护一个对应的权重PW（partial weight），由于每个VDI由很多对象构成，会在inode中记录一个PW数组，以下以单个对象为例进行说明。
3. 第一个VDI创建时，OW会传递1/2的权重到该VDI，即此时OW=2^62,PW1=2^62
4. 每创建一个VDI，该VDI都会得到父VDI 1/2的权重，同时父VDI的权重变为原来的1/2
![snapshot](/images/Image-1.png)
5. 依此类推，在基于原始对象创建一个新的VDI（snapshot或者clone）时，当前VDI会将自身持有权重PW的1/2传递到下一个VDI中

6. VDI删除时，操作ledger对象 将这个VDI的权重加回到OW中
7. 当TW==OW时，代表该对象已经没有VDI引用，触发删除操作
8. 以上便是整个算法的骨架，该算法维护这样一个核心恒等式： TW =OW+(PW1+PW2+....+PWn), PWn表示第N个引用该对象的VDI的权重

## 3. 边界处理与OW权重池
### 边界问题 
+ 如果一个VDI-x的权重已经为1,若还要基于其创建新VDI-y，需要判断ledger对象，进行如下流程：
  - 如果OW不为1，说明还有多余的权重可以分配，PW(x)=OW/2+PW(x), 从OW中分取一半的权重，继续进行后续的VDI创建流程.  
  - 如果OW为1, 说明该对象权重已经全部用完，无法继续分配。
   
### 应对方案
+ 针对此问题有两个方案可以解决:
  - 通过增加总权重TW的方式增加可用权重
  - 创建新的权重对象OW2。
+ 此处参考相关资料使用间接块的想法，初步设计采用权重池的方式

###权重池机制
1. 首先在ledger中维护32（个数待定）个OW权重数组OW[32]，每个OW初值均为TW
2. 每个引用该对象的VDI，维护两个变量，generation与PW值，generation代表该权重所属的OW在OW数组中的位置。
3. 开始创建第一个vdi时使用OW[0]作为原始权重，generation=0，按照第二节的规律后续的vdi依次创建
4. 回到上面的问题，当某个VDI-x权重为1且需要基于该VDI进行创建新VDI-y，如果OW[0]为1，已经无法分配权重，此时我们进行如下操作：
  + VDI-x将之前分配得到的权重还回OW[0]
  + 选择OW[1]重进行分配,此时PW(x)= OW[1]/2；当VDI-x得到足够的权重，便可继续进行VDI创建
  + 更新VDI-x的generation=1；
5. 当VDI删除时，根据自身的generation将PW还回到对应的OW中
6. 当整个ledger数组的OW都等于TW时，该对象已经么有引用，触发删操作

## 4. 结论讨论
1. snapshot的原来的机制采用世代引用计数算法(GRC)，带权引用计数算法（WRC）与之对比如下：
  ![snapshot](/images/Image-2.png)
2. 针对该方案，进行初步理论值统计  
  - MIN：在最差的情况下能够使用的快照个数。最差情况是指对同一个VDI只进行连续的快照操作，此时权重的消耗速度最快，判断为目前最差的情况,此种情况下总的快照次数计算公式为
		> sum =n（n+1）/2,  n为总权重的幂数  

  - MAX:  整个VDI即有clone也有snapshot，这个VDI以二叉树的方式全部铺开（参考意义不大）
![snapshot](/images/Image-3.jpg)

3. 当权重为1的VDI需要打快照时，就会导致快照创建时也会操作记账对象
	> 以2^31为TW总权重为例，在最差MIN情况下，每465快照中有30次快照，创建时会操作记账对象

4. 该优化方案，对代码的改动面较大，改动过程中需要考虑的比较全面，有出现新问题的风险，在做优化之前先把细节想清楚

## 5. 基于WRC的快照性能优化（后续）

1. 对于4T的VDI进行快照，若此时权重为1需要跟记账对象通信，按照每个对象4M计算，最差情况下大约有1M次的socket通信，可能会导致本次快照时间过长，快照的创建性能有会有波动。而且现在的机制中每个对象都对应一个ledger，在删除过程中最差情况会有1M次的socket通信，快照的删除性能比较差。
2. 上述提到的问题，有两个优化方案
 + 由上一章节的理论数据可知，当ledger大小为1个字节时，数据对象支持快照个数可达4960个，如果这个个数已经达到要求的话，根据这一点我们可以用一个ledger对象管理一个VDI的所有对象，ledger中的每个字节与VDI中数据对象一一对应。对于4T的数据盘而言，ledger对象为4M大大减少了创建跟删除时的网络通信以及ledger对象的创建所耗费的时间。会对性能有很大的提升
 + 第二个方案是一个ledger对象管理连续index的多个数据对象，通过hash的方式进行对象定位，通过这种方式，删除时理想情况下每删除256个对象，操作一次ledger对象

## 6. 参考文献

Garbage Collection Algorithms For Automatic Dynamic Memory Management -Richard Jones   
《垃圾收集》英文译- Richard Jones    
<https://en.wikipedia.org/wiki/Reference_counting>   
<http://www.cse.scu.edu/~jholliday/COEN317S06/DC3Naming.ppt>   
<http://www.cs.nyu.edu/goldberg/pubs/gold89.pdf>   
<http://students.depaul.edu/~cbaier1/435/ds-log.html>   
<https://www.researchgate.net/publication/220758907_Cyclic_Weighted_Reference_Counting_without_Delay>   

