---


---

<p>Reading code from google is a free method of learning from great minds, <strong>LevelDB</strong> is a fast, lightweight key-value store it is designed for high-performance storage using <strong>log-structured merge-trees (LSM-trees)</strong>,<br>
LSM only appends records and does not update i.e. multiple instances of a key can be present but the most recent update is the current version of the key.<br>
LSM maintains a set of files, where each file has a sorted run of a range of keys.<br>
To find the most recent version of a key, if the key is not present in the memory or cache, the database needs to scan the files from the newest to the oldest.<br>
The scan  finishes when the first instance of the key is found, this scan can be very expensive if the database is big with many files, moreover, if the key is not present, the scan will (in theory) should scan all the files.<br>
Of course, optimizations were implemented to reduce the scan to be very fast, for example, the range that the file contains is listed in the metadata block, and if the searched key is not within the rage, the file is skipped.<br>
Another highly efficient and elegant technique is the usage of bloom filters, where each file contains an array which probabilisticly resemebles which keys may be present in this files and more importantly, which keys are for sure not present in this file.<br>
So when searching for a key, the database queries the bloom filter if the key might be present in the file, if not, we can skip this file with certentity that the key is not there.<br>
The probablity that the bloom filter will response with false positive is dependent on the configred size of the bloom.<br>
Will go over level db implementation to and usage to see .</p>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

