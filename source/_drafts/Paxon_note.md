---
title: 'The Part-Time Parliament笔记'
categories: 技术
tags:
- 分布式
date: 2017-09-02
---

<script type="text/javascript" src="https://cdn.bootcss.com/mathjax/2.7.1/latest.js?config=default">
</script>

The Part-Time Parliament Note
Ballot:
1. a document listing the alternatives that is used in voting // 投票用的选票
2. a choice that is made by voting //一轮表决

前两年看过paxos协议, 网上各种通俗化解释, 更晕. 再看论文《The Part-Time Parliament》, paxos协议其实很清晰.

## 定义

\\(B\\) : 一轮表决(a choice that is made by voting)
\\(B_{dec}\\) : 一条法令(decree)  
\\(B_{qrm}\\) : 一个牧师的非空集合(表决的法定人数集quorum)  
\\(B_{vot}\\) : 一个牧师的集合(所有对法令作出投票(赞成)的牧师).
\\(B_{bal}\\) : 这轮表决的编号  
说一轮表决B是成功的,当且仅当 \\( B_{qrm} \subseteq B_{vot} \\) , 一轮成功的投票是每一个法定人数
集的成员都投了票的(赞成票,不赞成则不投票)

vote v 包含3个组成部分： v_{pst} 投票的牧师 ，v_{bal}本论表决的编号 ，和所表决的法令v_{dec}.
v_{dec}不一定是个有效的表决，是v_{pst}对v_{dec}的提议"点过赞" :-) 。

Votes(B)  MaxVote(b, p, beta)  -> MaxVote(b, Q, beta)


先把定义理解清楚了,不要被先入为主的投票概念混淆了, 这样证明比较简单了.


## 一致性约束
### 文本表述
### 形式化表述

\\(vote \space v \\) was defined to be  \\( \langle  v_{pst}\\) , \\(v_{bal}\\) , \\(v_{dec} \rangle \\)   
\\(v_{pst}\\)  a priest   
\\(v_{bal}\\)   a ballot  
\\(v_{dec}\\) a decree   

\\( null \\) vote: \\( v_{bal} = - \infty , v_{dec} = BLANK,  null_p \\)

\\( Votes(B) =\\\{ v \vert (v_{pst} \in B_{vot})  \land (v_{bal} =B_{bal}) \land (v_{dec} = B_{dec}) \\\}  \\)

\\( MaxVote(b, p, B) = \\)largest  vote in set \\(  \\\{v \in Votes(B) \vert (v_{pst} = p) \land (v_{bal} \lt b)\\\} \cup \\\{ null_p \\\}  \\)
p在编号小于b轮次中, 轮次号最大的投票.

\\( MaxVote(b, B_{qrm}, B) = \\)呢, 意思是: 编号为b的某轮次决议(B), 法定人数集B_{qrm}里, 最近投过票的那个vote.

### 引理及其证明思路
引理其实就是另一个表述, 一旦决议成功确定, 不再改动. 不然的话循环下去投票没法结束。 所谓进展性有问题了。

反证法不难得出结论, 思路是这样的：先假设个决议不同于B的决议的大于B轮最小轮次, 然后推导出决定C轮决议的那个MaxVote的轮次号大于B的轮次号，从而得出与假设矛盾的结论，得证。
(1) 不妨假设C轮表决是满足决议不同于B的大于B轮次的最小轮,  
(2) C轮表决中的法定人数集合必然有人q在B中投过票v, 根据MaxVote定义，MaxVote(Cbal, Cqrm, beta)bal >= v_{bal} = B_{bal}
(3) 而Cdec = MaxVote(Cbal, Cqrm, beta)dec != B_dec, 根据约束1，所以MaxVote(Cbal, Cqrm, beta)bal ,!= v_{bal} = B_{bal}
    去掉等于后，得出：MaxVote(Cbal, Cqrm, beta)bal > B_{bal}
(4) 而由MaxVote定义，Cbal > MaxVote(Cbal, Cqrm, beta)bal
(5) 与Cbal为满足假设的最小轮次矛盾。证毕.
## 从一致性约束条件到初级协议(The Preliminary Protocol)
(B1). 为了使 B1 成立，每一轮表决都必须获得一个唯一的编号
(B2). 为了使 B2 成立， 表决的法定人数集被选定为集合 M，开始 M 只是简单的包含多数牧师就可
以了

(B3). 一个牧师 p 在发起表决前，需要找出MaxVote(b, Q, beta)_{dec}, 必须询问Q集合的每个牧师小于b表决, 于是有下面表决过程之前的准备步骤(两阶段)：

步骤(1)：p询问某些牧师, 我准备发起编号b的表决
p选择一个编号b,给某些牧师(最好>Q集合(quorum)的牧师发一条消息NextBallot(b)
步骤(2): q回答p，告知满足小于b的最大vote.
牧师q响应NextBallot(b), 发送LastVote(b,v)给p, v 是 p 的编号小于 b的表决。q没投过票，这null_{p}
为了满足约束条件(B3), MaxVote(b,q,beta)在q发送LastVote(b,v)不再改变，q必须承诺在编号v_bal,b之间的表决过程中不再投票。(q,比较记录v和b来保证承诺)

步骤(3):
收到过半的q，中编号最大的表决的投票，法令选择满足B3， 然后给Q发送beginballot(b,d), 开始表决过程

步骤4,
在收到 BeginBallot(b,d)消息后， 牧师 q 发现他发送的LastVote编号就是b，那么他回复投上一票。如果q发送的LastVote(b2, v), 编号不同, 为了维持B3约束不改变MaxVote，那么他决定不投赞成票。

 步骤(3)的执行可以认为是将一个表决 B 加入到集合β 中， 其中 =b， =Q， = （还
没有人在这轮表决中投票）， =d。但此时还不会影响MaxVote(b,q,beta), 因为还没有人为bd投票呢。


在步骤(4)中，如果牧师 q 决定在表决中投票，那么执
行这一步骤可以认为是通过将 q加入到 B的投票者集合 中来改变了表决集合β ，这里 B
β

步骤5，如果p收到Q(法定人数)的投票(Voted(b,q)),那么他记下d，发送success(d)给所有人.
步骤6，其他人收到success(d), 记下这个决议d.

## 加强约束的基本协议(The Basic Protocal)
在基本协议中, 为维持MaxVote不变, p发送LastVote(b,v)后，承诺不对(v_bal, b)之间的表决投票. 加强条件，基本协议中不再对小于b的表决投票.

牧师需要记录如下信息，
* lastTried[p], 发起决议的人记住自己上一次曾发起决议的编号, 新发起的决议要更大编号
* nextBal[p], 牧师发出所有LastVote(b,v)中，最大的编号, 用于回答别人发起决议提案第一阶段的编号比较.
* nextBal(b), 曾投过票的决议编号最大值, 用于答复别人的最大编号询问.

步骤和基本协议基本相似, 投票过程更新记录的信息.

## 完整的神会协议(The Complelte Synod Protocal)
基本协议满足B1,B2,B3约束，可以保证一致性. 但progress(能成功表决)不能保证. 大家都不启动一轮表决(ballot), 显然没有progress. 过于频繁的启动一轮表决的话,不断有新的编号来抢占更新. progress也进展缓慢.

，
(1)完整的神会协议增强progress:
(a) 时间窗口, 没有能够BeginBal和secuceed(d),才重新启动一轮表决, 不至于过于频繁抢占.  需重新启动表决的原因有二，一是别人挂掉了或者通信出问题. 二是本牧师提的决议编号过小，跟不上形势发展了. 于是改进：
(b) 第一阶段回复提议时，如果q收到某落后分子p的提议编号小于nextBal[q], 虽然不投赞成票，但不妨告知提议者，让他尽快学习跟上新形势，免得发一些过小编号的噪声提议出来.

(2). 只有一个牧师发起表决，并且多数牧师不会离开议会厅的前提下，可进展性条件将会被满足。
完整协议因此包含一个流程来选择一个唯一的牧师，称作总统，来发起表决

## Multi-Decree Parliament




=================================
为了找出 ， p 需要对 Q 中的每一个 q 找出
### The Basic Protocol
增强的约束为
记录的信息可以优化为
###  The Complete Synod Protocol

## 性能优化，leader
q发送 LastVote(b,v)给 p,v 是 p 的编号小于 b 的表决
中编号最大的表决的投票
A priest q responds to the receipt of a NextBallot (b) message by sending a
LastVote(b, v) message to p, where v is the vote with the largest ballot number
less than b that q has cast, or his null vote null q if q did not vote in any ballot
numbered less than b.

===================================
## the basic protocal

* lastTried[p]:  The number of the last ballot that p tried to initiate
* lastVote[p]:   The vote cast by p in the highest-numbered ballot in which he voted
* nextBal[p]: The largest value of b for which p has sent a LastVote(b, v) message,
is kept on s slip paper

(1). p chooses b (b>lastTried[p]), set lastTried[p] = b, sends NextBallot(b)
(2). q: if b > nextBal[q], nextBal[q] = b, sends LastVote(b, v), where v = prevVote[q].
(3). p: LastVote(b,v) from quorum, b=LastTried[p],  <b, Q, d>  BeginBallot(b, d) ->A
(4). q: if b==nextBal[q], vote, set prevVote[q]=vote <q, b, d>,  sends Voted(b,d) to P
(5). p: if Voted(b,d) from Q, where b=lastTried[p],   Success(d) to allowfullscreen
(6).  enter d in ledger

===========================
\\(\mathbf {B1}(B)\\)  
\\(B_{bal}^{'}  \\)  
\\(
\Psi \in \Rightarrow \neq \forall \phi \infty \wedge \cap \cup \subseteq \geq
\gt \lt
\land \lor \lnot \forall \exists \top \bot \vdash \langle \rangle \vert
 \\)  

 \triangleq  
====================
