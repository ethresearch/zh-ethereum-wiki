<!-- TITLE: RLP -->

<p><strong>Contents</strong></p>
<ul>
<li><a href="#%E5%AE%9A%E4%B9%89">定义</a></li>
<li><a href="#%E4%BE%8B%E5%AD%90">例子</a></li>
</ul>

<p>RLP (递归长度前缀)提供了一种适用于任意二进制数据数组的编码，RLP已经成为以太坊中对对象进行序列化的主要编码方式。 RLP的唯一目标就是解决结构体的编码问题；对原子数据类型（比如，字符串，整数型，浮点型）的编码则交给更高层的协议；以太坊中要求数字必须是一个大端字节序的、没有零占位的存储的格式（也就是说，一个整数0和一个空数组是等同的）。</p>
<p>对于在 RLP 格式中对一个字典数据的编码问题，有两种建议的方式，一种是通过二维数组表达键值对，比如<code>[[k1,v1],[k2,v2]...]</code>，并且对键进行字典序排序；另一种方式是通过以太坊文档中提到的高级的<a href="https://github.com/ethereum/wiki/wiki/Patricia-Tree">基数树</a> 编码来实现。</p>
<h3>
<a id="user-content-定义" class="anchor" href="#%E5%AE%9A%E4%B9%89" aria-hidden="true"><svg class="octicon octicon-link" viewbox="0 0 16 16" version="1.1" width="16" height="16" aria-hidden="true"><path fill-rule="evenodd" d="M4 9h1v1H4c-1.5 0-3-1.69-3-3.5S2.55 3 4 3h4c1.45 0 3 1.69 3 3.5 0 1.41-.91 2.72-2 3.25V8.59c.58-.45 1-1.27 1-2.09C10 5.22 8.98 4 8 4H4c-.98 0-2 1.22-2 2.5S3 9 4 9zm9-3h-1v1h1c1 0 2 1.22 2 2.5S13.98 12 13 12H9c-.98 0-2-1.22-2-2.5 0-.83.42-1.64 1-2.09V6.25c-1.09.53-2 1.84-2 3.25C6 11.31 7.55 13 9 13h4c1.45 0 3-1.69 3-3.5S14.5 6 13 6z"></path></svg></a>定义</h3>
<p>RLP 编码函数接受一个item。定义如下：</p>
<ul>
<li>将一个字符串作为一个item（比如，一个 byte 数组）</li>
<li>一组item列表（list）作为一个item</li>
</ul>
<p>例如，一个空字符串可以是一个item，一个字符串"cat"也可以是一个item，一个含有多个字符串的列表也行，复杂的数据结构也行，比如这样的<code>["cat",["puppy","cow"],"horse",[[]],"pig",[""],"sheep"]</code>。注意在本文后续内容中，说"字符串"的意思其实就相当于"一个确定长度的二进制字节信息数据"；而不要假设或考虑关于字符的编码问题。</p>
<p>RLP 编码定义如下：</p>
<ul>
<li>对于 <code>[0x00, 0x7f]</code> 范围内的单个字节, RLP 编码内容就是字节内容本身。</li>
<li>否则，如果是一个 0-55 字节长的字符串，则RLP编码有一个特别的数值 <strong>0x80</strong> 加上字符串长度，再加上字符串二进制内容。这样，第一个字节的表达范围为 <code>[0x80, 0xb7]</code>.</li>
<li>如果字符串长度超过 55 个字节，RLP 编码由定值 <strong>0xb7</strong> 加上字符串长度所占用的字节数，加上字符串长度的编码，加上字符串二进制内容组成。比如，一个长度为 1024 的字符串，将被编码为<code>\xb9\x04\x00</code> 后面再加上字符串内容。第一字节的表达范围是<code>[0xb8, 0xbf]</code>。</li>
<li>如果列表的内容（它的所有项的组合长度）是0-55个字节长，它的RLP编码由0xC0加上所有的项的RLP编码串联起来的长度得到的单个字节，后跟所有的项的RLP编码的串联组成。 第一字节的范围因此是[0xc0, 0xf7]</li>
<li>如果列表的内容超过55字节，它的RLP编码由0xC0加上所有的项的RLP编码串联起来的长度的长度得到的单个字节，后跟所有的项的RLP编码串联起来的长度的长度，再后跟所有的项的RLP编码的串联组成。 第一字节的范围因此是[0xf8, 0xff] 。</li>
</ul>
<p>用Python代码表达以上逻辑:</p>
<div class="highlight highlight-source-python"><pre><span class="pl-k">def</span> <span class="pl-en">rlp_encode</span>(<span class="pl-smi">input</span>):
    <span class="pl-k">if</span> <span class="pl-c1">isinstance</span>(<span class="pl-c1">input</span>,<span class="pl-c1">str</span>):
        <span class="pl-k">if</span> <span class="pl-c1">len</span>(<span class="pl-c1">input</span>) <span class="pl-k">==</span> <span class="pl-c1">1</span> <span class="pl-k">and</span> <span class="pl-c1">chr</span>(<span class="pl-c1">input</span>) <span class="pl-k">&lt;</span> <span class="pl-c1">128</span>: <span class="pl-k">return</span> <span class="pl-c1">input</span>
        <span class="pl-k">else</span>: <span class="pl-k">return</span> encode_length(<span class="pl-c1">len</span>(<span class="pl-c1">input</span>),<span class="pl-c1">128</span>) <span class="pl-k">+</span> <span class="pl-c1">input</span>
    <span class="pl-k">elif</span> <span class="pl-c1">isinstance</span>(<span class="pl-c1">input</span>,<span class="pl-c1">list</span>):
        output <span class="pl-k">=</span> <span class="pl-s"><span class="pl-pds">'</span><span class="pl-pds">'</span></span>
        <span class="pl-k">for</span> item <span class="pl-k">in</span> <span class="pl-c1">input</span>: output <span class="pl-k">+=</span> rlp_encode(item)
        <span class="pl-k">return</span> encode_length(<span class="pl-c1">len</span>(output),<span class="pl-c1">192</span>) <span class="pl-k">+</span> output

<span class="pl-k">def</span> <span class="pl-en">encode_length</span>(<span class="pl-smi">L</span>,<span class="pl-smi">offset</span>):
    <span class="pl-k">if</span> L <span class="pl-k">&lt;</span> <span class="pl-c1">56</span>:
         <span class="pl-k">return</span> <span class="pl-c1">chr</span>(L <span class="pl-k">+</span> offset)
    <span class="pl-k">elif</span> L <span class="pl-k">&lt;</span> <span class="pl-c1">256</span><span class="pl-k">**</span><span class="pl-c1">8</span>:
         <span class="pl-c1">BL</span> <span class="pl-k">=</span> to_binary(L)
         <span class="pl-k">return</span> <span class="pl-c1">chr</span>(<span class="pl-c1">len</span>(<span class="pl-c1">BL</span>) <span class="pl-k">+</span> offset <span class="pl-k">+</span> <span class="pl-c1">55</span>) <span class="pl-k">+</span> <span class="pl-c1">BL</span>
    <span class="pl-k">else</span>:
         <span class="pl-k">raise</span> <span class="pl-c1">Exception</span>(<span class="pl-s"><span class="pl-pds">"</span>input too long<span class="pl-pds">"</span></span>)

<span class="pl-k">def</span> <span class="pl-en">to_binary</span>(<span class="pl-smi">x</span>):
    <span class="pl-k">return</span> <span class="pl-s"><span class="pl-pds">'</span><span class="pl-pds">'</span></span> <span class="pl-k">if</span> x <span class="pl-k">==</span> <span class="pl-c1">0</span> <span class="pl-k">else</span> to_binary(<span class="pl-c1">int</span>(x <span class="pl-k">/</span> <span class="pl-c1">256</span>)) <span class="pl-k">+</span> <span class="pl-c1">chr</span>(x <span class="pl-k">%</span> <span class="pl-c1">256</span>)</pre></div>
<h3>
<a id="user-content-例子" class="anchor" href="#%E4%BE%8B%E5%AD%90" aria-hidden="true"><svg class="octicon octicon-link" viewbox="0 0 16 16" version="1.1" width="16" height="16" aria-hidden="true"><path fill-rule="evenodd" d="M4 9h1v1H4c-1.5 0-3-1.69-3-3.5S2.55 3 4 3h4c1.45 0 3 1.69 3 3.5 0 1.41-.91 2.72-2 3.25V8.59c.58-.45 1-1.27 1-2.09C10 5.22 8.98 4 8 4H4c-.98 0-2 1.22-2 2.5S3 9 4 9zm9-3h-1v1h1c1 0 2 1.22 2 2.5S13.98 12 13 12H9c-.98 0-2-1.22-2-2.5 0-.83.42-1.64 1-2.09V6.25c-1.09.53-2 1.84-2 3.25C6 11.31 7.55 13 9 13h4c1.45 0 3-1.69 3-3.5S14.5 6 13 6z"></path></svg></a>例子</h3>
<p>字符串 "dog" = [ 0x83, 'd', 'o', 'g' ]</p>
<p>列表 [ "cat", "dog" ] = <code>[ 0xc8, 0x83, 'c', 'a', 't', 0x83, 'd', 'o', 'g' ]</code></p>
<p>空字符串 ('null') = <code>[ 0x80 ]</code></p>
<p>空列表 = <code>[ 0xc0 ]</code></p>
<p>数字15 ('\x0f') = <code>[ 0x0f ]</code></p>
<p>数字 1024 ('\x04\x00') = <code>[ 0x82, 0x04, 0x00 ]</code></p>
<p>The <a href="http://en.wikipedia.org/wiki/Set-theoretic_definition_of_natural_numbers" rel="nofollow">set theoretical representation</a> of two, <code>[ [], [[]], [ [], [[]] ] ] = [ 0xc7, 0xc0, 0xc1, 0xc0, 0xc3, 0xc0, 0xc1, 0xc0 ]</code></p>
<p>字符串 "Lorem ipsum dolor sit amet, consectetur adipisicing elit" = <code>[ 0xb8, 0x38, 'L', 'o', 'r', 'e', 'm', ' ', ... , 'e', 'l', 'i', 't' ]</code></p>