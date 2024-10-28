---
title: Summary of Ulrich Drepper's legendary tls.pdf
draft: false
date: 2024-01-10
---

## Disclaimer:
1. The targeted audience of this document is dynamic linker & loader implementors, so important information for other tools (such as linkers and compilers) might be missing. 

2. This writeup is DEFINITELY not a substitute for the original document. It is highly recommended to read the original document before reading this summary. I wrote it to help me understand the document better, and to be able to reference things quickly if I forget something. It would be very hard to understand this mechanism without reading the original document first.

3. There are some differences of implementation across different architectures. I will only cover x86_64 since this is what I care about. If you happen to work on a different architecture, feel free to contribute :)

## Terminology:

- `Module` - A shared object or executable
- `modid (Module ID)` - A unique identifier for a module
- `tid (Thread ID)` - A unique identifier for a thread
- `TLS Image` - The data stored in the TLS segment of a module
- `TLS block` - An **initilized** TLS image allocated for a thread
- `TCB (Thread Control Block)` - A structure that contains information about a thread
- `TP (Thread Pointer)` - A register that points to the TCB of a thread.
- `DTV (Dynamic Thread Vector)` - A vector that contains pointers to the TLS blocks of a thread.

To refer to a structure belonging to some thread `i`, the notation `STRUCTUREi` is used (e.g. `TPi`, `TCBi`, etc).

## General Summary:

- TLS variables are stored in a similar format to regular variables (except the `SHF_TLS` flag is set in the section header). `tdata` and `tbss` are the TLS equivalent of `.data` and `.bss` sections.

- At runtime, the dynamic linker must prepare `.tbss` image - copy it and zero out the uncopied part.

- Since TLS segment isn't directly accessed (it is used as an image to create one of the blocks for each thread), the `ST_VALUE` field isn't used to store the offset/address to which the symbol will be loaded - instead, (in executables and shared objects) the `ST_VALUE` field is used to store the offset of the symbol in the TLS image.

- The TLS program header is all the dynamic linker needs. (contains `.tdata` and `.tbss`)

- The `DF_STATIC_TLS` flag marks that the module uses static TLS. (more on that later on)

- Efficient dynamic linkers should be able to lazily allocate TLS blocks - often times a thread will require only a few TLS variables in certain TLS images. (this method can't be _always_ used though. more on that later)

- TLS variables are identified using 
  1. the module they are defined in 
  2. the offset in the module's TLS image.

  This is different from regular variables, which are defined simply by an address/offset from load base address. So there is no issue in accessing them. 
  So there is a need for some mechanism to map addresses to TLS variables. This is where the following 2 variants come into play.

- Each architecture uses a different variant. (For more info checkout the original document)

### TLS Variant I:

![Variant I](./assets/tls-variant1.png)

- First thing `DTVi` points to is a number `GENi` which is used to resize the DTV.

- First field `TPi` points to must be `DTVi`. The rest of the fields are typical TCB information (most importantly pointer to the TLS block. But each implementation can add their own fields as they need).

- TLS blocks allocated at startup are allocated contiguously. That way they can accessed with a fixed offset from `TPi`.

- Access to TLS variables **which are in blocks loaded at startup** can be done using either:
  1. `DTVi` index (ie modid) + offset in the TLS image
  2. offset from the `TPi`

### TLS Variant II:

![Variant II](./assets/tls-variant2.png)

- Almost everything in Variant I applies here too, only differences are:
  1. The TLS blocks are allocated to the _left_ of `TPi` 
  2. there is no restriction for `TPi` to have its first field point to `DTVi`.


- In both variants, the position of the `TCB` is computed using architecture-specific formulas: TODO: check what the fuck this is 

$$ round(x, y) = y \cdot \lceil \frac{x}{y} \rceil $$

There are 2 TLS models a module can use - static and dynamic (NOT TO BE CONFUSED WITH STATIC AND DYNAMIC LINKING!)

### Static TLS:

- In this case, TLS images are allocated at program startup. The dynamic linker applies relocations at startup to construct the final address of the variable. 

### Dynamic TLS:

- In this model, TLS images are allocated sometime during program runtime (this model is used for example shared objects loaded by calling `dlopen`). The dynamic linker provides a function called `__tls_get_addr`, which allocated memory and returns the address of some variable as needed. (more info regarding that function later on)


### Startup & Later

- (depends on architecture and runtime, but in general) statically-linked executables do not load any modules dynamically (both at startup using `DT_NEEDED` and using at runtime `dlopen`). This means that dealing with TLS here is way simpler:
  1. There is only a single TLS image, so only a single TLS block for each thread 
  2. TLS blocks can be allocated at startup (they _can_ use dynamic TLS model, but there is no reason to do so)

- The mechanism of the dynamic linker to deal with TLS looks similar to the following:
  1. Go over each module and if it uses TLS, save important information from it (all of this is stored in the program header) (Ulrich proposed using a linked list for this, which makes sense, but there is no obligation to do so):
    a. Pointer to the modules TLS image
    b. Size of the TLS images
    c. tlsoffset (offset of the TLS block allocated from the `TPi`)
    d. flag indicating if the module uses static TLS
  2. This list will be extended when a module using TLS is loaded at runtime 

- The thread library uses this information to setup the TLS blocks for each thread.

- Ulrich also mentions the possibility of merging a few static TLS images into a single one to shorten the number of entries in the list. 

- It is also mentioned that in the case that all TLS images are using the static model then:

$$ tlssizeS = tlsoffsetM + tlssizeM $$

($$M$$ is the number of static TLS modules, $$S$$ refers to the merged TLS image)

- Of course if no TLS images use the static model, this space isn't allocated. 

- `modid`'s are used to index into the `DTV`. Runtime is free to choose how it wants to assign these `modid`'s, but the first module (the main executable) should always be assigned `modid` 1. Of course 2 modules can't have the same `modid` (UNLESS a module is unloaded. Once the module is unloaded, you can repurpose it's `modid` (as well as other resources assigned to it, such as TLS image entry for exmaple)).

### __tls_get_addr:

- See the original document for exact details for each architecture. In general, it recieves a `modid` and an offset for a variable in the TLS image. It checks if the current thread has a TLS block for that `modid`, if not it allocates and initilizes it. Then it returns the address of the variable in the TLS block.

- As you can see, `__tls_get_addr` lets us implement lazy allocation of TLS blocks.

- Theoretically there is no limit to the amount of TLS images a program could have loaded. So the `DTVi` should be able to grow dynamically as needed. This is exactly what the `GENi` field is for; when a new thread is created, we first check if the current size of `DTVi` exceeds `GENi`, if it does, we reallocate `DTVi` to be larger (and when we allocate, we allocate a few additional slots and increase `GENi` by that amount).

### Statically linked executables:

- As stated before, statically linked executables are _usually_ not allowed to load modules at runtime. (and even when they do, only simple modules not using TLS could be loaded). This means that on statically linked executables:
  1. There is only a single TLS image (that of the executable. 0 TLS images if the executable doesn't use TLS of course)
  2. That TLS block is allocated on program startup - so static TLS model is used.
  3. Since there is only a single TLS image, the offsets of the variables can be used as is. (no need for `modid`)

## TLS Access models (specifically for 0x86_64):

- Whilst regular symbol relocations result in some address, TLS symbol relocatoins should result in a `modid` and an offset in the TLS image.

- Since it is discouraged to modify `.text`, references to TLS variables use the GOT, and the relocation is done on the GOT entry.

There are 4 models for accessing TLS variables:

### General dynamic model

- Most generic model, and the least efficient. 
- This is the default model. The compiler will use a different model only when certain conditions are met, so a more efficient model is possible. 
- Code compiled with this model can be used everywhere, and access variables defined anywhere.
- Variable offset and `modid` are not known, they are decided by the dynamic linker at runtime. They are passed to `__tls_get_addr` which returns the final address of the variable.
- Since `__tls_get_addr` is used here, we can defer the allocation of TLS blocks until needed.

#### Implementation example for x86_64

Consider the following code snippet:

```c
extern __thread int x;

&x;
```

- `__tls_get_addr` is called with a pointer to an `tls_index` type. This type is defined as follows:

```c
typedef struct
{
  size_t ti_module;
  size_t ti_offset;
} tls_index;
```

- `R_X86_64_TLSGD` relocation instructs the linker to allocate a `tls_index` in the GOT (if the linker chooses GOT entry `n` to place this type in, `ti_module` and `ti_offset` will be placed at `GOT[n]` and `GOT[n+1]` respectively).

- `R_X86_64_DTPMOD64` relocation is used by the dynamic linker to set `ti_module` to the `modid` of the variable.
- `R_X86_64_DTPOFF64` relocation is used by the dynamic linker to set `ti_offset` to the offset of the variable in the TLS image.


### Local dynamic model

- An optimisation of the general dynamic model.
- This model is used when the compiler knows that the variable is defined in the same module it referenced in. (for example _static global_ or _protected/hidden_ variables). This optimisation is possible because we know the _offset_ at link time. 
- By passing `0` as the offset to `__tls_get_addr` we get the start of the TLS images. Then this value can be saved and easily just added to the offset of the variable to get the final address. Good compilers generate code that does this.


#### Implementation example for x86_64

Consider the following code snippet:

```c
static __thread int x1;
static __thread int x2;

&x1;
&x2;
```

- Since in this case we already have the offsets, the only relocation required is `R_X86_64_DTPMOD64`.

### Initial exec model

- This is an optimisation we can use when we know that the module uses the static TLS model. In that case, the TLS block is allocated at startup, so we can easily know the `modid` and the offset of the variable. All we need to figure out in runtime is the offset from the TLS image to TP.

#### Implementation example for x86_64

Consider again the following code snippet:

```c
extern __thread int x;

&x;
```
- Only relocation required is `R_X86_64_TPOFF64` it is processed by finding the symbol in the TLS image, then adding the variable offset to the offset of the image to TP, and writing it to the GOT entry (where the relocation needs to be applied).

### Local exec model

- This is an optimisation of the local dynamic module. This is the special case in which the module is the main executable. In this case, the `modid` is always 1, so it's TLS block is always the first one. That means we don't need to know _anything_ about the other blocks.

#### Implementation example for x86_64

Consider the following code snippet:

```c
static __thread int x;

&x;
```

- The only relocation required is `R_X86_64_TPOFF64`. The offset of the variable is known at link time, so the only thing that needs to be done is to add the offset from TP.

## My own important notes

- TLS symbols ALWAYS requires relocation - this is because the base address of the TLS block is decided by the dynamic linker at runtime.
- Even if some modules are using dynamic TLS, it might be faster and more efficient somtimes to allocate the TLS blocks rightway, instead of doing it lazily. This is usually true when the TLS image of the module is small anyway.
- The similar optimisation used for `DTVi` using `GENi` can be used for `DTV` as well (if your implementation holds all `DTVi` in one vector and not independently).


## Things I myself don't understand

- Computing the thread-specific address of a TLS variable is therefore a simple oper-
ation which can be performed by compiler-generated code which uses variant I. But it
cannot be done by the compiler for architectures following variant II and there is also
a good reason to not do it: deferred allocation (see below). - don't understand what this means
