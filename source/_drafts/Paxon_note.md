<script type="text/javascript" src="https://cdn.bootcss.com/mathjax/2.7.1/latest.js?config=default">
</script>

The Part-Time Parliament Note
Ballot:
1. a document listing the alternatives that is used in voting // 投票用的选票
2. a choice that is made by voting //一轮表决

## dingyi

\\(B\\) : 一轮表决(a choice that is made by voting)
\\(B_{dec}\\) : 一条法令(decree)  
\\(B_{qrm}\\) : 一个牧师的非空集合(表决的法定人数集quorum)  
\\(B_{vot}\\) : 一个牧师的集合(所有对法令作出投票(赞成)的牧师), 
\\(B_{bal}\\) : 这轮表决的编号  
说一轮表决B是成功的,当且仅当 \\( B_{qrm} \subseteq B_{vot} \\) , 一轮成功的投票是每一个法定人数
集的成员都投了票的(赞成票,不赞成则不投票)

先把定义理解清楚了,不要很先入为主的投票概念混淆了, 这样证明比较简单了.


\\(vote \space v \\) was defined to be  \\( \langle  v_{pst}\\) , \\(v_{bal}\\) , \\(v_{dec} \rangle \\)   
\\(v_{pst}\\)  a priest   
\\(v_{bal}\\)   a ballot  
\\(v_{dec}\\) a decree   

\\( null \\) vote: \\( v_{bal} = - \infty , v_{dec} = BLANK,  null_p \\)

\\( Votes(B) =\\\{ v \vert (v_{pst} \in B_{vot})  \land (v_{bal} =B_{bal}) \land (v_{dec} = B_{dec}) \\\}  \\)

\\( MaxVote(b, p, B) = \\)largest  vote in set \\(  \\\{v \in Votes(B) \vert (v_{pst} = p) \land (v_{bal} \lt b)\\\} \cup \\\{ null_p \\\}  \\)
p在过去的(编号小于b轮次中), 过去轮次号最大的投票. 拗口, 简单一句话: p的上一次(最近)的投票.

\\( MaxVote(b, B_{qrm}, B) = \\)呢, 意思是: 编号为b的某轮次决议(B), 法定人数集B_{qrm}里, 最近投过票的那个vote.


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
