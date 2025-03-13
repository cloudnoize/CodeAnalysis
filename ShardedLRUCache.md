---


---

<h1 id="leveldb-shardedlrucache">LevelDB ShardedLRUCache</h1>
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
<hr>
<h2 id="class-lrucache">Class LRUCache</h2>
<p>A single sharded cache</p>
<h3 id="members">Members</h3>
<ul>
<li>size_t capacity, usage_ - When usage_ exceeds capacity_, eviction happens.</li>
<li>mutex _ - guard for thread safety.</li>
<li>LRUHandle  lru_ - a linked list of entries that are in the cache.</li>
<li>LRUHandle  in_use_ - linked list of entries that are in use by clients.</li>
<li>HandleTable  table_ - a hash table for fast lookup of entries (instead of linear search in the linked list).</li>
</ul>
<h3 id="methods">Methods</h3>
<h4 id="insert">insert</h4>
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

