
# C++ Code Analysis — `folly/MicroLock`

A great way to learn practical programming is by reading high-quality open source code and analyzing it. One excellent resource is [Facebook’s folly library](https://github.com/facebook/folly). This library provides a wide variety of components, utilities, and algorithms. It delves into low-level machine operations, incorporates smart optimizations, and is written with great care.

In this post, I will analyze the `MicroLock`, a tiny exclusive lock that uses just 2 bits. It is stored in a single byte and offers APIs for using the remaining 6 bits to store user data. A detailed description of the lock, its usage, and its performance can be found in the header file `folly/MicroLock.h`.

I will explain some of the less intuitive and tricky parts of the code in detail.

## Class MicoLockCore
### Members
    uint8_t  lock_{};
    static  constexpr  unsigned  kNumLockBits  =  2;
	static  constexpr  uint8_t  kLockBits  =
		static_cast<uint8_t>((1  <<  kNumLockBits) -  1);
	static  constexpr  uint8_t  kDataBits  =  static_cast<uint8_t>(~kLockBits);

    

 - *`uint8_t  lock_`*  - Represents the lock. As described, it uses 2 bits for the lock, while the remaining 6 bits are reserved for storing user data.
 - *`static constexpr unsigned kNumLockBits = 2`* -  A constant that defines the number of bits used for locking.
 - *`static  constexpr  uint8_t  kLockBits  = static_cast<uint8_t>((1  <<  kNumLockBits) -  1)`* -  mask for the lock bits.
	 -  Shift 1 by kNumLockBits (2) to the left i.e. `0xb100` The result **type** of a shift operation is the type of the left operand after integral promotions have been applied which is an **int** in this case.
	 - Subtracting 1 from a power of two results in a binary mask where all bits up to the power bit are set to `1`. For example: `0xb100 - 1 = 0xb011`.
	 -  The `static_cast<uint8_t>` is necessary because as described the resulting type of the bitwise operation is an int.
- `static  constexpr  uint8_t  kDataBits  =  static_cast<uint8_t>(~kLockBits)` - mask for the user data bits.
	- The `~` operator flips all bits, creating a mask for the user data bits.

### Methods


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjUxMjk5NjQxLDIwOTAzMTc0MjBdfQ==
-->