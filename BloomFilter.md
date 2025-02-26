<hr>
<h1 id="learning-from-google’s-code-exploring-bloom-filters-in-leveldb">Learning from Google’s Code: Exploring Bloom Filters in LevelDB</h1>
<p>Reading code from Google is an invaluable way to learn how great minds think and write software. <strong>LevelDB</strong> code is a master class: it’s fast, lightweight, and designed for high-performance storage using <strong>log-structured merge-trees (LSM-trees)</strong>.</p>
<p>LSM-trees only append records and do not update in place. This means multiple instances of a key can exist, but the most recent update represents the current version of that key. LSM maintains a set of files, each file storing a sorted range of keys. To find the most recent version of a key, LevelDB first scans memory and caches, then scans disk files from newest to oldest. The scan ends as soon as the key is found.</p>
<p>However, this can be <strong>very expensive</strong> if the database is large with many files, especially when the key doesn’t exist: every file may need to be checked. Multiple optimizations help reduce this overhead:</p>
<ul>
<li><strong>Metadata Block</strong>: Each file has a metadata block listing its key range, allowing quick skips if the target key is out of range.</li>
<li><strong>Bloom Filters</strong>: A probabilistic data structure that answers “might be present” or “definitely not present.” If the filter says “not present,” the file is skipped immediately (no false negatives), though false positives can occur.</li>
</ul>
<p>The probability of a false positive depends on the <strong>configured size</strong> of the Bloom filter. Below, we’ll explore how Bloom filters are implemented in LevelDB and how they’re used.</p>
<blockquote>
</blockquote>
<hr>
<h2 id="filterpolicy-interface">FilterPolicy Interface</h2>
<p>An interface that defines how a key filter operates. Much like GoLang idioms, Google interfaces are typically <strong>minimal, elegant, and well-defined</strong>, making them easy to implement without extra complexity.</p>
<p>The interface contains exactly three methods:</p>
<ul>
<li><code>virtual const char* Name() const = 0;</code></li>
<li><code>virtual void CreateFilter(const Slice* keys, int n, std::string* dst) const = 0;</code>
<ul>
<li>Creates a filter for <code>n</code> keys and appends it to <code>dst</code>.</li>
<li><strong>Note</strong>: <code>Slice</code> is somewhat like <code>std::string_view</code>, storing a pointer (<code>const char*</code>) and a size, but it does not own the data.</li>
</ul>
</li>
<li><code>virtual bool KeyMayMatch(const Slice&amp; key, const Slice&amp; filter) const = 0;</code>
<ul>
<li>Checks whether the key may be present in the filter.</li>
</ul>
</li>
</ul>
<p>The file also provides a factory method that returns a <strong>concrete</strong> <code>FilterPolicy</code> implementation, which becomes a <code>BloomFilterPolicy</code>.</p>
<hr>
<h2 id="bloomfilterpolicy">BloomFilterPolicy</h2>
<p>Implements the <code>FilterPolicy</code>.</p>
<h3 id="constructor">Constructor</h3>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">explicit</span> <span class="token function">BloomFilterPolicy</span><span class="token punctuation">(</span><span class="token keyword">int</span> bits_per_key<span class="token punctuation">)</span> <span class="token operator">:</span> <span class="token function">bits_per_key_</span><span class="token punctuation">(</span>bits_per_key<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    <span class="token comment">// We intentionally round down to reduce probing cost a little bit</span>
    k_ <span class="token operator">=</span> <span class="token keyword">static_cast</span><span class="token operator">&lt;</span>size_t<span class="token operator">&gt;</span><span class="token punctuation">(</span>bits_per_key <span class="token operator">*</span> <span class="token number">0.69</span><span class="token punctuation">)</span><span class="token punctuation">;</span> <span class="token comment">// 0.69 ≈ ln(2)</span>
    <span class="token keyword">if</span> <span class="token punctuation">(</span>k_ <span class="token operator">&lt;</span> <span class="token number">1</span><span class="token punctuation">)</span> k_ <span class="token operator">=</span> <span class="token number">1</span><span class="token punctuation">;</span>
    <span class="token keyword">if</span> <span class="token punctuation">(</span>k_ <span class="token operator">&gt;</span> <span class="token number">30</span><span class="token punctuation">)</span> k_ <span class="token operator">=</span> <span class="token number">30</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<ul>
<li>The <code>explicit</code> keyword prevents implicit conversions (e.g., from <code>int</code> to <code>BloomFilterPolicy</code>).</li>
<li><code>bits_per_key</code> indicates how many bits are allocated for each key in the filter. The more bits allocated, the lower the collision probability (i.e., fewer false positives).</li>
<li>This choice is a <strong>trade-off</strong>: more bits reduce false positives but increase memory usage. The authors of LevelDB suggest using 10 bits per key, which yields about a 1% false positive rate.</li>
<li><code>k_</code> represents how many <strong>hash probes</strong> each key uses when setting bits in the filter. A higher <code>k_</code> typically reduces false positives but increases the cost of building and querying the filter.</li>
</ul>
<hr>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span> <span class="token function">CreateFilter</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">*</span> keys<span class="token punctuation">,</span> <span class="token keyword">int</span> n<span class="token punctuation">,</span> std<span class="token operator">::</span>string<span class="token operator">*</span> dst<span class="token punctuation">)</span> <span class="token keyword">const</span> override <span class="token punctuation">{</span>
    <span class="token comment">// Compute bloom filter size (in both bits and bytes)</span>
    size_t bits <span class="token operator">=</span> n <span class="token operator">*</span> bits_per_key_<span class="token punctuation">;</span>

    <span class="token comment">// For small n, we can see a very high false positive rate.</span>
    <span class="token comment">// Fix it by enforcing a minimum bloom filter length.</span>
    <span class="token keyword">if</span> <span class="token punctuation">(</span>bits <span class="token operator">&lt;</span> <span class="token number">64</span><span class="token punctuation">)</span> bits <span class="token operator">=</span> <span class="token number">64</span><span class="token punctuation">;</span>

    size_t bytes <span class="token operator">=</span> <span class="token punctuation">(</span>bits <span class="token operator">+</span> <span class="token number">7</span><span class="token punctuation">)</span> <span class="token operator">/</span> <span class="token number">8</span><span class="token punctuation">;</span>
    bits <span class="token operator">=</span> bytes <span class="token operator">*</span> <span class="token number">8</span><span class="token punctuation">;</span>

    <span class="token keyword">const</span> size_t init_size <span class="token operator">=</span> dst<span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">size</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    dst<span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">resize</span><span class="token punctuation">(</span>init_size <span class="token operator">+</span> bytes<span class="token punctuation">,</span> <span class="token number">0</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    dst<span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">push_back</span><span class="token punctuation">(</span><span class="token keyword">static_cast</span><span class="token operator">&lt;</span><span class="token keyword">char</span><span class="token operator">&gt;</span><span class="token punctuation">(</span>k_<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>  <span class="token comment">// Remember # of probes</span>
    <span class="token keyword">char</span><span class="token operator">*</span> array <span class="token operator">=</span> <span class="token operator">&amp;</span><span class="token punctuation">(</span><span class="token operator">*</span>dst<span class="token punctuation">)</span><span class="token punctuation">[</span>init_size<span class="token punctuation">]</span><span class="token punctuation">;</span>

    <span class="token keyword">for</span> <span class="token punctuation">(</span><span class="token keyword">int</span> i <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span> i <span class="token operator">&lt;</span> n<span class="token punctuation">;</span> i<span class="token operator">++</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
        <span class="token comment">// Use double-hashing to generate a sequence of hash values.</span>
        <span class="token comment">// See analysis in [Kirsch, Mitzenmacher 2006].</span>
        uint32_t h <span class="token operator">=</span> <span class="token function">BloomHash</span><span class="token punctuation">(</span>keys<span class="token punctuation">[</span>i<span class="token punctuation">]</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token keyword">const</span> uint32_t delta <span class="token operator">=</span> <span class="token punctuation">(</span>h <span class="token operator">&gt;&gt;</span> <span class="token number">17</span><span class="token punctuation">)</span> <span class="token operator">|</span> <span class="token punctuation">(</span>h <span class="token operator">&lt;&lt;</span> <span class="token number">15</span><span class="token punctuation">)</span><span class="token punctuation">;</span>  <span class="token comment">// Rotate right 17 bits</span>
        <span class="token keyword">for</span> <span class="token punctuation">(</span>size_t j <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span> j <span class="token operator">&lt;</span> k_<span class="token punctuation">;</span> j<span class="token operator">++</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
            <span class="token keyword">const</span> uint32_t bitpos <span class="token operator">=</span> h <span class="token operator">%</span> bits<span class="token punctuation">;</span>
            array<span class="token punctuation">[</span>bitpos <span class="token operator">/</span> <span class="token number">8</span><span class="token punctuation">]</span> <span class="token operator">|</span><span class="token operator">=</span> <span class="token punctuation">(</span><span class="token number">1</span> <span class="token operator">&lt;&lt;</span> <span class="token punctuation">(</span>bitpos <span class="token operator">%</span> <span class="token number">8</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            h <span class="token operator">+</span><span class="token operator">=</span> delta<span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>

</code></pre>
<h4 id="explanation-of-createfilter">Explanation of <code>CreateFilter</code>:</h4>
<ol>
<li><strong>Calculate the number of bits</strong> required (<code>n * bits_per_key_</code>) and ensure a minimum size (64 bits) for small <code>n</code>.</li>
<li>Convert that to bytes, then expand <code>dst</code> by that many bytes, initializing them to zero.</li>
<li>Append <code>k_</code> (casted to <code>char</code>), which indicates how many probes will be used.</li>
<li>For each key:
<ul>
<li>Calculate a hash (<code>BloomHash</code>) to get <code>h</code>.</li>
<li>Derive <code>delta</code> by rotating <code>h</code> (17 bits right, 15 bits left) and combining them with bitwise OR.</li>
<li><strong>Double hashing</strong>: for each of the <code>k_</code> probes, pick a bit position (<code>h % bits</code>) and set it, then update <code>h</code> by adding <code>delta</code>.</li>
</ul>
</li>
</ol>
<h5 id="why-rotate-by-17-and-15-bits"><strong>Why Rotate by 17 and 15 Bits?</strong></h5>
<ul>
<li>Both numbers are <strong>coprime with 32</strong>, preserving randomness and preventing repetitive patterns.</li>
<li>Shifting and OR-ing helps distribute the bits widely, lowering collision.</li>
</ul>
<hr>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">static</span> uint32_t <span class="token function">BloomHash</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token function">Hash</span><span class="token punctuation">(</span>key<span class="token punctuation">.</span><span class="token function">data</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> key<span class="token punctuation">.</span><span class="token function">size</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token number">0xbc9f1d34</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

uint32_t <span class="token function">Hash</span><span class="token punctuation">(</span><span class="token keyword">const</span> <span class="token keyword">char</span><span class="token operator">*</span> data<span class="token punctuation">,</span> size_t n<span class="token punctuation">,</span> uint32_t seed<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    <span class="token comment">// Similar to murmur hash</span>
    <span class="token keyword">const</span> uint32_t m <span class="token operator">=</span> <span class="token number">0xc6a4a793</span><span class="token punctuation">;</span>
    <span class="token keyword">const</span> uint32_t r <span class="token operator">=</span> <span class="token number">24</span><span class="token punctuation">;</span>
    <span class="token keyword">const</span> <span class="token keyword">char</span><span class="token operator">*</span> limit <span class="token operator">=</span> data <span class="token operator">+</span> n<span class="token punctuation">;</span>
    uint32_t h <span class="token operator">=</span> seed <span class="token operator">^</span> <span class="token punctuation">(</span>n <span class="token operator">*</span> m<span class="token punctuation">)</span><span class="token punctuation">;</span>

    <span class="token comment">// Pick up four bytes at a time</span>
    <span class="token keyword">while</span> <span class="token punctuation">(</span>limit <span class="token operator">-</span> data <span class="token operator">&gt;=</span> <span class="token number">4</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
        uint32_t w <span class="token operator">=</span> <span class="token function">DecodeFixed32</span><span class="token punctuation">(</span>data<span class="token punctuation">)</span><span class="token punctuation">;</span>
        data <span class="token operator">+</span><span class="token operator">=</span> <span class="token number">4</span><span class="token punctuation">;</span>
        h <span class="token operator">+</span><span class="token operator">=</span> w<span class="token punctuation">;</span>
        h <span class="token operator">*</span><span class="token operator">=</span> m<span class="token punctuation">;</span>
        h <span class="token operator">^</span><span class="token operator">=</span> <span class="token punctuation">(</span>h <span class="token operator">&gt;&gt;</span> <span class="token number">16</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>

    <span class="token comment">// Pick up remaining bytes</span>
    <span class="token keyword">switch</span> <span class="token punctuation">(</span>limit <span class="token operator">-</span> data<span class="token punctuation">)</span> <span class="token punctuation">{</span>
        <span class="token keyword">case</span> <span class="token number">3</span><span class="token operator">:</span>
            h <span class="token operator">+</span><span class="token operator">=</span> <span class="token keyword">static_cast</span><span class="token operator">&lt;</span>uint8_t<span class="token operator">&gt;</span><span class="token punctuation">(</span>data<span class="token punctuation">[</span><span class="token number">2</span><span class="token punctuation">]</span><span class="token punctuation">)</span> <span class="token operator">&lt;&lt;</span> <span class="token number">16</span><span class="token punctuation">;</span>
            <span class="token punctuation">[</span><span class="token punctuation">[</span>fallthrough<span class="token punctuation">]</span><span class="token punctuation">]</span><span class="token punctuation">;</span>
        <span class="token keyword">case</span> <span class="token number">2</span><span class="token operator">:</span>
            h <span class="token operator">+</span><span class="token operator">=</span> <span class="token keyword">static_cast</span><span class="token operator">&lt;</span>uint8_t<span class="token operator">&gt;</span><span class="token punctuation">(</span>data<span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">]</span><span class="token punctuation">)</span> <span class="token operator">&lt;&lt;</span> <span class="token number">8</span><span class="token punctuation">;</span>
            <span class="token punctuation">[</span><span class="token punctuation">[</span>fallthrough<span class="token punctuation">]</span><span class="token punctuation">]</span><span class="token punctuation">;</span>
        <span class="token keyword">case</span> <span class="token number">1</span><span class="token operator">:</span>
            h <span class="token operator">+</span><span class="token operator">=</span> <span class="token keyword">static_cast</span><span class="token operator">&lt;</span>uint8_t<span class="token operator">&gt;</span><span class="token punctuation">(</span>data<span class="token punctuation">[</span><span class="token number">0</span><span class="token punctuation">]</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            h <span class="token operator">*</span><span class="token operator">=</span> m<span class="token punctuation">;</span>
            h <span class="token operator">^</span><span class="token operator">=</span> <span class="token punctuation">(</span>h <span class="token operator">&gt;&gt;</span> r<span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">break</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
    <span class="token keyword">return</span> h<span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<h4 id="how-the-hash-works">How the Hash Works:</h4>
<p>This function generates the initial hash for each key. It’s a variant of <a href="https://en.wikipedia.org/wiki/MurmurHash">MurmurHash</a>, which is <strong>fast</strong>, <strong>non-cryptographic</strong>, and has a low collision rate—perfect for Bloom filters.</p>
<ul>
<li><code>m = 0xc6a4a793</code>: A <strong>multiplicative mixing constant</strong> chosen for strong dispersion (good “avalanche” behavior).</li>
<li><code>r = 24</code>: Determines how many bits to shift for handling leftover bytes.</li>
<li><code>h = seed ^ (n * m)</code>: The initial hash value, where <code>seed</code> is <code>0xbc9f1d34</code> in <code>BloomHash</code>.</li>
<li><strong>Processing</strong>:
<ol>
<li>Read four bytes at a time, decode them as <code>uint32_t</code>, add to <code>h</code>, multiply by <code>m</code>, then XOR <code>h</code> with <code>h &gt;&gt; 16</code>.</li>
<li>If the input is not a multiple of 4 bytes, process the remaining 1–3 bytes by shifting and XORing with the constant <code>r</code>.</li>
</ol>
</li>
</ul>
<hr>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">bool</span> <span class="token function">KeyMayMatch</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> <span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> bloom_filter<span class="token punctuation">)</span> <span class="token keyword">const</span> override <span class="token punctuation">{</span>
    <span class="token keyword">const</span> size_t len <span class="token operator">=</span> bloom_filter<span class="token punctuation">.</span><span class="token function">size</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token keyword">if</span> <span class="token punctuation">(</span>len <span class="token operator">&lt;</span> <span class="token number">2</span><span class="token punctuation">)</span> <span class="token keyword">return</span> <span class="token boolean">false</span><span class="token punctuation">;</span>
    <span class="token keyword">const</span> <span class="token keyword">char</span><span class="token operator">*</span> array <span class="token operator">=</span> bloom_filter<span class="token punctuation">.</span><span class="token function">data</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token keyword">const</span> size_t bits <span class="token operator">=</span> <span class="token punctuation">(</span>len <span class="token operator">-</span> <span class="token number">1</span><span class="token punctuation">)</span> <span class="token operator">*</span> <span class="token number">8</span><span class="token punctuation">;</span>

    <span class="token comment">// Use the encoded k so that we can read filters generated by</span>
    <span class="token comment">// bloom filters created using different parameters.</span>
    <span class="token keyword">const</span> size_t k <span class="token operator">=</span> array<span class="token punctuation">[</span>len <span class="token operator">-</span> <span class="token number">1</span><span class="token punctuation">]</span><span class="token punctuation">;</span>

    uint32_t h <span class="token operator">=</span> <span class="token function">BloomHash</span><span class="token punctuation">(</span>key<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token keyword">const</span> uint32_t delta <span class="token operator">=</span> <span class="token punctuation">(</span>h <span class="token operator">&gt;&gt;</span> <span class="token number">17</span><span class="token punctuation">)</span> <span class="token operator">|</span> <span class="token punctuation">(</span>h <span class="token operator">&lt;&lt;</span> <span class="token number">15</span><span class="token punctuation">)</span><span class="token punctuation">;</span>  <span class="token comment">// Rotate right 17 bits</span>
    <span class="token keyword">for</span> <span class="token punctuation">(</span>size_t j <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span> j <span class="token operator">&lt;</span> k<span class="token punctuation">;</span> j<span class="token operator">++</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
        <span class="token keyword">const</span> uint32_t bitpos <span class="token operator">=</span> h <span class="token operator">%</span> bits<span class="token punctuation">;</span>
        <span class="token keyword">if</span> <span class="token punctuation">(</span><span class="token punctuation">(</span>array<span class="token punctuation">[</span>bitpos <span class="token operator">/</span> <span class="token number">8</span><span class="token punctuation">]</span> <span class="token operator">&amp;</span> <span class="token punctuation">(</span><span class="token number">1</span> <span class="token operator">&lt;&lt;</span> <span class="token punctuation">(</span>bitpos <span class="token operator">%</span> <span class="token number">8</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token operator">==</span> <span class="token number">0</span><span class="token punctuation">)</span> <span class="token keyword">return</span> <span class="token boolean">false</span><span class="token punctuation">;</span>
        h <span class="token operator">+</span><span class="token operator">=</span> delta<span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
    <span class="token keyword">return</span> <span class="token boolean">true</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<h3 id="how-keymaymatch-works">How <code>KeyMayMatch</code> Works:</h3>
<ul>
<li>This is the symmetric operation to <code>CreateFilter</code>.</li>
<li>It reads the last byte of <code>bloom_filter</code> to obtain <code>k</code>.</li>
<li>The total bits are <code>(len - 1) * 8</code>.</li>
<li>The same <strong>double hashing</strong> method is used to determine which bits should be set. If all the required bits are set, the key <strong>may</strong> be present; if any bit is missing, the key is definitely <strong>not</strong> present.</li>
</ul>
<hr>
<h2 id="usage-in-leveldb">Usage in LevelDB</h2>
<h3 id="creation">Creation</h3>
<p>Each <strong>SST file</strong> holds a sorted run of key-value pairs, split into blocks. LevelDB generates a *Bloom filter for all the keys in that SST file, storing it in a <strong>filter block</strong> at the file’s end.</p>
<h3 id="lookup">Lookup</h3>
<ol>
<li><strong>Check the memtable</strong>: If the key is found here, return it immediately.</li>
<li><strong>Search SST files from newest to oldest</strong>:
<ul>
<li>LevelDB <strong>caches</strong> metadata about each SST file, including its Bloom filter block.</li>
<li>Before scanning the file’s blocks, LevelDB first checks the Bloom filter.</li>
<li>If the filter says the key is <strong>not present</strong>, the file is skipped.</li>
<li>If it says the key <strong>may</strong> be present, LevelDB proceeds to look inside the file.</li>
</ul>
</li>
</ol>
<p>By using Bloom filters, LevelDB avoids reading from many files unnecessarily, resulting in better performance.</p>

