reference：
[1] https://www.cnblogs.com/lsgxeva/p/12021636.html
[2] zhihu: ChainingBlocks

前置知识：
--------------------------
-Schnorr签名算法
原理：
G：椭圆曲线
m: 待签名的数据，通常是一个32字节的哈希值
x: 私钥：P=xG， P为x对应的公钥
H(): 哈希函数：H（m || R || P）可以理解为将m, R, P三个字段拼接在一起然后做哈希运算

生成签名
签名者已知：G椭圆曲线, H哈希函数，m待签名消息， x私钥， P为x对应的公钥
1. 选择一个随机数k, 令R = kG
2. 令s = k + H(m || R || P)* x
那么，公钥P对消息m的Schnorr签名就是（R，s）

验证签名
验证者已知：G椭圆曲线，H哈希函数，m待签名信息，P公钥, (R,s)-Schnorr签名
若此等式成立，则可证明签名合法
sG = R + H(m || R || P)P
推演步骤如下：（椭圆曲线无法进行除法运算）
从生成签名我们已知：
s = k + H(m || R || P)* x，等式两边都乘以椭圆曲线G
sG = kG + H(m || R || P)*x*G； since R = kG, P = xG
sG = R + H(m || R || P)P; 因为椭圆曲线无法进行除法运算，所以第3步的等式，无法向前反推出第1步，也就不会暴露k值以及x私钥。同时也完成了等式验证
因为接收方接收到（R,s）所以把这两个参数代入上式就可以判断了

组签，Group Signature
一组N个公钥，签名得到N个签名。这个N个签名可以相加，最终得到一个签名。 这个签名的验证通过，则代表N把公钥的签名全部验证通过
已知：
椭圆曲线： G
待签名的数据： m
哈希函数：H()
私钥: x1, x2, 公钥：P1 = x1*G, P2=x2*G
随机数: k1, k2, 并有R1=k1*G，R2=k2*G
组公钥：P=P1+P2

则有：
私钥x1和x2的签名为：(R1,s1),(R2,s2)
两个签名相加得到组签名（R,s）其中：R=R1+R2, s=s1+s2
推演步骤如下：
R=R1+R2, s=s1+s2
s1 = k1 + H(m || R || P)* x1
s2 = k2 + H(m || R || P)* x2

s = s1 +s2
   = k1 + H(m || R || P)*x1
   + k2 + H(m || R || P)*x2
   = (k1 + k2) + H(m || R || P)*(x1 + x2)
两边同时乘以G，则有：
sG = (k1 + k2)G + H(m || R || P)*(x1 + x2)G
     = (k1G + k2G)+ H(m || R || P)*(x1G + x2G)
     = (R1 + R2) + H(m || R || P)(P1 + P2)
     = (R) + H(m || R || P)P
证明完成

--------------------------
-Keeping authorities" honest or bust" with decentralized witness cosigning."2016 IEEE Symposium on Security and Privacy (SP). Ieee, 2016

问题：
我们通常使用Certificate Authority来认证一个公钥是否属于某个组织或者公司，但是黑客可能会把认证中心的密钥偷了，或者存在中间人攻击，使我们拿到看似合理但是确实假的认证结果

解决思路：
认证中心(leader)不再是单独的认证中心，还有许多不同实体组成的见证者(witness), leader在认证和签名每一个消息S之前，都需要witness检查并签名一下，leader收集这些witness的签名，统一发给认证请求者。请求者通过认证各个witness的签名来确认结果的有效性。

步骤：
https://zhuanlan.zhihu.com/p/165991447
Announcement: leader发送一个公告给各个witness，表示将要进行的任务（可选地包含想认证和签名的消息S）
Commitment: 每个witnesses生成~Vj,将其发给父节点，父节点也生成自己的Vi = G^vi,然后将自己的Vi和子节点发来的合并，结果为~Vi
Challenge: leader合并到最后的值为~V0。leader计算c=H(~V0 || S)，将c广播给witnesses。
Response: 
-叶子节点计算： r3 = v3 - x3*c; ~r3 = r3; 之后将~r3向上传递给父节点
-非父节点计算： r1 = v1 - x1*c; ~r1 = r1 + r所有子节点
-根节点计算： r0 = v0 - x0*c; ~r0 = r0 + r所有传递上来的r节点值

最终将(c， ~r0)发给认证的请求者

--------------------------
-Bitcoin-ng: A scalable blockchain protocol
现存问题：
比特币平均10分钟出一个块，且每个块大小限制在1MB，使用隔离见证技术之后可以达到2MB一个块（这个是什么？），每秒的交易速度是7。低吞吐，高延迟，严重限制比特币在企业级的应用。
现在我们将成功解决工作量证明难题的矿工叫做leader，那么一个矿工成功算出一次PoW并成功打包一个区块，我们叫这个过程为一个新leader的选举。注意的是，比特币使用PoW的作用是选出具有打包资格的leader。
总体思路：
因此在这个过程中比特币的leader选举和区块的打包是一起完成的，作者想把这两个过程分开，让两者以不同的速率工作，比如10分钟一次leader选举，但每30秒打包一次，并且在这期间所有的区块都由这个leader打包，这样是不是可以提高系统的吞吐量？
流程：
任何矿工成功解决了PoW难题之后，就是当前时期的leader,成为leader之后，它的公钥就会保存在一个新的key block中。因为micro block不需要挖矿，所以生成它的代价很小也很快。因此恶意的leader可能一次性生成不同的micro block，发给不同的节点，使得整个区块链形成叉链从而达到double spending attacks.
为了解决这个问题，bitcoin-NG引入poison transaction和proof of fraud,如果其他leader发现之前有某一个leader有这样的作恶行为，就生成一个proof of fraud,不仅有奖励，还会让那个作恶的leader的挖矿奖励失效。
分叉：
micro block不需要挖矿，所以leader election似乎是基于算出key block的PoW实现的，这就会出现因网络延迟导致新leader看不到最后一个micro block的情况，解决方法是等待一定的网络延迟时间段后，再将新key block连接到上一个leader生成的micro blocks上。

--------------------------
-Enhancing bitcoin security and performance with strong consistency via collective signing

本文是cosigning，bitcoin-NG和PBFT的内容三个技术的结合
这篇论文的笔记全部写在图片里，看图片....




