---


---

<h1 id="leveldb-shardedlrucache">LevelDB ShardedLRUCache</h1>
<p>Before diving to the cache, we’ll review some data structures that support the implementation.</p>
<h3 id="lruhandle">LRUHandle</h3>
<p>Represents an entry in the cache</p>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">struct</span>  LRUHandle <span class="token punctuation">{</span>
	<span class="token keyword">void</span><span class="token operator">*</span>  value<span class="token punctuation">;</span>
	<span class="token keyword">void</span> <span class="token punctuation">(</span><span class="token operator">*</span>deleter<span class="token punctuation">)</span><span class="token punctuation">(</span><span class="token keyword">const</span>  Slice<span class="token operator">&amp;</span><span class="token punctuation">,</span> <span class="token keyword">void</span><span class="token operator">*</span>  value<span class="token punctuation">)</span><span class="token punctuation">;</span>
	LRUHandle<span class="token operator">*</span>  next_hash<span class="token punctuation">;</span>
	LRUHandle<span class="token operator">*</span>  next<span class="token punctuation">;</span>
	LRUHandle<span class="token operator">*</span>  prev<span class="token punctuation">;</span>
	size_t  charge<span class="token punctuation">;</span> <span class="token comment">// TODO(opt): Only allow uint32_t?</span>
	size_t  key_length<span class="token punctuation">;</span>
	<span class="token keyword">bool</span>  in_cache<span class="token punctuation">;</span> <span class="token comment">// Whether entry is in the cache.</span>
	uint32_t  refs<span class="token punctuation">;</span> <span class="token comment">// References, including cache reference, if present.</span>
	uint32_t  hash<span class="token punctuation">;</span> <span class="token comment">// Hash of key(); used for fast sharding and comparisons</span>
	<span class="token keyword">char</span>  key_data<span class="token punctuation">[</span><span class="token number">1</span><span class="token punctuation">]</span><span class="token punctuation">;</span> <span class="token comment">// Beginning of key</span>

	Slice  <span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token keyword">const</span> <span class="token punctuation">{</span>
	<span class="token comment">// next is only equal to this if the LRU handle is the list head of an</span>
	<span class="token comment">// empty list. List heads never have meaningful keys.</span>
	<span class="token function">assert</span><span class="token punctuation">(</span>next  <span class="token operator">!=</span>  <span class="token keyword">this</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token keyword">return</span>  <span class="token function">Slice</span><span class="token punctuation">(</span>key_data<span class="token punctuation">,</span> key_length<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token punctuation">}</span>
<span class="token punctuation">}</span><span class="token punctuation">;</span>
</code></pre>
<p>Let’s list some interesting features, without going to details on each field as we’re going to cover it when dealing the operations.</p>
<ul>
<li>value is an opaque pointer to the user value.</li>
<li>next and prev are used to compose the linked lists that are used for the LRU implementation.</li>
<li>hash defines to which shard this entry belongs to.</li>
<li>key_data[1]- is a trick to have a variable length array as a member of the struct without a need for extra allocation and management, it’s the last member of the struct which gets its allocation when allocating a new LRUHandle as follows <code>reinterpret_cast&lt;LRUHandle*&gt;(malloc(sizeof(LRUHandle) - 1 + key.size()));</code> i.e. in addition to the size of the LRUHandle struct the size of the key is added, resulting in space at the end of the struct that the key_data can store the whole key.</li>
</ul>
<h3 id="class-handletable">Class HandleTable</h3>
<p>A proprietary hash table implementation that optimizes the access to the LRUHandles that are stored in the cache,  as the cache uses linked lists to store the objects  i.e. linear time to find an item in contrast to the O (1) of the hash table.<br>
The LRUHandle is aware to the hash table by having the <code>next_hash</code> member which is used to maintain a single linked list for the bucket.<br>
The HashTable does not manage the lifetime of the its contained LRUHandle’s therefore adding and removing does not allocate or deallocate memory.</p>
<h4 id="members">Members</h4>
<ul>
<li>uint32_t length_ - the number of buckets the hash table has.</li>
<li>uint32_t elems_ - the number of items that are currently stored in the table.</li>
<li>LRUHandle&amp;&amp; list_ - An array where each cell is a bucket that contains a linked list of entries that hash into that bucket.</li>
</ul>
<h4 id="methods">Methods</h4>
<h5 id="findpointer">FindPointer</h5>
<p>Finds the entry that matches the hash and key</p>
<pre class=" language-cpp"><code class="prism  language-cpp">	LRUHandle<span class="token operator">*</span><span class="token operator">*</span> <span class="token function">FindPointer</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> uint32_t hash<span class="token punctuation">)</span> <span class="token punctuation">{</span>
	    LRUHandle<span class="token operator">*</span><span class="token operator">*</span> ptr <span class="token operator">=</span> <span class="token operator">&amp;</span>list_<span class="token punctuation">[</span>hash <span class="token operator">&amp;</span> <span class="token punctuation">(</span>length_ <span class="token operator">-</span> <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">]</span><span class="token punctuation">;</span>
	    
	    <span class="token keyword">while</span> <span class="token punctuation">(</span><span class="token operator">*</span>ptr <span class="token operator">!=</span> <span class="token keyword">nullptr</span> <span class="token operator">&amp;&amp;</span> <span class="token punctuation">(</span><span class="token punctuation">(</span><span class="token operator">*</span>ptr<span class="token punctuation">)</span><span class="token operator">-</span><span class="token operator">&gt;</span>hash <span class="token operator">!=</span> hash <span class="token operator">||</span> key <span class="token operator">!=</span> <span class="token punctuation">(</span><span class="token operator">*</span>ptr<span class="token punctuation">)</span><span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
	        ptr <span class="token operator">=</span> <span class="token operator">&amp;</span><span class="token punctuation">(</span><span class="token operator">*</span>ptr<span class="token punctuation">)</span><span class="token operator">-</span><span class="token operator">&gt;</span>next_hash<span class="token punctuation">;</span>
	    <span class="token punctuation">}</span>
	    
	    <span class="token keyword">return</span> ptr<span class="token punctuation">;</span>
	<span class="token punctuation">}</span>
</code></pre>
<ul>
<li><code>LRUHandle** ptr = &amp;list_[hash &amp; (length_ - 1)]</code> - Find the bucket index  by perming hash modulo the number of buckets, the modulo is performed using an optimization to avoid an expensive division operation.<br>
The optimization assumes that length_ is a power of two, When <code>length_</code> is a power of two, subtracting 1 gives a <strong>bitmask</strong> that extracts only the lower bits of <code>hash</code>. for example:<br>
8  = 1000  (binary)<br>
7  = 0111  (binary)<br>
Doing <code>hash &amp; 7</code> extracts the lowest 3 bits of <code>hash</code>, ensuring the result is always between <code>0</code> and <code>7</code>.<br>
<code>hash &amp; (length_ - 1)</code> is equivalent to <code>hash % length_</code><br>
This is why many hash table implementations (like this LRU cache) <strong>always keep the array size as a power of two</strong>—so they can use this efficient masking trick.</li>
<li>The loop terminates when either the entry is found i.e. only when the input hash  and key matches or when reaching the end of the list returning a nullptr…</li>
</ul>
<hr>
<h4 id="insert">Insert</h4>
<pre class=" language-cpp"><code class="prism  language-cpp">	 LRUHandle<span class="token operator">*</span> <span class="token function">Insert</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span> h<span class="token punctuation">)</span> <span class="token punctuation">{</span>
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
<ol>
<li>Find if the entry already exists i.e. an handle with the same hash and key.</li>
<li>if exists, set the next field of the input handle to point to the handle that the old entry points to.</li>
<li>As ptr is of type pointer to pointer, performing <code>*ptr=h</code> will replace the LRUHandle which the returned entry pointing too.</li>
<li>if it’s a new entry i.e. FindPointer return nullptr, increment the number of stored objects and resize if it exceeds the length (TODO what length represents).</li>
</ol>
<p>–<br>
Resize</p>
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
<ul>
<li>calculate new_length to be the first power of two which is bigger than the current number of stored elements.</li>
<li>allocate a new <code>LRUHandle*</code> array of size <code>new_length</code>.</li>
<li>iterate over the current array, for each bucket iterate over its list and for each <code>LRUHandle*</code> in the list:<br>
- save the next LRUHandle* it points to in the hash list.<br>
- calculate the bucket of that hash in the new array and take the head of the list for this bucket.<br>
-  make the current entry as the head of the list by first, assigning the current head to its next_hash pointer <code>h-&gt;next_hash = *ptr</code> and then assigning the head pointer the point to it <code>*ptr = h</code><br>
- <code>h = next</code> move to the next entry which we saved before<br>
- increment the element counter.</li>
<li>once all elements are migrated to the new array, assert that the count matches the expected elements count.</li>
<li>delete the old array.</li>
<li>assign the new array to the list_ member and initialize the new length to be the length_.</li>
</ul>
<hr>
<h2 id="class-lrucache">Class LRUCache</h2>
<p>TODO describe how the LRU works<br>
A single sharded cache</p>
<h3 id="members-1">Members</h3>
<ul>
<li>size_t capacity, usage_ - When usage_ exceeds capacity_, eviction happens.</li>
<li>mutex _ - guard for thread safety.</li>
<li>LRUHandle  lru_ - a circular linked list of entries that are in the cache.</li>
<li>LRUHandle  in_use_ - a circular linked list of entries that are in use by clients.</li>
<li>HandleTable  table_ - a hash table for fast lookup of entries (instead of linear search in the linked list).</li>
</ul>
<h3 id="methods-1">Methods</h3>
<p>We’ll go first with the construction and the private methods as they define the low level building block that the public methods use</p>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">private</span><span class="token operator">:</span>
	<span class="token keyword">void</span>  <span class="token function">LRU_Remove</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span>  e<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token keyword">void</span>  <span class="token function">LRU_Append</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span>  list<span class="token punctuation">,</span> LRUHandle<span class="token operator">*</span>  e<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token keyword">void</span>  <span class="token function">Ref</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span>  e<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token keyword">void</span>  <span class="token function">Unref</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span>  e<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token keyword">bool</span>  <span class="token function">FinishErase</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span>  e<span class="token punctuation">)</span> <span class="token function">EXCLUSIVE_LOCKS_REQUIRED</span><span class="token punctuation">(</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre>
<h3 id="constructor">constructor</h3>
<pre class=" language-cpp"><code class="prism  language-cpp">  LRUCache<span class="token operator">::</span><span class="token function">LRUCache</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token operator">:</span> <span class="token function">capacity_</span><span class="token punctuation">(</span><span class="token number">0</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token function">usage_</span><span class="token punctuation">(</span><span class="token number">0</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
	<span class="token comment">// Make empty circular linked lists.</span>
	lru_<span class="token punctuation">.</span>next  <span class="token operator">=</span>  <span class="token operator">&amp;</span>lru_<span class="token punctuation">;</span>
	lru_<span class="token punctuation">.</span>prev  <span class="token operator">=</span>  <span class="token operator">&amp;</span>lru_<span class="token punctuation">;</span>
	in_use_<span class="token punctuation">.</span>next  <span class="token operator">=</span>  <span class="token operator">&amp;</span>in_use_<span class="token punctuation">;</span>
	in_use_<span class="token punctuation">.</span>prev  <span class="token operator">=</span>  <span class="token operator">&amp;</span>in_use_<span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre>
<p>The circular linked list is implemented by using a sentinel node that its next and prev pointers are initialized to point to itself, it’s a neat trick that eliminate the need to handle special cases and creates the circular list automictically.</p>
<h3 id="lru_append">LRU_Append</h3>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span>  LRUCache<span class="token operator">::</span><span class="token function">LRU_Append</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span>  list<span class="token punctuation">,</span> LRUHandle<span class="token operator">*</span>  e<span class="token punctuation">)</span> <span class="token punctuation">{</span>
	<span class="token comment">// Make "e" newest entry by inserting just before *list</span>
	e<span class="token operator">-</span><span class="token operator">&gt;</span>next  <span class="token operator">=</span>  list<span class="token punctuation">;</span>
	e<span class="token operator">-</span><span class="token operator">&gt;</span>prev  <span class="token operator">=</span>  list<span class="token operator">-</span><span class="token operator">&gt;</span>prev<span class="token punctuation">;</span>
	e<span class="token operator">-</span><span class="token operator">&gt;</span>prev<span class="token operator">-</span><span class="token operator">&gt;</span>next  <span class="token operator">=</span>  e<span class="token punctuation">;</span>
	e<span class="token operator">-</span><span class="token operator">&gt;</span>next<span class="token operator">-</span><span class="token operator">&gt;</span>prev  <span class="token operator">=</span>  e<span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre>
<p>The list is circular starting from the oldest items therefore new items are inserted as the last item pointing back to the <code>list</code> sentinel item.</p>
<h3 id="lru_remove">LRU_Remove</h3>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span>  LRUCache<span class="token operator">::</span><span class="token function">LRU_Remove</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span>  e<span class="token punctuation">)</span> <span class="token punctuation">{</span>
	e<span class="token operator">-</span><span class="token operator">&gt;</span>next<span class="token operator">-</span><span class="token operator">&gt;</span>prev  <span class="token operator">=</span>  e<span class="token operator">-</span><span class="token operator">&gt;</span>prev<span class="token punctuation">;</span>
	e<span class="token operator">-</span><span class="token operator">&gt;</span>prev<span class="token operator">-</span><span class="token operator">&gt;</span>next  <span class="token operator">=</span>  e<span class="token operator">-</span><span class="token operator">&gt;</span>next<span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre>
<p>removes the entry from the list.</p>
<h3 id="ref">Ref</h3>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span>  LRUCache<span class="token operator">::</span><span class="token function">Ref</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span>  e<span class="token punctuation">)</span> <span class="token punctuation">{</span>
	<span class="token keyword">if</span> <span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs  <span class="token operator">==</span>  <span class="token number">1</span>  <span class="token operator">&amp;&amp;</span>  e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache<span class="token punctuation">)</span> <span class="token punctuation">{</span> <span class="token comment">// If on lru_ list, move to in_use_ list.</span>
		<span class="token function">LRU_Remove</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
		<span class="token function">LRU_Append</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>in_use_<span class="token punctuation">,</span> e<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token punctuation">}</span>
	e<span class="token operator">-</span><span class="token operator">&gt;</span>refs<span class="token operator">++</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre>
<p>Increment the reference count,<br>
if the entry is in the <code>in_cache</code> list and its reference count is 1, move it to the in use list.</p>
<h3 id="unref">Unref</h3>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span>  LRUCache<span class="token operator">::</span><span class="token function">Unref</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span>  e<span class="token punctuation">)</span> <span class="token punctuation">{</span>
	<span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs  <span class="token operator">&gt;</span>  <span class="token number">0</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
	e<span class="token operator">-</span><span class="token operator">&gt;</span>refs<span class="token operator">--</span><span class="token punctuation">;</span>
	<span class="token keyword">if</span> <span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs  <span class="token operator">==</span>  <span class="token number">0</span><span class="token punctuation">)</span> <span class="token punctuation">{</span> <span class="token comment">// Deallocate.</span>
		<span class="token function">assert</span><span class="token punctuation">(</span><span class="token operator">!</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache<span class="token punctuation">)</span><span class="token punctuation">;</span>
		<span class="token punctuation">(</span><span class="token operator">*</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>deleter<span class="token punctuation">)</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> e<span class="token operator">-</span><span class="token operator">&gt;</span>value<span class="token punctuation">)</span><span class="token punctuation">;</span>
		<span class="token function">free</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token punctuation">}</span> <span class="token keyword">else</span>  <span class="token keyword">if</span> <span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache  <span class="token operator">&amp;&amp;</span>  e<span class="token operator">-</span><span class="token operator">&gt;</span>refs  <span class="token operator">==</span>  <span class="token number">1</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
		<span class="token comment">// No longer in use; move to lru_ list.</span>
		<span class="token function">LRU_Remove</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
		<span class="token function">LRU_Append</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>lru_<span class="token punctuation">,</span> e<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre>
<p>Handles decrementing the reference count of the entry,</p>
<ul>
<li>asserts that the entry ref count is positive as zero ref count entries are deleted.</li>
<li>decrement the ref count.</li>
<li>if the ref count is zero, asserts that the entry is not in use by the cache, invoke the custom deleter callback that the user gave for this entry, and deallocate the memory.</li>
<li>if the ref count is one and the entry is in the cache, remove it from the in use list and move to the lru list.</li>
</ul>
<h3 id="lookup">LookUp</h3>
<pre class=" language-cpp"><code class="prism  language-cpp">Cache<span class="token operator">::</span>Handle<span class="token operator">*</span>  LRUCache<span class="token operator">::</span><span class="token function">Lookup</span><span class="token punctuation">(</span><span class="token keyword">const</span>  Slice<span class="token operator">&amp;</span>  key<span class="token punctuation">,</span> uint32_t  hash<span class="token punctuation">)</span> <span class="token punctuation">{</span>
	MutexLock  <span class="token function">l</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>
	LRUHandle<span class="token operator">*</span>  e  <span class="token operator">=</span>  table_<span class="token punctuation">.</span><span class="token function">Lookup</span><span class="token punctuation">(</span>key<span class="token punctuation">,</span> hash<span class="token punctuation">)</span><span class="token punctuation">;</span>	
	<span class="token keyword">if</span> <span class="token punctuation">(</span>e  <span class="token operator">!=</span>  <span class="token keyword">nullptr</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
		<span class="token function">Ref</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token punctuation">}</span>
	<span class="token keyword">return</span>  <span class="token keyword">reinterpret_cast</span><span class="token operator">&lt;</span>Cache<span class="token operator">::</span>Handle<span class="token operator">*</span><span class="token operator">&gt;</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre>
<p>The public interface to find an item in the cache, for fast retrieval it uses the hash table to find the cached item.<br>
If found it increments the reference count .<br>
The return type is not the internal handle representation but the public opaque handle which is used by the client when calling to the cache APIs on that item.<br>
This design is used to decouple the user code from the cache code, enabling changes without breaking the public interface,</p>
<h3 id="release">Release</h3>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span>  LRUCache<span class="token operator">::</span><span class="token function">Release</span><span class="token punctuation">(</span>Cache<span class="token operator">::</span>Handle<span class="token operator">*</span>  handle<span class="token punctuation">)</span> <span class="token punctuation">{</span>
	MutexLock  <span class="token function">l</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token function">Unref</span><span class="token punctuation">(</span><span class="token keyword">reinterpret_cast</span><span class="token operator">&lt;</span>LRUHandle<span class="token operator">*</span><span class="token operator">&gt;</span><span class="token punctuation">(</span>handle<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre>
<p>The public api to release the cached item by the user, handling the opaque item pointer to the cache which is then casted to the concrete handler and caliing the UnRef with it.</p>
<h3 id="erase-and-finisherase">Erase and FinishErase</h3>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span>  LRUCache<span class="token operator">::</span><span class="token function">Erase</span><span class="token punctuation">(</span><span class="token keyword">const</span>  Slice<span class="token operator">&amp;</span>  key<span class="token punctuation">,</span> uint32_t  hash<span class="token punctuation">)</span> <span class="token punctuation">{</span>
	MutexLock  <span class="token function">l</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token function">FinishErase</span><span class="token punctuation">(</span>table_<span class="token punctuation">.</span><span class="token function">Remove</span><span class="token punctuation">(</span>key<span class="token punctuation">,</span> hash<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span> 

<span class="token keyword">bool</span>  LRUCache<span class="token operator">::</span><span class="token function">FinishErase</span><span class="token punctuation">(</span>LRUHandle<span class="token operator">*</span>  e<span class="token punctuation">)</span> <span class="token punctuation">{</span>
	<span class="token keyword">if</span> <span class="token punctuation">(</span>e  <span class="token operator">!=</span>  <span class="token keyword">nullptr</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
		<span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache<span class="token punctuation">)</span><span class="token punctuation">;</span>
		<span class="token function">LRU_Remove</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
		e<span class="token operator">-</span><span class="token operator">&gt;</span>in_cache  <span class="token operator">=</span>  <span class="token boolean">false</span><span class="token punctuation">;</span>
		usage_  <span class="token operator">-</span><span class="token operator">=</span>  e<span class="token operator">-</span><span class="token operator">&gt;</span>charge<span class="token punctuation">;</span>
		<span class="token function">Unref</span><span class="token punctuation">(</span>e<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token punctuation">}</span>
	<span class="token keyword">return</span>  e  <span class="token operator">!=</span>  <span class="token keyword">nullptr</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre>
<p>Erase only proxies the removed Handle from the hash table to the finish erase that removes the handle from the in use list i.e. active cache , it then calls Unref on it which decides whether to delete the Handle if if it’s reference count is 0.</p>
<h3 id="prune">Prune</h3>
<pre class=" language-cpp"><code class="prism  language-cpp"><span class="token keyword">void</span>  LRUCache<span class="token operator">::</span><span class="token function">Prune</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
	MutexLock  <span class="token function">l</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>
	<span class="token keyword">while</span> <span class="token punctuation">(</span>lru_<span class="token punctuation">.</span>next  <span class="token operator">!=</span>  <span class="token operator">&amp;</span>lru_<span class="token punctuation">)</span> <span class="token punctuation">{</span>
		LRUHandle<span class="token operator">*</span>  e  <span class="token operator">=</span>  lru_<span class="token punctuation">.</span>next<span class="token punctuation">;</span>
		<span class="token function">assert</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span>refs  <span class="token operator">==</span>  <span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
		<span class="token keyword">bool</span>  erased  <span class="token operator">=</span>  <span class="token function">FinishErase</span><span class="token punctuation">(</span>table_<span class="token punctuation">.</span><span class="token function">Remove</span><span class="token punctuation">(</span>e<span class="token operator">-</span><span class="token operator">&gt;</span><span class="token function">key</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span> e<span class="token operator">-</span><span class="token operator">&gt;</span>hash<span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
		<span class="token keyword">if</span> <span class="token punctuation">(</span><span class="token operator">!</span>erased<span class="token punctuation">)</span> <span class="token punctuation">{</span> <span class="token comment">// to avoid unused variable when compiled NDEBUG</span>
			<span class="token function">assert</span><span class="token punctuation">(</span>erased<span class="token punctuation">)</span><span class="token punctuation">;</span>
		<span class="token punctuation">}</span>
	<span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre>
<p>The cache logic manages the cached items in lru list the items that are not in use any more, ordered from the oldest</p>
<h3 id="insert-1">insert</h3>
<pre class=" language-cpp"><code class="prism  language-cpp">Cache<span class="token operator">::</span>Handle<span class="token operator">*</span> LRUCache<span class="token operator">::</span><span class="token function">Insert</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span> uint32_t hash<span class="token punctuation">,</span> <span class="token keyword">void</span><span class="token operator">*</span> value<span class="token punctuation">,</span>
                                size_t charge<span class="token punctuation">,</span>
                                <span class="token keyword">void</span> <span class="token punctuation">(</span><span class="token operator">*</span>deleter<span class="token punctuation">)</span><span class="token punctuation">(</span><span class="token keyword">const</span> Slice<span class="token operator">&amp;</span> key<span class="token punctuation">,</span>
                                                <span class="token keyword">void</span><span class="token operator">*</span> value<span class="token punctuation">)</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
  MutexLock <span class="token function">l</span><span class="token punctuation">(</span><span class="token operator">&amp;</span>mutex_<span class="token punctuation">)</span><span class="token punctuation">;</span>

  LRUHandle<span class="token operator">*</span> e <span class="token operator">=</span>
      <span class="token keyword">reinterpret_cast</span><span class="token operator">&lt;</span>LRUHandle<span class="token operator">*</span><span class="token operator">&gt;</span><span class="token punctuation">(</span><span class="token function">malloc</span><span class="token punctuation">(</span><span class="token keyword">sizeof</span><span class="token punctuation">(</span>LRUHandle<span class="token punctuation">)</span> <span class="token operator">-</span> <span class="token number">1</span> <span class="token operator">+</span> key<span class="token punctuation">.</span><span class="token function">size</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
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
<p>code analysis</p>
<ul>
<li>Lock the entire insertion operation.</li>
<li>allocate a new LRUHandle , recall the trick where the key allocation array is done dynamically at the end of the struct, this is the reason for the <code>- 1 + key.size())</code> when defining the size for the malloc.</li>
<li>some initializations happen next where the more interesting are the deleter which is a callback to delete (TODO) , charge which let the user determine the entry “cost” in the cache, and refs for reference counting.</li>
<li><code>std::memcpy(e-&gt;key_data, key.data(), key.size())</code> copies the key to the handle, while the value is not copied by pointing to the original value as keys are much shorter.</li>
<li>if the cache capacity is greater than zero:
<ul>
<li>increment the ref count.</li>
<li>mark the entry as in the cache.</li>
<li>append the entry to the in use list.</li>
<li>Add to the usage the given charge, note that charge is an input by the user i.e, the user can determine the “cost” of the entry and whether caching it should result in removing other elements.</li>
<li>Insert the entry to the hash table as well for fast lookup, in case the insert is an update to an existing key, the hash table insert will return a pointer to the old entry. which will be erased when calling finishErase on it.</li>
<li>TODO - why can capacity be 0.</li>
<li>on finish the item insertion the cache checks whether the usage threshold exceeded, if yes and the lru list is not empty , iterate over the lru list from the oldest element, and call finishErase on eaach item until usage is less then capacity or the entire lru list has been erased.</li>
</ul>
</li>
</ul>
<hr>
<h2 id="shardedlrucache">ShardedLRUCache</h2>
<p>by now we covered the infrastructure and logic to of single</p>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

