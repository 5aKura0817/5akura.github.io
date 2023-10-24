<h1>[1155] 掷骰子等于目标和的方法数</h1>
<ul>
<li>
<p>address</p>
<p><a href="https://leetcode.cn/problems/number-of-dice-rolls-with-target-sum">[1155] 掷骰子等于目标和的方法数</a></p>
</li>
<li>
<p>tags</p>
<p>​#leetcode#​ ​#alg/动态规划#​</p>
</li>
<li>
<p>solution explain</p>
<blockquote>
<p>🥰很经典的动态规划题</p>
<p>首先按题意<code>n</code>​个<code>k</code>​面骰子一共能产生<span class="language-math">k^n</span>种结果，要从这些结果中筛选出结果为<code>target</code>​的结果数量。</p>
<p>​<code>k</code>​可以视为常量，利用动态规划思想拆分问题：</p>
<p>n个骰子能组合出target的结果数量 = <u>n-1个骰子能组合出target - 1的结果数量（假设第n个骰子掷出点数为1）</u> + ... + <u>n-1个骰子能组合出target - k的结果数量（假设第n个骰子掷出点数为k）</u></p>
<p>这样转换公式就显而易见了：<span class="language-math">f(n, target) = 	\sum_{pick=1}^k f(n-1, target-pick)</span>​</p>
<p>因为条件限制了 <code>1 &lt;= n,k &lt;= 30</code>​，则初始状态就是<span class="language-math">f(1, 1..target)</span>，当我们只有一个骰子时，所有<strong>小于等于</strong>​<code>**k**</code>​的target, 其结果都是1，反之则为0</p>
<p><span class="language-math">f(1, target) = \begin{cases}	1 &amp;\text{if } target \leq k \\	0 &amp;\text{if } target \gt k\end{cases}</span></p>
<p>这样从上直下，就可以推算出整个dp数组，dp[n][target] = <span class="language-math">f(n, target)</span> 即为所求</p>
</blockquote>
</li>
<li>
<p>solution code</p>
<ul>
<li>
<p>Python3</p>
<pre><code class="language-python">class Solution:
    def numRollsToTarget(self, n: int, k: int, target: int) -&gt; int:
        dp = [[0] * (target + 1) for _ in range(n + 1)]
        for i in range(1, target + 1):
            if k &gt;= i:
                dp[1][i] = 1

        for cnt in range(2, n + 1):
            for sum in range(1, target + 1):
                for pick in range(1, k + 1):
                    if sum &gt; pick:
                        dp[cnt][sum] += dp[cnt - 1][sum - pick]
                        dp[cnt][sum] %= (10 ** 9 + 7)

        return dp[-1][-1]
</code></pre>
<blockquote>
<p>Tips: 在确定了target和k之后，可以通过大小关系对循环次数进行缩减，甚至可以对dp数组进行压缩，以优化空间占用和运行时间。</p>
</blockquote>
</li>
</ul>
</li>
</ul>
