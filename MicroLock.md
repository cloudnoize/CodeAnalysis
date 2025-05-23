
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
I'll go over the method in incremental order i.e. from low level functionality to more complex logic which uses that functionality. 

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
- - -

 **decodeDataFromWord()** - returns the data that is stored in the user bits of the lock.

    static  constexpr  uint8_t  decodeDataFromWord(
    uint32_t  word, unsigned  baseShift) noexcept {
	    return  static_cast<uint8_t>(
	    static_cast<uint8_t>(word  >>  baseShift) >>  kNumLockBits);
    }
Shift the byte of the lock to the right until it's  the most right bit, then shift the bits of the user data until they are the most right bits, and return the byte. 

---

 
 ```
 template <typename Func>
void MicroLockCore::unlockAndStoreWithModifier(Func modifier) noexcept {
    detail::Futex<>* wordPtr = word();
    uint32_t oldWord;
    uint32_t newWord;

    oldWord = wordPtr->load(std::memory_order_relaxed);
    do {
        assert(oldWord & heldBit());
        newWord = modifier(oldWord) & ~(heldBit() | waitBit());
    } while (!wordPtr->compare_exchange_weak(
        oldWord, newWord, std::memory_order_release, std::memory_order_relaxed));

    if (oldWord & waitBit()) {
        detail::futexWake(wordPtr, 1, heldBit());
    }
}

 ```
 - The method accepts a function object to enable the user the opportunity to store a value in the user data part of the lock.
 - it loads the word that holds the lock, is uses `memory_order_relaxed` which doesn't guarantee to read the latest value that was written to that address,  the lack of synchronization improves performance.
 - To my understanding the assertion `assert(oldWord  &  heldBit())` specifies that it's not defined to call unlock if that thread is not lock owner i.e. the assumption is a synchronized call to unlock without possible race conditions.
 - next the user function modifies the old value, to the  user value, the `~(heldBit() |  waitBit())` clears the held and wait bits.
- loop on `compare_exchange_weak` which tries to replace the oldWord with the new word, the memory order on success is `memory_order_release` which ensures that the compare operation is done on the latest value that is written to the word address, furthermore it synchronizes all future reads on that value to read the new value. this enable the usage of the relaxed load we saw before.
- after unlocking, check if at least one thread is sleeping waiting for the lock, if so call the futexWake to wake it. 
- Once a thread is woken it will try to lock the lock, we'll see that code later below.
---
## Class MicroLockBase
All the functions till now defines the infrastructure to manipulate the lock, now let's see how it's implemented.

    template <unsigned  MaxSpins  =  1000, unsigned  MaxYields  =  0>
    class  MicroLockBase : public  MicroLockCore
  The following class defines the lock interface, it inherits from the class we covered above.
  It accepts two template unsigned integers which determines the behavior of the slow path lock acquisition (we'll see the usage soon)

 ---
 **lockAndLoad()** - Try to acquire the lock in fast path, if fails degrades to slow path. returns the user stored data.
 
    template <unsigned  MaxSpins, unsigned  MaxYields>
    uint8_t  MicroLockBase<MaxSpins, MaxYields>::lockAndLoad() noexcept {
		    static_assert(MaxSpins + MaxYields < (unsigned)-1, "overflow")
		    detail::Futex<>* wordPtr =  word();
		    uint32_t oldWord;
		    oldWord =  wordPtr->load(std::memory_order_relaxed);
		    if ((oldWord &  heldBit()) ==  0  &&
			    wordPtr->compare_exchange_weak(
			    oldWord,
			    oldWord |  heldBit(),
			    std::memory_order_acquire,
			    std::memory_order_relaxed)) {
		    // Fast uncontended case: memory_order_acquire above is our barrier
			    return  decodeDataFromWord(oldWord |  heldBit());
		    } else {
		    // lockSlowPath doesn't call waitBit(); it just shifts the input bit. Make
		    // sure its shifting produces the same result a call to waitBit would.
		    assert(heldBit() <<  1  ==  waitBit());
		    // lockSlowPath emits its own memory barrier
		    return  lockSlowPath(oldWord, wordPtr, baseShift(), MaxSpins, MaxYields); 
	    }
    }

Finally, some action, let's see 

 - `static_assert(MaxSpins + MaxYields < (unsigned)-1, "overflow")` assert if the result of MaxSpins + MaxYields  is bigger than unsigned can hold, in case it is `MaxSpins + MaxYields` will be promoted to a bigger type than 32bits unsigned, the cast to `(unsigned)-1` results in a all 1 bit pattern i.e. the max unsigned value.
 - Atomic load the word that contains the lock, the load uses [memory_order_relaxed](https://en.cppreference.com/w/cpp/atomic/memory_order) not impose any synchronization across threads, it won't be a problem as it's being used in conjunction to  a compare exchange operation.
 - try to lock the lock in fast path:
 - if the lock is free try to perform compare exchange i.e. if the **current** value of the word equals the read value of oldWord, replace it with a locked version of the word. the success case will use memory_order_acquire acting as a **barrier** that prevents reordering of memory operations and ensures that the thread acquiring the lock sees all memory updates made by other threads before they released the lock.
 - if the thread succeeded to lock the lock ,return the user value stored in the lock.
 - if locking fails, we degrade to the **slowpath** locking, passing it the  `MaxSpins, MaxYields` template parameters which defies the behaviour of the slow path 
 - I don't understand the assertion and comment above the call  `lockSlowPath doesn't call waitBit(); it just shifts...`  i..e. why not use waitBit() . 
 - --
- **lockSlowPath**() - Preforms few waiting strategies for the lock to be free.
-   


    uint8_t MicroLockCore::lockSlowPath(
        uint32_t oldWord,
        detail::Futex<>* wordPtr,
        unsigned baseShift,
        unsigned maxSpins,
        unsigned maxYields) noexcept {
      uint32_t newWord;
      unsigned spins = 0;
      uint32_t heldBit = 1 << baseShift;
      uint32_t waitBit = heldBit << 1;
      uint32_t needWaitBit = 0;
    
    retry:
      if ((oldWord & heldBit) != 0) {
        ++spins;
        if (spins > maxSpins + maxYields) {
          newWord = oldWord | waitBit;
          if (newWord != oldWord) {
            if (!wordPtr->compare_exchange_weak(
                    oldWord,
                    newWord,
                    std::memory_order_relaxed,
                    std::memory_order_relaxed)) {
              goto retry;
            }
          }
          detail::futexWait(wordPtr, newWord, heldBit);
          needWaitBit = waitBit;
        } else if (spins > maxSpins) {
          std::this_thread::yield();
        } else {
          folly::asm_volatile_pause();
        }
        oldWord = wordPtr->load(std::memory_order_relaxed);
        goto retry;
      }
    
      newWord = oldWord | heldBit | needWaitBit;
      if (!wordPtr->compare_exchange_weak(
              oldWord,
              newWord,
              std::memory_order_acquire,
              std::memory_order_relaxed)) {
        goto retry;
      }
      return decodeDataFromWord(newWord, baseShift);
    }
    
- Performs a loop, for each time the lock is locked increment the spins counter and perform a wait based in the spins value:
- `if spins is lower than masSpins perform folly::asm_volatile_pause()` we didn't execute many wait rounds, there is hope to get the lock with short wait, it instructs the cpu to execute some pause instruction without performing context switch.
- `if spins is larger than maxSpins perform yield` might result in an expensive system call `sched_yield` which relinquishes the CPU entirely, allowing the scheduler to run other threads.
- if we kept spinning until spins is bigger than both maxSpins and maxYield we'll stop busy waiting and put the thread to sleep using the futex implementation which associates the address of the lock with a list of threads that waits for that lock and wakes a sleeping thread on a lock release event. to enable futex wait, the thread sets the wait bit on the lock, to signal the threads that holds the lock that some threads are asleep waiting for the lock. 
- node - it's not clear to me why memory_order_relaxed is used to store the waitBit as it doesn't guarantee any synchronization, resulting in an opportunity for the releasing thread to miss the update to my understanding.
- When eventually the lock is not held, the thread moves to try locking the lock, it "remembers" if it was asleep with futex, and sets the waitbit to true if so, by including `needWaitBit` in the final state of the lock (`newWord`), it ensures that the thread transitioning out of the waiting state doesn't inadvertently clear the bit, preserving the state for consistency.
- if the thread succeeded to perform the compare exchange, it finally has the lock! 
- return the user decoded value. 
 
 - List item

> Written with [StackEdit](https://stackedit.io/).





<!--stackedit_data:
eyJoaXN0b3J5IjpbMTE2NjI4NjY0OCwtMTQ3MTc1NjUwNCwxMD
AxMzQwOTc5LDE0Mjc1NzkyNDksLTIyMjc4NTM5MCw3MzU4MzM0
OTQsMTAyNTY0Njc5NSwyMzI1OTU1MDQsLTgwOTk0NDU4NywtNT
Y5OTMzNzgsNTA2NDY1NzAzLDYzOTE4NjMyNywtMTM4OTYxMTA5
OSw3Mjk1MzQxNjAsLTE3NTU4NzE3NjAsODgyNDU4ODI0LC0xMz
I4OTI2MjE3LC0xNTQ5MTMyMzUxLDIwNDY1MDgyMjYsLTgyNzk5
MDEyNl19
-->