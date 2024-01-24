# Back Engineering Labs Fork

I just added cmkr/cmake as the build system and changed a few lines of code. Mostly i added MSVC pre-defined macros to determine the
width of the `ee_ptr_int`. At Back Engineering Labs we are building an adhoc recompiler that will also be the base of for our inhouse bin2bin obfuscator. 
We need to have some benchmarks to see how passes effect performance.

### Build

Build 64bit

```
cmake -B .build
cd .build
cmake --build . --config Release
```

Build 32bit

```
cmake -B .build -DCMAKE_GENERATOR_PLATFORM=WIN32
cd .build
cmake --build . --config Release
```

# Log File Format
The log files have the following format

~~~
2K performance run parameters for coremark.	(Run type)
CoreMark Size    	: 666					(Buffer size)
Total ticks			: 25875					(platform dependent value)
Total time (secs) 	: 25.875000				(actual time in seconds)
Iterations/Sec 		: 3864.734300			(Performance value to report)
Iterations			: 100000				(number of iterations used)
Compiler version	: GCC3.4.4				(Compiler and version)	
Compiler flags		: -O2					(Compiler and linker flags)
Memory location		: Code in flash, data in on chip RAM
seedcrc				: 0xe9f5				(identifier for the input seeds)
[0]crclist			: 0xe714				(validation for list part)
[0]crcmatrix		: 0x1fd7				(validation for matrix part)
[0]crcstate			: 0x8e3a				(validation for state part)
[0]crcfinal			: 0x33ff				(iteration dependent output)
Correct operation validated. See README.md for run and reporting rules.  (*Only when run is successful*)
CoreMark 1.0 : 6508.490622 / GCC3.4.4 -O2 / Heap 						  (*Only on a successful performance run*)
~~~

# Theory of Operation

This section describes the initial goals of CoreMark and their implementation.

## Small and easy to understand

* X number of source code lines for timed portion of the benchmark.
* Meaningful names for variables and functions.
* Comments for each block of code more than 10 lines long.

## Portability

A thin abstraction layer will be provided for I/O and timing in a separate file. All I/O and timing of the benchmark will be done through this layer.

### Code / data size

* Compile with gcc on x86 and make sure all sizes are according to requirements. 
* If dynamic memory allocation is used, take total memory allocated into account as well. 
* Avoid recursive functions and keep track of stack usage.
* Use the same memory block as data site for all algorithms, and initialize the data before each algorithm – while this means that initialization with data happens during the timed portion, it will only happen once during the timed portion and so have negligible effect on the results.

## Controlled output

This may be the most difficult goal. Compilers are constantly improving and getting better at analyzing code. To create work that cannot be computed at compile time and must be computed at run time, we will rely on two assumptions:

* Some system functions (e.g. time, scanf) and parameters cannot be computed at compile time.  In most cases, marking a variable volatile means the compiler is force to read this variable every time it is read. This will be used to introduce a factor into the input that cannot be precomputed at compile time. Since the results are input dependent, that will make sure that computation has to happen at run time.

* Either a system function or I/O (e.g. scanf) or command line parameters or volatile variables will be used before the timed portion to generate data which is not available at compile time. Specific method used is not relevant as long as it can be controlled, and that it cannot be computed or eliminated by the compiler at compile time. E.g. if the clock() functions is a compiler stub, it may not be used. The derived values will be reported on the output so that verification can be done on a different machine. 

* We cannot rely on command line parameters since some embedded systems do not have the capability to provide command line parameters. All 3 methods above will be implemented (time based, scanf and command line parameters) and all 3 are valid if the compiler cannot determine the value at compile time. 

* It is important to note that The actual values that are to be supplied at run time will be standardized. The methodology is not intended to provide random data, but simply to provide controlled data that cannot be precomputed at compile time.

* Printed results must be valid at run time. This will be used to make sure the computation has been executed.

* Some embedded systems do not provide “printf” or other I/O functionality. All I/O will be done through a thin abstraction interface to allow execution on such systems (e.g. allow output via JTAG).

## Key Algorithms

### Linked List

The following linked list structure will be used:

~~~
typedef struct list_data_s {
	ee_s16 data16;
	ee_s16 idx;
} list_data;

typedef struct list_head_s {
	struct list_head_s *next;
	struct list_data_s *info;
} list_head;
~~~

While adding a level of indirection accessing the data, this structure is realistic and used in many embedded applications for small to medium lists.

The list itself will be initialized on a block of memory that will be passed in to the initialization function. While in general linked lists use malloc for new nodes, embedded applications sometime control the memory for small data structures such as arrays and lists directly to avoid the overhead of system calls, so this approach is realistic.

The linked list will be initialized such that 1/4 of the list pointers point to sequential areas in memory, and 3/4 of the list pointers are distributed in a non sequential manner. This is done to emulate a linked list that had add/remove happen for a while disrupting the neat order, and then a series of adds that are likely to come from sequential memory locations.

For the benchmark itself:
- Multiple find operations are going to be performed. These find operations may result in the whole list being traversed. The result of each find will become part of the output chain.
- The list will be sorted using merge sort based on the data16 value, and then derive CRC of the data16 item in order for part of the list. The CRC will become part of the output chain.
- The list will be sorted again using merge sort based on the idx value. This sort will guarantee that the list is returned to the primary state before leaving the function, so that multiple iterations of the function will have the same result. CRC of the data16 for part of the list will again be calculated and become part of the output chain.

The actual `data16` in each cell will be pseudo random based on a single 16b input that cannot be determined at compile time. In addition, the part of the list which is used for CRC will also be passed to the function, and determined based on an input that cannot be determined at run time.

### Matrix Multiply

This very simple algorithm forms the basis of many more complex algorithms. The tight inner loop is the focus of many optimizations (compiler as well as hardware based) and is thus relevant for embedded processing. 

The total available data space will be divided to 3 parts:
1. NxN matrix A.
2. NxN matrix B.
3. NxN matrix C.

E.g. for 2K we will have 3 12x12 matrices (assuming data type of 32b 12(len)*12(wid)*4(size)*3(num) =1728 bytes). 

Matrix A will be initialized with small values (upper 3/4 of the bits all zero).
Matrix B will be initialized with medium values (upper half of the bits all zero).
Matrix C will be used for the result.

For the benchmark itself:
- Multiple A by a constant into C, add the upper bits of each of the values in the result matrix. The result will become part of the output chain.
- Multiple A by column X of B into C, add the upper bits of each of the values in the result matrix. The result will become part of the output chain.
- Multiple A by B into C, add the upper bits of each of the values in the result matrix. The result will become part of the output chain.

The actual values for A and B must be derived based on input that is not available at compile time.

### State Machine

This part of the code needs to exercise switch and if statements. As such, we will use a small Moore state machine. In particular, this will be a state machine that identifies string input as numbers and divides them according to format.

The state machine will parse the input string until either a “,” separator or end of input is encountered. An invalid number will cause the state machine to return invalid state and a valid number will cause the state machine to return with type of number format (int/float/scientific).

This code will perform a realistic task, be small enough to easily understand, and exercise the required functionality. The other option used in embedded systems is a mealy based state machine, which is driven by a table. The table then determines the number of states and complexity of transitions. This approach, however, tests mainly the load/store and function call mechanisms and less the handling of branches. If analysis of the final results shows that the load/store functionality of the processor is not exercised thoroughly, it may be a good addition to the benchmark (codesize allowing).

For input, the memory block will be initialized with comma separated values of mixed formats, as well as invalid inputs.

For the benchmark itself:
- Invoke the state machine on all of the input and count final states and state transitions. CRC of all final states and transitions will become part of the output chain.
- Modify the input at intervals (inject errors) and repeat the state machine operation.
- Modify the input back to original form.

The actual input must be initialized based on data that cannot be determined at compile time. In addition the intervals for modification of the input and the actual modification must be based on input that cannot be determined at compile time.

# Validation

This release was tested on the following platforms:
* x86 cygwin and gcc 3.4 (Quad, dual and single core systems)
* x86 linux (Ubuntu/Fedora) and gcc (4.2/4.1) (Quad and single core systems)
* MIPS64 BE linux and gcc 3.4 16 cores system
* MIPS32 BE linux with CodeSourcery compiler 4.2-177 on Malta/Linux with a 1004K 3-core system
* PPC simulator with gcc 4.2.2 (No OS)
* PPC 64b BE linux (yellowdog) with gcc 3.4 and 4.1 (Dual core system)
* BF533 with VDSP50 
* Renesas R8C/H8 MCU with HEW 4.05
* NXP LPC1700 armcc v4.0.0.524
* NEC 78K with IAR v4.61
* ARM simulator with armcc v4

# Memory Analysis

Valgrind 3.4.0 used and no errors reported.
	
# Balance Analysis

Number of instructions executed for each function tested with cachegrind and found balanced with gcc and -O0.

# Statistics

Lines:
~~~
Lines  Blank  Cmnts  Source     AESL     
=====  =====  =====  =====  ==========  =======================================
  469     66    170    251       627.5  core_list_join.c  (C)
  330     18     54    268       670.0  core_main.c  (C)
  256     32     80    146       365.0  core_matrix.c  (C)
  240     16     51    186       465.0  core_state.c  (C)
  165     11     20    134       335.0  core_util.c  (C)
  150     23     36     98       245.0  coremark.h  (C)
 1610    166    411   1083      2707.5  ----- Benchmark -----  (6 files)
  293     15     74    212       530.0  linux/core_portme.c  (C)
  235     30    104    104       260.0  linux/core_portme.h  (C)
  528     45    178    316       790.0  ----- Porting -----  (2 files)
 
* For comparison, here are the stats for Dhrystone
Lines  Blank  Cmnts  Source     AESL     
=====  =====  =====  =====  ==========  =======================================
  311     15    242     54       135.0  dhry.h  (C)
  789    132    119    553      1382.5  dhry_1.c  (C)
  186     26     68    107       267.5  dhry_2.c  (C)
 1286    173    429    714      1785.0  ----- C -----  (3 files)
~~~

# Credits
Many thanks to all of the individuals who helped with the development or testing of CoreMark including (Sorted by company name; note that company names may no longer be accurate as this was written in 2009).
* Alan Anderson, ADI
* Adhikary Rajiv, ADI
* Elena Stohr, ARM
* Ian Rickards, ARM
* Andrew Pickard, ARM
* Trent Parker, CAVIUM
* Shay Gal-On, EEMBC
* Markus Levy, EEMBC
* Peter Torelli, EEMBC
* Ron Olson, IBM
* Eyal Barzilay, MIPS
* Jens Eltze, NEC
* Hirohiko Ono, NEC
* Ulrich Drees, NEC
* Frank Roscheda, NEC
* Rob Cosaro, NXP
* Shumpei Kawasaki, RENESAS

# Legal
Please refer to LICENSE.md in this repository for a description of your rights to use this code.

# Copyright
Copyright © 2009 EEMBC All rights reserved. 
CoreMark is a trademark of EEMBC and EEMBC is a registered trademark of the Embedded Microprocessor Benchmark Consortium.

