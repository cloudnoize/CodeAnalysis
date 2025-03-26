---


---

<h1 id="leveldb-shardedlrucache">LevelDB ShardedLRUCache</h1>
<p>Before diving to the cache, we’ll review some data structures that support the implementation.</p>
<h3 id="lruhandle">LRUHandle</h3>
<p>Represents an entry in the cache</p>
<pre><code>struct  LRUHandle {
	void*  value;
	void (*deleter)(const  Slice&amp;, void*  value);
	LRUHandle*  next_hash;
	LRUHandle*  next;
	LRUHandle*  prev;
	size_t  charge; // TODO(opt): Only allow uint32_t?
	size_t  key_length;
	bool  in_cache; // Whether entry is in the cache.
	uint32_t  refs; // References, including cache reference, if present.
	uint32_t  hash; // Hash of key(); used for fast sharding and comparisons
	char  key_data[1]; // Beginning of key

	Slice  key() const {
	// next is only equal to this if the LRU handle is the list head of an
	// empty list. List heads never have meaningful keys.
	assert(next  !=  this);
	return  Slice(key_data, key_length);
	}
};
</code></pre>
<p>Let’s list some interesting features, without going to details on each field as we’re going to cover it when dealing the operations.</p>
<ul>
<li>value is an opaque pointer to the user value.</li>
<li>next and prev are used to compose the linked lists that are used for the LRU implementation.</li>
<li>hash defines to which shard this entry belongs to.</li>
<li>key_data[1]- is a trick to have a variable length array as a member of the struck without a need for extra allocation and management, it’s the last member of the struct which gets its allocation when allocating a new LRUHandle as follows <code>reinterpret_cast&lt;LRUHandle*&gt;(malloc(sizeof(LRUHandle) - 1 + key.size()));</code> i.e. in addition to the size of the LRUHandle struct the size of the key is added, resulting in space at the end of the struct that the key_data can store the whole key.</li>
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
<pre><code>LRUHandle** FindPointer(const Slice&amp; key, uint32_t hash) {
    LRUHandle** ptr = &amp;list_[hash &amp; (length_ - 1)];
    
    while (*ptr != nullptr &amp;&amp; ((*ptr)-&gt;hash != hash || key != (*ptr)-&gt;key())) {
        ptr = &amp;(*ptr)-&gt;next_hash;
    }
    
    return ptr;
}
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
<pre><code> LRUHandle* Insert(LRUHandle* h) {
    LRUHandle** ptr = FindPointer(h-&gt;key(), h-&gt;hash);
    LRUHandle* old = *ptr;
    h-&gt;next_hash = (old == nullptr ? nullptr : old-&gt;next_hash);
    *ptr = h;

    if (old == nullptr) {
        ++elems_;
        if (elems_ &gt; length_) {
            // Since each cache entry is fairly large, we aim for a small
            // average linked list length (&lt;= 1).
            Resize();
        }
    }
    
    return old;
    }
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
<p>A single sharded cache</p>
<h3 id="members-1">Members</h3>
<ul>
<li>size_t capacity, usage_ - When usage_ exceeds capacity_, eviction happens.</li>
<li>mutex _ - guard for thread safety.</li>
<li>LRUHandle  lru_ - a linked list of entries that are in the cache.</li>
<li>LRUHandle  in_use_ - linked list of entries that are in use by clients.</li>
<li>HandleTable  table_ - a hash table for fast lookup of entries (instead of linear search in the linked list).</li>
</ul>
<h3 id="methods-1">Methods</h3>
<h4 id="insert-1">insert</h4>
<pre><code>Cache::Handle* LRUCache::Insert(const Slice&amp; key, uint32_t hash, void* value,
                                size_t charge,
                                void (*deleter)(const Slice&amp; key,
                                                void* value)) {
  MutexLock l(&amp;mutex_);

  LRUHandle* e =
      reinterpret_cast&lt;LRUHandle*&gt;(malloc(sizeof(LRUHandle) - 1 + key.size()));
  e-&gt;value = value;
  e-&gt;deleter = deleter;
  e-&gt;charge = charge;
  e-&gt;key_length = key.size();
  e-&gt;hash = hash;
  e-&gt;in_cache = false;
  e-&gt;refs = 1;  // for the returned handle.
  std::memcpy(e-&gt;key_data, key.data(), key.size());

  if (capacity_ &gt; 0) {
    e-&gt;refs++;  // for the cache's reference.
    e-&gt;in_cache = true;
    LRU_Append(&amp;in_use_, e);
    usage_ += charge;
    FinishErase(table_.Insert(e));
  } else {  // don't cache. (capacity_==0 is supported and turns off caching.)
    // next is read by key() in an assert, so it must be initialized
    e-&gt;next = nullptr;
  }

  while (usage_ &gt; capacity_ &amp;&amp; lru_.next != &amp;lru_) {
    LRUHandle* old = lru_.next;
    assert(old-&gt;refs == 1);
    bool erased = FinishErase(table_.Remove(old-&gt;key(), old-&gt;hash));
    if (!erased) {  // to avoid unused variable when compiled NDEBUG
      assert(erased);
    }
  }

  return reinterpret_cast&lt;Cache::Handle*&gt;(e);
}
</code></pre>
<ul>
<li>Lock the mutex via a <a href="https://en.cppreference.com/w/cpp/language/raii">RAII</a> MutexLock object</li>
<li>The memory size to allocate is enough to hold the key in the array as described above.</li>
<li>initializing the LRUHandle members.</li>
<li>if cache is enabled i.e. capacity is bigger than 0:<br>
- increment the reference counter as it will be inserted to cache and set the in cache field to true.<br>
-  Append the new entry as the head of the in_use_ list<br>
- add the given “cost” of that entry to the cache to the total cache usage.<br>
- TODO analyze the hash table before continue and add the analysis before this class analysis</li>
<li></li>
</ul>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

