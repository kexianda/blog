---
title: '[Paxos]读The Part-Time Parliament笔记(一)'
categories: 技术
tags:
- 分布式
- Paxos
date: 2017-09-02
---
前两年看过Paxos协议, 网上各种通俗化解释, 更晕. 不如自己看论文. 自己没有实践中实现Paxos, 看chubby论文就知道坑非常多. 但理论上的理解,论文《The Part-Time Parliament》其实讲的很清晰.
这不是Paxos介绍讲解什么的, 只是读论文做的笔记, 把数学证明过程再证一次.
<!--more-->
## 一致性约束文字描述
$\beta$是表决的集合, 满足一致性的约束条件如下:
B1($\beta$): $\beta$中的每一轮表决,都有一个唯一的编号
B2($\beta$): $\beta$中任意两轮表决的法定人数集中,至少有一个公共的牧师成员
B3($\beta$): 对于$\beta$中每一轮表决$B$,如果$B$的法定人数集中的任何一个牧师在$\beta$中一个更小
轮的表决中投过(赞成)票,那么$B$的法令与所有这些更小轮表决中的最大的那次表决的法令相同
## 符号化的定义
### 表决
$B$ : 一轮表决(a choice that is made by voting)
$B\_{dec}$ : 一条法令(decree)  
$B\_{qrm}$ : 一个牧师的非空集合(表决的法定人数集quorum)  
$B\_{vot}$ : 一个牧师的集合(所有对法令作出投票(赞成)的牧师).
$B\_{bal}$ : 这轮表决的编号  
说一轮表决B是成功的,当且仅当 $ B\_{qrm} \subseteq B\_{vot} $ , 一轮成功的投票是每一个法定人数
集的成员都投了赞成票.

### 投票
$ null \space vote$ 为: $\langle null\_p, - \infty , BLANK \rangle$
$v\_{pst}$ 为投票的牧师, $v\_{bal}$ 为本论表决的编号, $v\_{dec}$ 为所表决的法令.
$v\_{dec}$不一定是个有效的表决，是$v\_{pst}$对$v\_{dec}$的提议"点过赞" :-).
$Votes(\beta)$ 为所有在$\beta$中的投票.

$MaxVote(b, p, \beta): max(\\\{ v \in Votes(\beta) \vert (v\_{pst}=p) \land (v\_{bal} \lt b) \\\} \cup \{ null_p \})$
$Votes(\beta)$中由p牧师投出的表决编号小与b的最大投票或空投票$null\_p$.

$MaxVote(b, Q, \beta): max(\\\{ v \in Votes(\beta) \vert (v\_{pst} \in Q) \land (v\_{bal} \lt b) \\\} \cup \{ null\_p \})$
$Votes(\beta)$中由Q集合的牧师们投出的表决变化小与b的最大投票或空投票$null_p$.  

先把定义搞清楚了,这样理解三个约束条件的形式化和严格的数学证明比较简单了.

## 形式化的一致性约束
* $B1(\beta) \doteq \forall B,B' \in \beta : (B \neq B') \implies (B\_{bal} \neq B'\_{bal})$
* $B2(\beta) \doteq \forall B,B' \in \beta : B\_{qrm} \cap B'\_{qrm} \neq \emptyset$
* $B3(\beta) \doteq \forall B \in \beta :(MaxVote(B\_{bal}, B\_{qrm}, \beta)\_{bal} \neq - \infty) \implies (B\_{dec} = MaxVote(B\_{bal}, B\_{qrm}, \beta)\_{dec})$

## 引理及其证明思路
$\forall B,B' \in \beta : ((B\_{qrm} \subseteq B\_{vot}) \land (B'\_{bal} > B\_{bal})) \implies (B'\_{dec} = B\_{dec})$
引理其实就是约束条件的另一种表述. $(B\_{qrm} \subseteq B\_{vot}$表示B轮表决满足法定人数. 引理意思是, 一旦决议成功确定, 不再改动. 不然的话这种**抢占式**循环下去投票没法结束,进展性有问题了.

反证法不难得出结论, 思路是这样的：** 先假设个决议不同于B的决议的大于B轮最小轮次, 然后推导出决定C轮决议的那个MaxVote的轮次号大于B的轮次号，从而得出与假设矛盾的结论得证** .  
数学证明如下:
(1) 不妨假设C轮表决是满足决议不同于B的大于B轮次的最小轮,  
(2) C轮表决中的法定人数集合必然有某人q在B轮中投过票v, 根据MaxVote定义有: $MaxVote(C\_{bal}, C\_{qrm}, \beta)\_{bal} >= v\_{bal} = B\_{bal}$
(3)而$C\_{dec} = MaxVote(C\_{bal}, C\_{qrm}, \beta)dec \neq B\_{dec}$, 根据约束B1有: $MaxVote(C\_{bal}, C\_{qrm}, \beta)\_{bal} \neq v\_{bal} = B\_{bal}$. 去掉这个等于后，综合(2)得出：$MaxVote(C\_{bal}, C\_{qrm}, \beta)\_{bal} > B\_{bal}$
(4) 而由MaxVote定义，$C\_{bal} > MaxVote(C\_{bal}, C\_{qrm}, \beta)\_{bal}$
(5) 与$C_{bal}$为满足假设的最小轮次矛盾.  
证毕.

## 从一致性约束到协议
初级协议(The Preliminary Protocol)
(B1). 为了使 B1 成立，每一轮表决都必须获得一个唯一的编号
(B2). 为了使 B2 成立，多数牧师作为表决的法定人数集.
(B3). 一个牧师 p 在发起表决前，需要找出$MaxVote(b, Q, beta)\_{dec}$, 必须询问Q集合的每个牧师小于b表决, 于是有下面表决过程步骤(两阶段)：

1. p询问某些牧师, 我p准备发起编号b的表决了,p选择一个编号b,给某些牧师(最好>Q集合(quorum)的牧师发一条消息NextBallot(b);
2. q回答p，告知满足小于b的最大vote;
牧师q响应NextBallot(b), 发送LastVote(b,v)给p, v 是 p 的编号小于 b的表决。q没投过票就是$null\_{p}$
为了满足约束条件(B3), $MaxVote(b,q,\beta)$在q发送LastVote(b,v)后不能再改变. 而新表决的发起和投票的进行会改变
集合$\beta$, 所以q必须承诺在编号集合$[v\_{bal}, b]$中的表决过程中不再投票(q必须记录下v和b,来保证承诺);
3. p收到过半的回应中选编号最大的表决的投票(满足B3)，然后给Q(大多数牧师)发送BeginBallot(b,d), 开始表决过程;
4. 在收到 BeginBallot(b,d)消息后， 牧师 q 发现他发送的LastVote编号就是b，那么他回复投上一票。如果q发送的LastVote(b2, v), 编号不同, 为了维持B3约束不改变MaxVote，那么他决定不投赞成票;
5. 如果p收到Q(法定人数)的投票(Voted(b,q)),那么他记下决议d，发送success(d)给所有人;
6. 其他人收到success(d), 记下这个决议d.  

步骤(3)的执行可以认为是将一个表决B加入到集合$\beta$中,其中 $B\_{bal}=b,B\_{qrm}=Q,B\_{vot}= \emptyset, B\_{dec}=d$. 但此时还不会影响MaxVote(b,q,beta), 因为还没有人投票呢.
在步骤(4)中，如果牧师q决定在表决中投票，那么q加入B的投票者集合$\beta$,改变了表决集合$\beta$，这里$B \in \beta$.

后面的基本协议(The Basic Protocal)是继续加强约束.  完整的神会协议(The Complelte Synod Protocal)呢,就是改善进展性(progress). Multi-Decree Parliament等等(to be continued...)
(Hexo对LaTex不友好,十一折腾一阵搞定LaTex的支持)
