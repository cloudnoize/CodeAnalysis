
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
In very high level, the logic is divided to a fast path of try acquiring the lock bit with a compare exchange operation, and a slow path where the compare exchange fails and some other strategies are tried, where the least option is to use `futex` to put the thread to sleep until its woken up.
The slow path implementation has some very interesting details. 

---

**word()** - returns a pointer to a 32 bit aligned address that contains the address of the lock.

    inline  detail::Futex<>*  MicroLockCore::word() const  noexcept {
	    uintptr_t lockptr = (uintptr_t)&lock_;
		lockptr &=  ~(sizeof(uint32_t) -  1);
	    return (detail::Futex<>*)lockptr; 
	}
*Analysis* 
 - The return type is defined as follows
     `<template <typename> class Atom = std::atomic>
        	  using Futex = Atom<std::uint32_t>;`
 - The futex is used for the slow path,  and it operates on a shared 32 bit unsigned integer as done in the ParkingLot class where it queue threads that waits on a lock by its address.
 - first we store the address of the lock in an unsigned integer type guaranteed to be large enough to hold a pointer.
 - the expression `lockptr &=  ~(sizeof(uint32_t) -  1)` 
	 - Create a mask which aligns an address to the nearest 32-bit boundary by decrementing 1 from a power of two (the sizeof()) and flipping its bits e.g. 0xb...11111100 .
	 - `lockptr &` - Apply the the mask to clear to lowest bits of lockptr, this rounds lockptr down to the nearest uint32_t boundary.
 - return the aligned address, casted to the Futex*. 
 - --
  **baseShift()** - calculates how many bits should be shifted from the beginning of the word to reach the first bit of the lock. 

    inline  unsigned  MicroLockCore::baseShift() const  noexcept {
	    unsigned offset_bytes = (unsigned)((uintptr_t)&lock_ - (uintptr_t)word());
	   
	    return  static_cast<unsigned>( 
	    kIsLittleEndian ?  CHAR_BIT  * offset_byte
	    :  CHAR_BIT  * (sizeof(uint32_t) - offset_bytes -  1));
    }
    	 
*Analysis*

 - The expression `(unsigned)((uintptr_t)&lock_ - (uintptr_t)word())` casts the lock and word addresses to numeric value and performs subtraction to determine the byte offset of the lock within the word.
 - The next expression determines how many bits should be shifted in order to reach the first **bit** of the lock,  it got me a little confused as I thought that the byte difference is constant regardless of the architecture i.e. little vs big endian. hence the bit shift should be multiplication of the byte difference and the bits per byte (CHAR_BIT). 
 But the calculation determines the bit position of the lock which is depends on how to system orders the bytes.
 After some discussions with ChatGPT I got to the following reply that satisfied my understanding

> **Little-Endian vs. Big-Endian Memory Layout**  
> Consider a `uint32_t` (4 bytes = 32 bits) stored in memory at address `0x4`:  
> 
> **Little-Endian**:  
> The least significant byte (LSB) is stored first, at the lowest address (`0x4`).  
> **Memory layout**:  
> ```
> Address:     0x4      0x5      0x6      0x7  
> Byte Order: [LSB]    [Byte1]  [Byte2]  [MSB]  
> Bits:       [0-7]    [8-15]   [16-23]  [24-31]  
> ```
> In little-endian, the byte at address `0x6` (where `lock_` is) corresponds to **bits 16–23** of the word starting at `0x4`.  
> **Bit shift calculation**:  
> ```
> Shift = 8 * offset_bytes = 8 * 2 = 16 bits  
> ```
> 
> **Big-Endian**:  
> The most significant byte (MSB) is stored first, at the lowest address (`0x4`).  
> **Memory layout**:  
> ```
> Address:     0x4      0x5      0x6      0x7  
> Byte Order: [MSB]    [Byte1]  [Byte2]  [LSB]  
> Bits:       [24-31]  [16-23]  [8-15]   [0-7]  
> ```
> In big-endian, the byte at address `0x6` (where `lock_` is) corresponds to **bits 8–15** of the word starting at `0x4`.  
> **Bit shift calculation**:  
> ```
> Shift = 8 * (sizeof(uint32_t) - offset_bytes - 1)  
>       = 8 * (4 - 2 - 1) = 8 bits  
> ```
> 
> **Why is the Shift Different?**  
> - **Byte Offset is Absolute**: The `lock_` is always **2 bytes** away from the start of the word (`0x4`). This does not depend on endianness.  
> - **Bit Interpretation is Endian-Dependent**: Endianness affects how bytes are arranged in memory. The byte at `0x6` corresponds to **bits 16–23** in little-endian and **bits 8–15** in big-endian.  
> Thus, the shift calculation accounts for the **bit position within the word**, which changes with the memory layout.

---
**heldBit**() - Returns a mask which is used to test whether the lock is locked.
   

     inline  unsigned  MicroLockCore::heldBit() const  noexcept {
    	    return  1U  << (baseShift() +  0);
        }
---

    
**waitBit()** - Returns a mask to the bit past the lock bit, which represents whether another thread is waiting for the lock.

    inline  unsigned  MicroLockCore::waitBit() const  noexcept {
        return  1U  << (baseShift() +  1);
    }
    
---
**encodeDataToWord()** - Stores user data in the redundant bits of the lock. 

    static  constexpr  uint32_t  encodeDataToWord(uint32_t  word, unsigned  shiftToByte, uint8_t  value) noexcept {
	    const  uint32_t  preservedBits  =  word  &  ~(kDataBits  <<  shiftToByte);
	    const  uint32_t  newBits  =  encodeDataToByte(value) <<  shiftToByte;
	    return  preservedBits  |  newBits;
    }

 - `preservedBits = word  &  ~(kDataBits  <<  shiftToByte)` recall that kDataBits is a uint8 mask for the 6 user bits, the mask is shifted to the location of the lock within the word. within the parenthesis the result is a 32 bit unsigned mask, where 6 bits in the location of the user bits are lit. the `~` operator inverse the bits creates a mask for all the bits besides the user bits. the `&` with the `word` stores in  `preservedBits` all the bits that are set in the word besides the user bits. 
 - `const  uint32_t  newBits  =  encodeDataToByte(value) <<  shiftToByte` encodeDataToByte simply shifts the value, `kNumLockBits` (2) to the left and returns a byte with the value encoded in the correct position, then we shift that byte to the lock address within the word, the final result `newBits` encodes just the value bits within a 32 bit unsigned. the return preserves all the bits of the word beside the data bits which now contains the input value.
 

 

---
Now that we have all the infrastructure to access and test the lock bits, we can start doing the fun stuff

> Written with [StackEdit](https://stackedit.io/).


<!--stackedit_data:
eyJoaXN0b3J5IjpbMTM3NDU1NDM2MF19
-->