---


---

<p>Reading code from google is a invaluable method of learning from how great minds thinks and write code , <strong>LevelDB</strong>  code is a master class, it’s is a fast, lightweight key-value store that is designed for high-performance storage using <strong>log-structured merge-trees (LSM-trees)</strong>,<br>
LSM only appends records and does not update i.e. multiple instances of a key can be present but the most recent update is the current version of the key.<br>
LSM maintains a set of files, where each file has a sorted run of a range of keys.<br>
To find the most recent version of a key,  the database first scans the memory and caches and then the disk files from the newest to the oldest.<br>
The scan  finishes when the first instance of the key is found, this scan can be very expensive if the database is big with many files, moreover, if the key is not present, the scan needs to search all the files.<br>
Optimizations were implemented to reduce the scan to be fast, for example, the range that the file contains is listed in the metadata block, and if the searched key is not within the rage, the file is skipped.<br>
Another efficient and elegant technique is the use of bloom filters,  a probabilistic data structure that can be queried whether a key is present in the file, a false means that the file can be skipped as it guarantees no false negative, while false positives are possible.<br>
The probablity that the bloom filter will response with false positive is dependent on the configred size of the bloom.<br>
we’ll go over level db implementation to and usage to see .</p>
<p>TODO - describe how the filter works e.g.  on each key it lits a bit per hash function and stores it in a position in the filter.</p>
<h2 id="filterpolicy-interface">FilterPolicy interface</h2>
<p>An interface that defines the functionality of a key filter, as with GoLang idioms , google interfaces tend to be minimal, elegant, and well defined.<br>
The minimal design, enables to implement the interface without redundant functionality that complicates the implementation.<br>
The interface contains only three methods</p>
<ul>
<li><code>virtual const char* Name() const = 0</code></li>
<li><code>virtual void CreateFilter(const Slice* keys, int n, std::string* dst) const = 0</code> - Create a filter from the n keys and append it to dst.
<ul>
<li>Note - Slice is similar to std::string_view i.e. it stores a pointer to const char* and a size, it does not own the pointer but give access to view it.</li>
</ul>
</li>
<li><code>virtual bool KeyMayMatch(const Slice&amp; key, const Slice&amp; filter) const = 0</code> - query the filter whether the key is present.</li>
</ul>
<p>The file also exposes a factory method to create a concreate implementation of a FilterPolicy instance which creates a BloomFilterPolicy.</p>
<h2 id="bloomfilterpolicy">BloomFilterPolicy</h2>
<p>Implements a FilterPolicy</p>
<hr>
<p>Constructor</p>
<pre><code>explicit  BloomFilterPolicy(int  bits_per_key) : bits_per_key_(bits_per_key) {
	    // We intentionally round down to reduce probing cost a little bit
	    k_  =  static_cast&lt;size_t&gt;(bits_per_key  *  0.69); // 0.69 =~ ln(2)
	    if (k_  &lt;  1) k_  =  1;
	    if (k_  &gt;  30) k_  =  30; 
    }
</code></pre>
<ul>
<li>The explicit keyword is used to disable implicit conversion, i.e. without the explicit the compiler can implicitly create a BloomFilterPolicy object from an int.</li>
<li>the input parameter defines how many bits are allocated per key when creating the filter, the more bits are assigned the less collisions are possible due to the spread of the resulting hash per key (it will become clear when we’ll analyze the filter creation).</li>
<li>The trade off when choosing a bits_per_key value is between false positive rate and the filter size i.e. memory usage. the authors of leveldb recommends a 10 bits per key which  yields a filter with ~1% false positive.</li>
<li>the constructor initializes the member <code>k_</code> which determines the number of  bits that are being set to represent the existence of the key (this also will become clear when we’ll see the filter creation). the chosen value is a balance between performance and false positive ratio i.e. more probes i.e. larger k , the more time is spent on insertion and lookup but less false positives (explain formula)</li>
</ul>
<hr>
<pre><code>void CreateFilter(const Slice* keys, int n, std::string* dst) const override {
    // Compute bloom filter size (in both bits and bytes)
    size_t bits = n * bits_per_key_;

    // For small n, we can see a very high false positive rate. Fix it
    // by enforcing a minimum bloom filter length.
    if (bits &lt; 64) bits = 64;

    size_t bytes = (bits + 7) / 8;
    bits = bytes * 8;

    const size_t init_size = dst-&gt;size();
    dst-&gt;resize(init_size + bytes, 0);
    dst-&gt;push_back(static_cast&lt;char&gt;(k_));  // Remember # of probes in filter
    char* array = &amp;(*dst)[init_size];

    for (int i = 0; i &lt; n; i++) {
        // Use double-hashing to generate a sequence of hash values.
        // See analysis in [Kirsch, Mitzenmacher 2006].
        uint32_t h = BloomHash(keys[i]);
        const uint32_t delta = (h &gt;&gt; 17) | (h &lt;&lt; 15);  // Rotate right 17 bits
        for (size_t j = 0; j &lt; k_; j++) {
            const uint32_t bitpos = h % bits;
            array[bitpos / 8] |= (1 &lt;&lt; (bitpos % 8));
            h += delta;
        }
    }
}
</code></pre>
<p>Creates a bloom filter for <code>n</code> keys and <strong>appends</strong> to dst.</p>
<ul>
<li>compute how many bits are required for the n keys, by computing how many bytes are required and multuplying by 8.</li>
<li>resize the input filter to match the desired size, initialize appended chars to 0.</li>
<li>insert the number of probes that the filter is configured to use, the k_ member is of type size_t byte casted to char i.e. a byte, the code in the contructor ensures that k_ will be in the range that a char can represent, though any change to it can break it, so it has some danger .</li>
<li>take the address of the filter that corresponds to the current filter.</li>
<li>iterate over each key:</li>
</ul>
<ol>
<li>
<p>Hash the key to a uint32 value (will analyze the hashing function as well).</p>
</li>
<li>
<p><code>const uint32_t delta = (h &gt;&gt; 17) | (h &lt;&lt; 15)</code> this is an interesting line, <code>delta</code> is <strong>a modified version of the hash value <code>h</code></strong>, used to generate multiple hash values <strong>efficiently</strong> via <strong>double hashing</strong>. This approach avoids computing multiple independent hash functions, which would be computationally expensive.<br>
This line <strong>rotates</strong> <code>h</code> by <strong>17 bits to the right</strong> and <strong>15 bits to the left</strong>, then <strong>combines</strong> the results using bitwise OR (<code>|</code>).</p>
<ul>
<li>Since a <code>uint32_t</code> has <strong>32 bits</strong>, a <strong>right shift by 17 bits</strong> moves the <strong>upper 17 bits</strong> to the <strong>lower 17 bits</strong>.</li>
<li>The <strong>left shift by 15 bits</strong> moves the <strong>lower 15 bits</strong> into the <strong>upper 15 bits</strong>.</li>
<li>The bitwise OR <strong>merges the shifted values</strong>, ensuring that <code>delta</code> is a <strong>rearranged version</strong> of <code>h</code> that retains entropy.</li>
</ul>
<h4 id="why-this-specific-shift--17---15"><strong>Why This Specific Shift (<code>&gt;&gt; 17 | &lt;&lt; 15</code>)?</strong></h4>
<ul>
<li>The chosen shifts (<code>17</code> and <code>15</code>) are <strong>coprime with 32</strong>, ensuring that <code>delta</code> <strong>preserves randomness</strong> across different iterations.</li>
<li>This prevents the generated sequence from <strong>cycling too soon</strong>, ensuring that the bits are distributed <strong>evenly</strong> in the Bloom filter.</li>
</ul>
</li>
<li>
<p>the <code>for</code> loop is where we use the probe factor, iterate K times over the hash, for each iteration a bit is set at a specific byte within the filter range, that bit represnts that key for that hash function, resulting in K bits that represnets the key.<br>
the operations done are:</p>
<ul>
<li>calculate the bit location within the byte by performing module on the hash value.</li>
<li>calculate the byte position within the filter of the hash value.</li>
<li>set the corresponding bit at that byte.</li>
<li>apply the delta to h to perform the double hashing some notes about this technique</li>
</ul>
</li>
<li>
<p><strong>Efficient Hash Computation</strong></p>
<ul>
<li>Computing multiple independent hash functions is expensive. <strong>Double hashing</strong> (using <code>h += delta</code>) generates <code>k_</code> hash values in <strong>constant time</strong> instead of <code>O(k_)</code> time.</li>
</ul>
</li>
<li>
<p><strong>Avoids Collisions &amp; Clustering</strong></p>
<ul>
<li>By using a <strong>rotated version of <code>h</code></strong>, <code>delta</code> ensures that the <strong>additional hash values are well-distributed</strong> in the Bloom filter, reducing collisions.</li>
</ul>
</li>
<li>
<p><strong>Ensures Uniform Distribution of Bits</strong></p>
<ul>
<li>The choice of <code>&gt;&gt; 17 | &lt;&lt; 15</code> ensures that bits <strong>spread out evenly</strong> and do not cluster in predictable patterns.</li>
</ul>
</li>
</ol>
<p>The result of the function is a buffer of size X bytes that is propotional to the number of input keys, where each key has K bits that are set to 1 across that buffer.</p>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

