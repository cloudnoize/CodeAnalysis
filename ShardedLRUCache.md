---


---

<h1 id="leveldbs-sharded-lru-cache-a-deep-dive-into-elegant-design">LevelDB’s Sharded LRU Cache: A Deep Dive into Elegant Design</h1>
<p>An LRU (Least Recently Used) cache evicts the least recently accessed item when its capacity is exceeded. It’s widely used in systems requiring fast access to frequently used data with limited memory, such as databases, web browsers, and operating systems. LevelDB implements this using a hash table for quick lookups and a doubly-linked list to track usage order.</p>
<p>LevelDB’s LRU cache is notable for its elegance, high performance, and educational value. It employs a circular doubly-linked list, a custom hash table, fine-grained reference counting, and shards the cache into independent partitions for scalability and concurrency.</p>
<h2 id="the-interface-clean-abstraction">The Interface: Clean Abstraction</h2>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">class</span> <span class="token class-name">LEVELDB_EXPORT</span> Cache<span class="token punctuation">;</span>

LEVELDB_EXPORT Cache<span class="token operator">*</span> <span class="token function">NewLRUCache</span><span class="token punctuation">(</span>size_t capacity<span class="token punctuation">)</span><span class="token punctuation">;</span>

<span class="token keyword">class</span> <span class="token class-name">LEVELDB_EXPORT</span> Cache <span class="token punctuation">{</span>
 <span class="token keyword">public</span><span class="token operator">:</span>
  <span class="token function">Cache</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token keyword">default</span><span class="token punctuation">;</span>
  <span class="token function">Cache</span><span class="token punctuation">(</span><span class="token keyword">const</span> Cache<span class="token operator">&amp;</span><span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token keyword">delete</span><span class="token punctuation">;</span>
  Cache<span class="token operator">&amp;</span> <span class="token keyword">operator</span><span class="token operator">=</span><span class="token punctuation">(</span><span class="token keyword">const</span> Cache<span class="token operator">&amp;</span><span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token keyword">delete</span><span class="token punctuation">;</span>
  <span class="token keyword">virtual</span> <span class="token operator">~</span><span class="token function">Cache</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

  <span class="token keyword">struct</span> Handle <span class="token punctuation">{</span><span class="token punctuation">}</span><span class="token punctuation">;</span>

  <span class="token keyword">virtual</span> Handle<span class="token operator">*</span> <span class="token function">Insert</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> <span class="token keyword">void</span><span class="token operator">*</span> value<span class="token punctuation">,</span> size_t charge<span class="token punctuation">,</span>
                         <span class="token keyword">void</span> <span class="token punctuation">(</span><span class="token operator">*</span>deleter<span class="token punctuation">)</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> <span class="token keyword">void</span><span class="token operator">*</span> value<span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
  <span class="token keyword">virtual</span> Handle<span class="token operator">*</span> <span class="token function">Lookup</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
  <span class="token keyword">virtual</span> <span class="token keyword">void</span> <span class="token function">Release</span><span class="token punctuation">(</span>Handle<span class="token operator">*</span> handle<span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
  <span class="token keyword">virtual</span> <span class="token keyword">void</span><span class="token operator">*</span> <span class="token function">Value</span><span class="token punctuation">(</span>Handle<span class="token operator">*</span> handle<span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
  <span class="token keyword">virtual</span> <span class="token keyword">void</span> <span class="token function">Erase</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
  <span class="token keyword">virtual</span> uint64_t <span class="token function">NewId</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
  <span class="token keyword">virtual</span> <span class="token keyword">void</span> <span class="token function">Prune</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{</span><span class="token punctuation">}</span>
  <span class="token keyword">virtual</span> size_t <span class="token function">TotalCharge</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token keyword">const</span> <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span><span class="token punctuation">;</span>

</code></pre>
<p>The interface design employs a combination of the Factory Method and Pimpl (Opaque Pointer) Idiom. It defines a pure abstract Cache interface, exposing only a factory function (NewLRUCache) to create instances, while encapsulating all implementation details privately within the .cpp file to achieve complete separation between interface and implementation.</p>
<p>To further strengthen decoupling, cached items are represented by an opaque Handle struct, through which all user interactions with the cache are performed. The underlying implementation internally maps the Handle to its concrete representation.</p>
<p>This design results in a highly stable and minimal interface that remains unaffected by changes to the implementation, eliminating the need for user code recompilation and preserving ABI compatibility.</p>
<h2 id="core-data-structures">Core Data Structures</h2>
<h3 id="lruhandle-the-cache-entry">LRUHandle: The Cache Entry</h3>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">struct</span> LRUHandle <span class="token punctuation">{</span>
  <span class="token keyword">void</span><span class="token operator">*</span> value<span class="token punctuation">;</span>
  <span class="token keyword">void</span> <span class="token punctuation">(</span><span class="token operator">*</span>deleter<span class="token punctuation">)</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span><span class="token punctuation">,</span> <span class="token keyword">void</span><span class="token operator">*</span> value<span class="token punctuation">)</span><span class="token punctuation">;</span>
  LRUHandle<span class="token operator">*</span> next_hash<span class="token punctuation">;</span>
  LRUHandle<span class="token operator">*</span> next<span class="token punctuation">;</span>
  LRUHandle<span class="token operator">*</span> prev<span class="token punctuation">;</span>
  size_t charge<span class="token punctuation">;</span>
  size_t key_length<span class="token punctuation">;</span>
  <span class="token keyword">bool</span> in_cache<span class="token punctuation">;</span>
  uint32_t refs<span class="token punctuation">;</span>
  uint32_t hash<span class="token punctuation">;</span>
  <span class="token keyword">char</span> key_data<span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">]</span><span class="token punctuation">;</span>  <span class="token comment">// Beginning of key</span>

  Slice <span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token keyword">const</span> <span class="token punctuation">{</span>
    <span class="token comment">// next is only equal to this if the LRU handle is the list head of an</span>
    <span class="token comment">// empty list. List heads never have meaningful keys.</span>
    <span class="token function">assert</span><span class="token punctuation">(</span>next <span class="token operator">!=</span> <span class="token keyword">this</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token keyword">return</span> <span class="token function">Slice</span><span class="token punctuation">(</span>key_data<span class="token punctuation">,</span> key_length<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span><span class="token punctuation">;</span>

</code></pre>
<p>Let’s list some interesting features, without going into details on each field as we’ll cover them when explaining the operations:</p>
<ul>
<li><code>value</code> is an opaque pointer to the user value</li>
<li><code>next</code> and <code>prev</code> pointers compose the linked lists used for the LRU implementation</li>
<li><code>next_hash</code> supports hash table operations</li>
<li><code>hash</code> determines which shard this entry belongs to</li>
<li><code>key_data[1]</code> uses a clever trick to have a variable length array as the last member of the struct without requiring extra allocation. When allocating a new LRUHandle: <code>reinterpret_cast&lt;LRUHandle*&gt;(malloc(sizeof(LRUHandle) - 1 + key.size()))</code>, the size calculation adds space for the key at the end of the struct, allowing key_data to store the entire key inline</li>
</ul>
<h3 id="handletable-custom-hash-table">HandleTable: Custom Hash Table</h3>
<p>The HandleTable class provides O(1) access to cached entries through a custom hash table implementation:</p>
<pre class=" language-cpp"><code class="prism  language-cpp">LRUHandle<span class="token operator">*</span><span class="token operator">*</span> <span class="token function">FindPointer</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> uint32_t hash<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  LRUHandle<span class="token operator">*</span><span class="token operator">*</span> ptr <span class="token operator">=</span> <span class="token operator">&amp;</span>list_<span class="token punctuation">[</span>hash <span class="token operator">&amp;</span> <span class="token punctuation">(</span>length_ <span class="token operator">-</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">]</span><span class="token punctuation">;</span>
  
  <span class="token keyword">while</span> <span class="token punctuation">(</span><span class="token operator">*</span>ptr <span class="token operator">!=</span> <span class="token keyword">nullptr</span> <span class="token operator">&amp;&amp;</span> <span class="token punctuation">(</span><span class="token punctuation">(</span><span class="token operator">*</span>ptr<span class="token punctuation">)</span><span class="token operator">-</span><span class="token operator">&gt;</span>hash <span class="token operator">!=</span> hash <span class="token operator">||</span> key <span class="token operator">!=</span> <span class="token punctuation">(</span><span class="token operator">*</span>ptr<span class="token punctuation">)</span><span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    ptr <span class="token operator">=</span> <span class="token operator">&amp;</span><span class="token punctuation">(</span><span class="token operator">*</span>ptr<span class="token punctuation">)</span><span class="token operator">-</span><span class="token operator">&gt;</span>next_hash<span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  
  <span class="token keyword">return</span> ptr<span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<h3 id="handletable-custom-hash-table-1">HandleTable: Custom Hash Table</h3>
<p>A proprietary hash table implementation that optimizes access to the LRUHandles stored in the cache, as the cache uses linked lists to store objects (linear time to find an item) in contrast to the O(1) of the hash table.</p>
<p>The LRUHandle is aware of the hash table through its next_hash member which maintains a single linked list for the bucket. The HashTable doesn’t manage the lifetime of its contained LRUHandle objects, so adding and removing doesn’t allocate or deallocate memory.</p>
<h4 id="members">Members</h4>
<ul>
<li><code>uint32_t length_</code> - the number of buckets the hash table has</li>
<li><code>uint32_t elems_</code> - the number of items currently stored in the table</li>
<li><code>LRUHandle** list_</code> - An array where each cell is a bucket containing a linked list of entries that hash into that bucket</li>
</ul>
<h4 id="key-methods">Key Methods</h4>
<h5 id="findpointer">FindPointer</h5>
<pre class=" language-cpp"><code class="prism  language-cpp">LRUHandle<span class="token operator">*</span><span class="token operator">*</span> <span class="token function">FindPointer</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> uint32_t hash<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    LRUHandle<span class="token operator">*</span><span class="token operator">*</span> ptr <span class="token operator">=</span> <span class="token operator">&amp;</span>list_<span class="token punctuation">[</span>hash <span class="token operator">&amp;</span> <span class="token punctuation">(</span>length_ <span class="token operator">-</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">]</span><span class="token punctuation">;</span>
    
    <span class="token keyword">while</span> <span class="token punctuation">(</span><span class="token operator">*</span>ptr <span class="token operator">!=</span> <span class="token keyword">nullptr</span> <span class="token operator">&amp;&amp;</span> <span class="token punctuation">(</span><span class="token punctuation">(</span><span class="token operator">*</span>ptr<span class="token punctuation">)</span><span class="token operator">-</span><span class="token operator">&gt;</span>hash <span class="token operator">!=</span> hash <span class="token operator">||</span> key <span class="token operator">!=</span> <span class="token punctuation">(</span><span class="token operator">*</span>ptr<span class="token punctuation">)</span><span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
        ptr <span class="token operator">=</span> <span class="token operator">&amp;</span><span class="token punctuation">(</span><span class="token operator">*</span>ptr<span class="token punctuation">)</span><span class="token operator">-</span><span class="token operator">&gt;</span>next_hash<span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
    
    <span class="token keyword">return</span> ptr<span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<ul>
<li><code>LRUHandle** ptr = &amp;list_[hash &amp; (length_ - 1)]</code> - Finds the bucket index by performing hash modulo the number of buckets. The modulo is performed using an optimization to avoid an expensive division operation.</li>
<li>The optimization assumes that length_ is a power of two. When length_ is a power of two, subtracting 1 gives a bitmask that extracts only the lower bits of hash. For example:
<ul>
<li>8 = 1000 (binary)</li>
<li>7 = 0111 (binary)</li>
<li>Doing hash &amp; 7 extracts the lowest 3 bits of hash, ensuring results between 0 and 7</li>
</ul>
</li>
<li>The loop terminates when either the entry is found (when both hash and key match) or when reaching the end of the list, returning a pointer to the pointer that would contain the entry.</li>
</ul>
<h5 id="insert">Insert</h5>
<pre class=" language-cpp"><code class="prism  language-cpp">LRUHandle<span class="token operator">*</span> <span class="token function">Insert</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span> h<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    LRUHandle<span class="token operator">*</span><span class="token operator">*</span> ptr <span class="token operator">=</span> <span class="token function">FindPointer</span><span class="token punctuation">(</span>h<span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> h<span class="token operator">-</span><span class="token operator">&gt;</span>hash<span class="token punctuation">)</span><span class="token punctuation">;</span>
    LRUHandle<span class="token operator">*</span> old <span class="token operator">=</span> <span class="token operator">*</span>ptr<span class="token punctuation">;</span>
    h<span class="token operator">-</span><span class="token operator">&gt;</span>next_hash <span class="token operator">=</span> <span class="token punctuation">(</span>old <span class="token operator">==</span> <span class="token keyword">nullptr</span> <span class="token operator">?</span> <span class="token keyword">nullptr</span> <span class="token operator">:</span> old<span class="token operator">-</span><span class="token operator">&gt;</span>next_hash<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token operator">*</span>ptr <span class="token operator">=</span> h<span class="token punctuation">;</span>

    <span class="token keyword">if</span> <span class="token punctuation">(</span>old <span class="token operator">==</span> <span class="token keyword">nullptr</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
        <span class="token operator">++</span>elems_<span class="token punctuation">;</span>
        <span class="token keyword">if</span> <span class="token punctuation">(</span>elems_ <span class="token operator">&gt;</span> length_<span class="token punctuation">)</span> <span class="token punctuation">{</span>
            <span class="token comment">// Since each cache entry is fairly large, we aim for a small</span>
            <span class="token comment">// average linked list length (&lt;= 1).</span>
            <span class="token function">Resize</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
    
    <span class="token keyword">return</span> old<span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<ul>
<li>Find if the entry already exists (a handle with the same hash and key)</li>
<li>If exists, set the next_hash field of the input handle to point to where the old entry points</li>
<li>As ptr is of type pointer-to-pointer, performing *ptr=h replaces the LRUHandle in the list</li>
<li>If it’s a new entry (FindPointer returned nullptr), increment the number of stored objects and resize if it exceeds the length</li>
</ul>
<h5 id="resize">Resize</h5>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span> <span class="token function">Resize</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    uint32_t new_length <span class="token operator">=</span> <span class="token number">4</span><span class="token punctuation">;</span>
    <span class="token keyword">while</span> <span class="token punctuation">(</span>new_length <span class="token operator">&lt;</span> elems_<span class="token punctuation">)</span> <span class="token punctuation">{</span>
      new_length <span class="token operator">*</span><span class="token operator">=</span> <span class="token number">2</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
    LRUHandle<span class="token operator">*</span><span class="token operator">*</span> new_list <span class="token operator">=</span> <span class="token keyword">new</span> LRUHandle<span class="token operator">*</span><span class="token punctuation">[</span>new_length<span class="token punctuation">]</span><span class="token punctuation">;</span>
    <span class="token function">memset</span><span class="token punctuation">(</span>new_list<span class="token punctuation">,</span> <span class="token number">0</span><span class="token punctuation">,</span> <span class="token keyword">sizeof</span><span class="token punctuation">(</span>new_list<span class="token punctuation">[</span><span class="token number">0</span><span class="token punctuation">]</span><span class="token punctuation">)</span> <span class="token operator">*</span> new_length<span class="token punctuation">)</span><span class="token punctuation">;</span>
    uint32_t count <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span>
    <span class="token keyword">for</span> <span class="token punctuation">(</span>uint32_t i <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span> i <span class="token operator">&lt;</span> length_<span class="token punctuation">;</span> i<span class="token operator">++</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
      LRUHandle<span class="token operator">*</span> h <span class="token operator">=</span> list_<span class="token punctuation">[</span>i<span class="token punctuation">]</span><span class="token punctuation">;</span>
      <span class="token keyword">while</span> <span class="token punctuation">(</span>h <span class="token operator">!=</span> <span class="token keyword">nullptr</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
        LRUHandle<span class="token operator">*</span> next <span class="token operator">=</span> h<span class="token operator">-</span><span class="token operator">&gt;</span>next_hash<span class="token punctuation">;</span>
        uint32_t hash <span class="token operator">=</span> h<span class="token operator">-</span><span class="token operator">&gt;</span>hash<span class="token punctuation">;</span>
        LRUHandle<span class="token operator">*</span><span class="token operator">*</span> ptr <span class="token operator">=</span> <span class="token operator">&amp;</span>new_list<span class="token punctuation">[</span>hash <span class="token operator">&amp;</span> <span class="token punctuation">(</span>new_length <span class="token operator">-</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">]</span><span class="token punctuation">;</span>
        h<span class="token operator">-</span><span class="token operator">&gt;</span>next_hash <span class="token operator">=</span> <span class="token operator">*</span>ptr<span class="token punctuation">;</span>
        <span class="token operator">*</span>ptr <span class="token operator">=</span> h<span class="token punctuation">;</span>
        h <span class="token operator">=</span> next<span class="token punctuation">;</span>
        count<span class="token operator">++</span><span class="token punctuation">;</span>
      <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
    <span class="token function">assert</span><span class="token punctuation">(</span>elems_ <span class="token operator">==</span> count<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token keyword">delete</span><span class="token punctuation">[</span><span class="token punctuation">]</span> list_<span class="token punctuation">;</span>
    list_ <span class="token operator">=</span> new_list<span class="token punctuation">;</span>
    length_ <span class="token operator">=</span> new_length<span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<p>The resize operation:</p>
<ol>
<li>Calculates new_length as the first power of two larger than the current number of stored elements</li>
<li>Allocates a new LRUHandle* array of size new_length</li>
<li>Iterates over the current array, and for each bucket iterates over its list</li>
<li>For each LRUHandle* in the list:
<ul>
<li>Saves the next LRUHandle* it points to in the hash list</li>
<li>Calculates the bucket for that hash in the new array</li>
<li>Makes the current entry the head of the list in the new bucket</li>
<li>Moves to the next entry using the previously saved pointer</li>
<li>Increments the element counter</li>
</ul>
</li>
<li>Once all elements are migrated, verifies the count matches the expected elements count</li>
<li>Deletes the old array and updates the list_ member and length_</li>
</ol>
<p>This hash table implementation maintains an average linked list length of less than or equal to 1, optimizing lookups to approach true O(1) time complexity.</p>
<h2 id="the-lrucache-class">The LRUCache Class</h2>
<p>The LRUCache class implements a single cache shard. Let’s examine its key components:</p>
<ul>
<li><code>size_t capacity_</code>, <code>usage_</code> - When usage_ exceeds capacity_, eviction happens</li>
<li><code>mutex_</code> - guard for thread safety</li>
<li><code>LRUHandle lru_</code> - a circular linked list of entries that are in the cache</li>
<li><code>LRUHandle in_use_</code> - a circular linked list of entries currently in use by clients</li>
<li><code>HandleTable table_</code> - a hash table for fast lookup of entries (instead of linear search in the linked list)</li>
</ul>
<h3 id="constructor-and-destructor">Constructor and Destructor</h3>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token comment">// Constructor - initializes the circular linked lists</span>
LRUCache<span class="token operator">::</span><span class="token function">LRUCache</span><span class="token punctuation">(</span><span class="token punctuation">)</span>
    <span class="token operator">:</span> <span class="token function">usage_</span><span class="token punctuation">(</span><span class="token number">0</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token comment">// Make empty circular linked lists by pointing each sentinel node to itself</span>
  <span class="token comment">// This elegant approach eliminates the need for special case handling</span>
  lru_<span class="token punctuation">.</span>next <span class="token operator">=</span> <span class="token operator">&amp;</span>lru_<span class="token punctuation">;</span>
  lru_<span class="token punctuation">.</span>prev <span class="token operator">=</span> <span class="token operator">&amp;</span>lru_<span class="token punctuation">;</span>
  in_use_<span class="token punctuation">.</span>next <span class="token operator">=</span> <span class="token operator">&amp;</span>in_use_<span class="token punctuation">;</span>
  in_use_<span class="token punctuation">.</span>prev <span class="token operator">=</span> <span class="token operator">&amp;</span>in_use_<span class="token punctuation">;</span>
<span class="token punctuation">}</span>

<span class="token comment">// Destructor - cleans up remaining entries</span>
LRUCache<span class="token operator">::</span><span class="token operator">~</span><span class="token function">LRUCache</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token comment">// First clear entries in the in_use_ list</span>
  <span class="token keyword">while</span> <span class="token punctuation">(</span>in_use_<span class="token punctuation">.</span>next <span class="token operator">!=</span> <span class="token operator">&amp;</span>in_use_<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    LRUHandle<span class="token operator">*</span> e <span class="token operator">=</span> in_use_<span class="token punctuation">.</span>next<span class="token punctuation">;</span>
    <span class="token function">LRU_Remove</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache<span class="token punctuation">)</span><span class="token punctuation">;</span>
    e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache <span class="token operator">=</span> <span class="token boolean">false</span><span class="token punctuation">;</span>
    <span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs <span class="token operator">&gt;=</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">;</span>  <span class="token comment">// Should be pinned by client</span>
    <span class="token function">Unref</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  
  <span class="token comment">// Then clear entries in the lru_ list</span>
  <span class="token keyword">while</span> <span class="token punctuation">(</span>lru_<span class="token punctuation">.</span>next <span class="token operator">!=</span> <span class="token operator">&amp;</span>lru_<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    LRUHandle<span class="token operator">*</span> e <span class="token operator">=</span> lru_<span class="token punctuation">.</span>next<span class="token punctuation">;</span>
    <span class="token function">LRU_Remove</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache<span class="token punctuation">)</span><span class="token punctuation">;</span>
    e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache <span class="token operator">=</span> <span class="token boolean">false</span><span class="token punctuation">;</span>
    <span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs <span class="token operator">==</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">;</span>  <span class="token comment">// Only the cache's reference</span>
    <span class="token function">Unref</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span>

</code></pre>
<p>The constructor initializes the circular linked lists by setting the sentinel nodes to point to themselves, creating empty lists. The destructor methodically cleans up by first removing entries from the in_use_ list, then from the lru_ list, ensuring proper cleanup of all allocated resources.</p>
<h3 id="key-internal-methods">Key Internal Methods</h3>
<p>The linked list operations are elegantly simple:</p>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span> LRUCache<span class="token operator">::</span><span class="token function">LRU_Append</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span> list<span class="token punctuation">,</span> LRUHandle<span class="token operator">*</span> e<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token comment">// Make "e" newest entry by inserting just before *list</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>next <span class="token operator">=</span> list<span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>prev <span class="token operator">=</span> list<span class="token operator">-</span><span class="token operator">&gt;</span>prev<span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>prev<span class="token operator">-</span><span class="token operator">&gt;</span>next <span class="token operator">=</span> e<span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>next<span class="token operator">-</span><span class="token operator">&gt;</span>prev <span class="token operator">=</span> e<span class="token punctuation">;</span>
<span class="token punctuation">}</span>

<span class="token keyword">void</span> LRUCache<span class="token operator">::</span><span class="token function">LRU_Remove</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span> e<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>next<span class="token operator">-</span><span class="token operator">&gt;</span>prev <span class="token operator">=</span> e<span class="token operator">-</span><span class="token operator">&gt;</span>prev<span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>prev<span class="token operator">-</span><span class="token operator">&gt;</span>next <span class="token operator">=</span> e<span class="token operator">-</span><span class="token operator">&gt;</span>next<span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<p>The circular linked list is implemented by using a sentinel node that has its next and prev pointers initialized to point to itself. This elegant trick eliminates the need to handle special cases and automatically creates the circular list.</p>
<p>Reference counting governs entry lifetime:</p>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span> LRUCache<span class="token operator">::</span><span class="token function">Ref</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span> e<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token keyword">if</span> <span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs <span class="token operator">==</span> <span class="token number">1</span> <span class="token operator">&amp;&amp;</span> e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache<span class="token punctuation">)</span> <span class="token punctuation">{</span>  <span class="token comment">// If on lru_ list, move to in_use_ list.</span>
    <span class="token function">LRU_Remove</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token function">LRU_Append</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>in_use_<span class="token punctuation">,</span> e<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>refs<span class="token operator">++</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

<span class="token keyword">void</span> LRUCache<span class="token operator">::</span><span class="token function">Unref</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span> e<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs <span class="token operator">&gt;</span> <span class="token number">0</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token comment">// Decrement the reference count</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>refs<span class="token operator">--</span><span class="token punctuation">;</span>
  <span class="token keyword">if</span> <span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs <span class="token operator">==</span> <span class="token number">0</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    <span class="token comment">// The entry is no longer referenced, so deallocate it</span>
    <span class="token function">assert</span><span class="token punctuation">(</span><span class="token operator">!</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token comment">// Call the deleter callback provided by the user</span>
    <span class="token punctuation">(</span><span class="token operator">*</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>deleter<span class="token punctuation">)</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> e<span class="token operator">-</span><span class="token operator">&gt;</span>value<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token comment">// Free the memory used by the entry</span>
    <span class="token function">free</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span> <span class="token keyword">else</span> <span class="token keyword">if</span> <span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache <span class="token operator">&amp;&amp;</span> e<span class="token operator">-</span><span class="token operator">&gt;</span>refs <span class="token operator">==</span> <span class="token number">1</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    <span class="token comment">// Entry is still in cache but no longer referenced by clients</span>
    <span class="token comment">// Move from in_use_ list to lru_ list (ready for eviction if needed)</span>
    <span class="token function">LRU_Remove</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token function">LRU_Append</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>lru_<span class="token punctuation">,</span> e<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span>

</code></pre>
<p>The <code>Unref</code> method is critical to the cache’s operation:</p>
<ul>
<li>It handles decrementing the reference count of the entry</li>
<li>First asserts that the entry ref count is positive as zero ref count entries are deleted</li>
<li>Decrements the ref count</li>
<li>If the ref count is zero, it:
<ul>
<li>Asserts that the entry is not in use by the cache</li>
<li>Invokes the custom deleter callback that the user gave for this entry</li>
<li>Deallocates the memory</li>
</ul>
</li>
<li>If the ref count is one and the entry is in the cache, it:
<ul>
<li>Removes it from the in_use_ list</li>
<li>Moves it to the lru_ list (making it eligible for eviction)</li>
</ul>
</li>
</ul>
<h3 id="key-operations">Key Operations</h3>
<h4 id="release">Release</h4>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token comment">// Public method to release a handle that was previously looked up</span>
<span class="token keyword">void</span> LRUCache<span class="token operator">::</span><span class="token function">Release</span><span class="token punctuation">(</span>Handle<span class="token operator">*</span> handle<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  MutexLock <span class="token function">l</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token function">Unref</span><span class="token punctuation">(</span><span class="token keyword">reinterpret_cast</span><span class="token operator">&lt;</span>LRUHandle<span class="token operator">*</span><span class="token operator">&gt;</span><span class="token punctuation">(</span>handle<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<p>The <code>Release</code> method is the public interface that allows clients to release their reference to a cache entry. It acquires the mutex for thread safety, then calls <code>Unref</code> on the handle. This is how clients indicate they’re done using a particular cache entry.</p>
<h4 id="erase-and-finisherase">Erase and FinishErase</h4>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token comment">// Removes an entry from the cache, if it exists</span>
<span class="token keyword">void</span> LRUCache<span class="token operator">::</span><span class="token function">Erase</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> uint32_t hash<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  MutexLock <span class="token function">l</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token function">FinishErase</span><span class="token punctuation">(</span>table_<span class="token punctuation">.</span><span class="token function">Remove</span><span class="token punctuation">(</span>key<span class="token punctuation">,</span> hash<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

<span class="token comment">// Completes the erasure of an entry removed from the hash table</span>
<span class="token keyword">bool</span> LRUCache<span class="token operator">::</span><span class="token function">FinishErase</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span> e<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token keyword">if</span> <span class="token punctuation">(</span>e <span class="token operator">!=</span> <span class="token keyword">nullptr</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    <span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token comment">// Remove from cache</span>
    <span class="token function">LRU_Remove</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
    e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache <span class="token operator">=</span> <span class="token boolean">false</span><span class="token punctuation">;</span>
    usage_ <span class="token operator">-</span><span class="token operator">=</span> e<span class="token operator">-</span><span class="token operator">&gt;</span>charge<span class="token punctuation">;</span>
    <span class="token function">Unref</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token keyword">return</span> <span class="token boolean">true</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  <span class="token keyword">return</span> <span class="token boolean">false</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<p>The <code>Erase</code> method removes an entry from the cache:</p>
<ul>
<li>It acquires the mutex for thread safety</li>
<li>Removes the entry from the hash table</li>
<li>Calls <code>FinishErase</code> to complete the removal process</li>
</ul>
<p>The <code>FinishErase</code> method:</p>
<ul>
<li>Takes a handle that’s been removed from the hash table</li>
<li>If the handle exists:
<ul>
<li>Removes it from the LRU list</li>
<li>Marks it as no longer in the cache</li>
<li>Reduces the usage counter by the entry’s charge</li>
<li>Decrements its reference count (potentially freeing it)</li>
<li>Returns true to indicate successful erasure</li>
</ul>
</li>
<li>Returns false if the handle was null (entry not found)</li>
</ul>
<h4 id="prune">Prune</h4>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token comment">// Removes all entries from the cache</span>
<span class="token keyword">void</span> LRUCache<span class="token operator">::</span><span class="token function">Prune</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
  MutexLock <span class="token function">l</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>
  
  <span class="token comment">// Remove all entries from both lists</span>
  <span class="token keyword">while</span> <span class="token punctuation">(</span>lru_<span class="token punctuation">.</span>next <span class="token operator">!=</span> <span class="token operator">&amp;</span>lru_<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    LRUHandle<span class="token operator">*</span> e <span class="token operator">=</span> lru_<span class="token punctuation">.</span>next<span class="token punctuation">;</span>
    <span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs <span class="token operator">==</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">;</span>  <span class="token comment">// Only reference should be from cache</span>
    <span class="token keyword">bool</span> erased <span class="token operator">=</span> <span class="token function">FinishErase</span><span class="token punctuation">(</span>table_<span class="token punctuation">.</span><span class="token function">Remove</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> e<span class="token operator">-</span><span class="token operator">&gt;</span>hash<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token keyword">if</span> <span class="token punctuation">(</span><span class="token operator">!</span>erased<span class="token punctuation">)</span> <span class="token punctuation">{</span>  <span class="token comment">// to avoid unused variable when compiled NDEBUG</span>
      <span class="token function">assert</span><span class="token punctuation">(</span>erased<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
  <span class="token punctuation">}</span>
  
  <span class="token keyword">while</span> <span class="token punctuation">(</span>in_use_<span class="token punctuation">.</span>next <span class="token operator">!=</span> <span class="token operator">&amp;</span>in_use_<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    LRUHandle<span class="token operator">*</span> e <span class="token operator">=</span> in_use_<span class="token punctuation">.</span>next<span class="token punctuation">;</span>
    <span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs <span class="token operator">&gt;</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">;</span>  <span class="token comment">// Should have client references</span>
    <span class="token keyword">bool</span> erased <span class="token operator">=</span> <span class="token function">FinishErase</span><span class="token punctuation">(</span>table_<span class="token punctuation">.</span><span class="token function">Remove</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> e<span class="token operator">-</span><span class="token operator">&gt;</span>hash<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token keyword">if</span> <span class="token punctuation">(</span><span class="token operator">!</span>erased<span class="token punctuation">)</span> <span class="token punctuation">{</span>  <span class="token comment">// to avoid unused variable when compiled NDEBUG</span>
      <span class="token function">assert</span><span class="token punctuation">(</span>erased<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span>

</code></pre>
<p>The <code>Prune</code> method completely empties the cache:</p>
<ul>
<li>It acquires the mutex for thread safety</li>
<li>First removes all entries from the lru_ list (entries not currently in use)</li>
<li>Then removes all entries from the in_use_ list (entries being used by clients)</li>
<li>For each entry:
<ul>
<li>Removes it from the hash table</li>
<li>Calls <code>FinishErase</code> to complete the removal</li>
<li>Verifies that the removal succeeded</li>
</ul>
</li>
</ul>
<h4 id="lookup">Lookup</h4>
<pre class=" language-cpp"><code class="prism  language-cpp">Cache<span class="token operator">::</span>Handle<span class="token operator">*</span> LRUCache<span class="token operator">::</span><span class="token function">Lookup</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> uint32_t hash<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  MutexLock <span class="token function">l</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>
  LRUHandle<span class="token operator">*</span> e <span class="token operator">=</span> table_<span class="token punctuation">.</span><span class="token function">Lookup</span><span class="token punctuation">(</span>key<span class="token punctuation">,</span> hash<span class="token punctuation">)</span><span class="token punctuation">;</span>	
  <span class="token keyword">if</span> <span class="token punctuation">(</span>e <span class="token operator">!=</span> <span class="token keyword">nullptr</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    <span class="token function">Ref</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  <span class="token keyword">return</span> <span class="token keyword">reinterpret_cast</span><span class="token operator">&lt;</span>Cache<span class="token operator">::</span>Handle<span class="token operator">*</span><span class="token operator">&gt;</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<p>The public interface to find an item in the cache, using the hash table for fast retrieval. If found, it increments the reference count.</p>
<p>The return type is not the internal handle representation but the public opaque handle which is used by the client when calling the cache APIs on that item. This design decouples the user code from the cache code, enabling changes without breaking the public interface.</p>
<h4 id="insert-1">Insert</h4>
<pre class=" language-cpp"><code class="prism  language-cpp">Cache<span class="token operator">::</span>Handle<span class="token operator">*</span> LRUCache<span class="token operator">::</span><span class="token function">Insert</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> uint32_t hash<span class="token punctuation">,</span> <span class="token keyword">void</span><span class="token operator">*</span> value<span class="token punctuation">,</span>
                              size_t charge<span class="token punctuation">,</span>
                              <span class="token keyword">void</span> <span class="token punctuation">(</span><span class="token operator">*</span>deleter<span class="token punctuation">)</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> <span class="token keyword">void</span><span class="token operator">*</span> value<span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
  MutexLock <span class="token function">l</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>

  LRUHandle<span class="token operator">*</span> e <span class="token operator">=</span> <span class="token keyword">reinterpret_cast</span><span class="token operator">&lt;</span>LRUHandle<span class="token operator">*</span><span class="token operator">&gt;</span><span class="token punctuation">(</span>
      <span class="token function">malloc</span><span class="token punctuation">(</span><span class="token keyword">sizeof</span><span class="token punctuation">(</span>LRUHandle<span class="token punctuation">)</span> <span class="token operator">-</span> <span class="token number">1</span> <span class="token operator">+</span> key<span class="token punctuation">.</span><span class="token function">size</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>value <span class="token operator">=</span> value<span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>deleter <span class="token operator">=</span> deleter<span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>charge <span class="token operator">=</span> charge<span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>key_length <span class="token operator">=</span> key<span class="token punctuation">.</span><span class="token function">size</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>hash <span class="token operator">=</span> hash<span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache <span class="token operator">=</span> <span class="token boolean">false</span><span class="token punctuation">;</span>
  e<span class="token operator">-</span><span class="token operator">&gt;</span>refs <span class="token operator">=</span> <span class="token number">1</span><span class="token punctuation">;</span>  <span class="token comment">// for the returned handle.</span>
  std<span class="token operator">::</span><span class="token function">memcpy</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>key_data<span class="token punctuation">,</span> key<span class="token punctuation">.</span><span class="token function">data</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> key<span class="token punctuation">.</span><span class="token function">size</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

  <span class="token keyword">if</span> <span class="token punctuation">(</span>capacity_ <span class="token operator">&gt;</span> <span class="token number">0</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    e<span class="token operator">-</span><span class="token operator">&gt;</span>refs<span class="token operator">++</span><span class="token punctuation">;</span>  <span class="token comment">// for the cache's reference.</span>
    e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache <span class="token operator">=</span> <span class="token boolean">true</span><span class="token punctuation">;</span>
    <span class="token function">LRU_Append</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>in_use_<span class="token punctuation">,</span> e<span class="token punctuation">)</span><span class="token punctuation">;</span>
    usage_ <span class="token operator">+</span><span class="token operator">=</span> charge<span class="token punctuation">;</span>
    <span class="token function">FinishErase</span><span class="token punctuation">(</span>table_<span class="token punctuation">.</span><span class="token function">Insert</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span> <span class="token keyword">else</span> <span class="token punctuation">{</span>  <span class="token comment">// don't cache. (capacity_==0 is supported and turns off caching.)</span>
    <span class="token comment">// next is read by key() in an assert, so it must be initialized</span>
    e<span class="token operator">-</span><span class="token operator">&gt;</span>next <span class="token operator">=</span> <span class="token keyword">nullptr</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>

  <span class="token keyword">while</span> <span class="token punctuation">(</span>usage_ <span class="token operator">&gt;</span> capacity_ <span class="token operator">&amp;&amp;</span> lru_<span class="token punctuation">.</span>next <span class="token operator">!=</span> <span class="token operator">&amp;</span>lru_<span class="token punctuation">)</span> <span class="token punctuation">{</span>
    LRUHandle<span class="token operator">*</span> old <span class="token operator">=</span> lru_<span class="token punctuation">.</span>next<span class="token punctuation">;</span>
    <span class="token function">assert</span><span class="token punctuation">(</span>old<span class="token operator">-</span><span class="token operator">&gt;</span>refs <span class="token operator">==</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token keyword">bool</span> erased <span class="token operator">=</span> <span class="token function">FinishErase</span><span class="token punctuation">(</span>table_<span class="token punctuation">.</span><span class="token function">Remove</span><span class="token punctuation">(</span>old<span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> old<span class="token operator">-</span><span class="token operator">&gt;</span>hash<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token keyword">if</span> <span class="token punctuation">(</span><span class="token operator">!</span>erased<span class="token punctuation">)</span> <span class="token punctuation">{</span>  <span class="token comment">// to avoid unused variable when compiled NDEBUG</span>
      <span class="token function">assert</span><span class="token punctuation">(</span>erased<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
  <span class="token punctuation">}</span>

  <span class="token keyword">return</span> <span class="token keyword">reinterpret_cast</span><span class="token operator">&lt;</span>Cache<span class="token operator">::</span>Handle<span class="token operator">*</span><span class="token operator">&gt;</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<p>The insertion process:</p>
<ol>
<li>Locks the entire operation for thread safety</li>
<li>Allocates a new LRUHandle, using the trick where the key allocation array is done dynamically at the end of the struct</li>
<li>Initializes fields including the deleter callback and charge (user-determined “cost” of the entry)</li>
<li>Copies the key to the handle while only pointing to the original value (keys are typically much shorter)</li>
<li>If cache capacity is greater than zero:
<ul>
<li>Increments the reference count</li>
<li>Marks the entry as in the cache</li>
<li>Appends to the in_use_ list</li>
<li>Adds the charge to usage tracking</li>
<li>Inserts into the hash table (handling updates to existing keys)</li>
</ul>
</li>
<li>After insertion, checks if usage exceeds capacity and evicts oldest entries from the LRU list if needed</li>
</ol>
<h2 id="sharding-for-scalability">Sharding for Scalability</h2>
<p>The ShardedLRUCache class distributes cache entries across multiple LRU caches:</p>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">static</span> <span class="token keyword">const</span> <span class="token keyword">int</span> kNumShardBits <span class="token operator">=</span> <span class="token number">4</span><span class="token punctuation">;</span>
<span class="token keyword">static</span> <span class="token keyword">const</span> <span class="token keyword">int</span> kNumShards <span class="token operator">=</span> <span class="token number">1</span> <span class="token operator">&lt;&lt;</span> kNumShardBits<span class="token punctuation">;</span>  <span class="token comment">// 16 shards</span>

</code></pre>
<h3 id="shard-selection">Shard Selection</h3>
<p>Keys are deterministically distributed across shards by hashing:</p>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">static</span> <span class="token keyword">inline</span> uint32_t <span class="token function">HashSlice</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> s<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token keyword">return</span> <span class="token function">Hash</span><span class="token punctuation">(</span>s<span class="token punctuation">.</span><span class="token function">data</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> s<span class="token punctuation">.</span><span class="token function">size</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token number">0</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

<span class="token keyword">static</span> uint32_t <span class="token function">Shard</span><span class="token punctuation">(</span>uint32_t hash<span class="token punctuation">)</span> <span class="token punctuation">{</span> 
  <span class="token keyword">return</span> hash <span class="token operator">&gt;&gt;</span> <span class="token punctuation">(</span><span class="token number">32</span> <span class="token operator">-</span> kNumShardBits<span class="token punctuation">)</span><span class="token punctuation">;</span> 
<span class="token punctuation">}</span>

</code></pre>
<p>The high-order bits of the hash determine the shard, ensuring good distribution.</p>
<h3 id="operation-dispatch">Operation Dispatch</h3>
<p>All operations are forwarded to the appropriate shard:</p>
<pre class=" language-cpp"><code class="prism  language-cpp">Handle<span class="token operator">*</span> <span class="token function">Insert</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> <span class="token keyword">void</span><span class="token operator">*</span> value<span class="token punctuation">,</span> size_t charge<span class="token punctuation">,</span>
             <span class="token keyword">void</span> <span class="token punctuation">(</span><span class="token operator">*</span>deleter<span class="token punctuation">)</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> <span class="token keyword">void</span><span class="token operator">*</span> value<span class="token punctuation">)</span><span class="token punctuation">)</span> override <span class="token punctuation">{</span>
  <span class="token keyword">const</span> uint32_t hash <span class="token operator">=</span> <span class="token function">HashSlice</span><span class="token punctuation">(</span>key<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token keyword">return</span> shard_<span class="token punctuation">[</span><span class="token function">Shard</span><span class="token punctuation">(</span>hash<span class="token punctuation">)</span><span class="token punctuation">]</span><span class="token punctuation">.</span><span class="token function">Insert</span><span class="token punctuation">(</span>key<span class="token punctuation">,</span> hash<span class="token punctuation">,</span> value<span class="token punctuation">,</span> charge<span class="token punctuation">,</span> deleter<span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

Handle<span class="token operator">*</span> <span class="token function">Lookup</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">)</span> override <span class="token punctuation">{</span>
  <span class="token keyword">const</span> uint32_t hash <span class="token operator">=</span> <span class="token function">HashSlice</span><span class="token punctuation">(</span>key<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token keyword">return</span> shard_<span class="token punctuation">[</span><span class="token function">Shard</span><span class="token punctuation">(</span>hash<span class="token punctuation">)</span><span class="token punctuation">]</span><span class="token punctuation">.</span><span class="token function">Lookup</span><span class="token punctuation">(</span>key<span class="token punctuation">,</span> hash<span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>

</code></pre>
<h3 id="sharded-cache-implementation">Sharded Cache Implementation</h3>
<p>The following class implements sharding—using multiple instances of the cache to scale performance. It implements the Cache interface as well to abstract the cache and expose it as a monolithic cache to the user.</p>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">static</span> <span class="token keyword">const</span> <span class="token keyword">int</span> kNumShardBits <span class="token operator">=</span> <span class="token number">4</span><span class="token punctuation">;</span>
<span class="token keyword">static</span> <span class="token keyword">const</span> <span class="token keyword">int</span> kNumShards <span class="token operator">=</span> <span class="token number">1</span> <span class="token operator">&lt;&lt;</span> kNumShardBits<span class="token punctuation">;</span>  <span class="token comment">// 16 shards</span>

</code></pre>
<p>Keys are distributed across shards deterministically by first hashing them (which spreads the bits in a random order), then shifting the hash 28 bits to the right, resulting in a number in the range [0-15].</p>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">static</span> <span class="token keyword">inline</span> uint32_t <span class="token function">HashSlice</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> s<span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token keyword">return</span> <span class="token function">Hash</span><span class="token punctuation">(</span>s<span class="token punctuation">.</span><span class="token function">data</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> s<span class="token punctuation">.</span><span class="token function">size</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token number">0</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>
<span class="token keyword">static</span> uint32_t <span class="token function">Shard</span><span class="token punctuation">(</span>uint32_t hash<span class="token punctuation">)</span> <span class="token punctuation">{</span> <span class="token keyword">return</span> hash <span class="token operator">&gt;&gt;</span> <span class="token punctuation">(</span><span class="token number">32</span> <span class="token operator">-</span> kNumShardBits<span class="token punctuation">)</span><span class="token punctuation">;</span> <span class="token punctuation">}</span>

</code></pre>
<p>The sharded cache constructor is called with the total capacity, then creates kNumShards instances of single cache and distributes the capacity per cache:</p>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">explicit</span> <span class="token function">ShardedLRUCache</span><span class="token punctuation">(</span>size_t capacity<span class="token punctuation">)</span> <span class="token operator">:</span> <span class="token function">last_id_</span><span class="token punctuation">(</span><span class="token number">0</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token keyword">const</span> size_t per_shard <span class="token operator">=</span> <span class="token punctuation">(</span>capacity <span class="token operator">+</span> <span class="token punctuation">(</span>kNumShards <span class="token operator">-</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token operator">/</span> kNumShards<span class="token punctuation">;</span>
  <span class="token keyword">for</span> <span class="token punctuation">(</span><span class="token keyword">int</span> s <span class="token operator">=</span> <span class="token number">0</span><span class="token punctuation">;</span> s <span class="token operator">&lt;</span> kNumShards<span class="token punctuation">;</span> s<span class="token operator">++</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
    shard_<span class="token punctuation">[</span>s<span class="token punctuation">]</span><span class="token punctuation">.</span><span class="token function">SetCapacity</span><span class="token punctuation">(</span>per_shard<span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span>

</code></pre>
<p>All methods of the sharded cache simply proxy the call to the relevant shard by calling the Shard method with the hash of the key.</p>
<h2 id="key-design-lessons">Key Design Lessons</h2>
<p>From LevelDB’s cache implementation, we can extract several valuable design principles:</p>
<ul>
<li>Clean interfaces: Separating interface from implementation using Factory Method and Pimpl patterns</li>
<li>Memory efficiency: Clever allocation techniques like inline key storage minimize overhead</li>
<li>Concurrency: Fine-grained locking via sharding improves throughput</li>
<li>Data structure choices: Custom implementations tailored to specific needs (hash table, circular lists)</li>
<li>Reference counting: Precise memory management without garbage collection</li>
</ul>
<p>LevelDB’s cache implementation demonstrates how thoughtful design creates code that’s both efficient and maintainable. Its clean abstractions and performance optimizations make it a model worth studying for any system developer.</p>

