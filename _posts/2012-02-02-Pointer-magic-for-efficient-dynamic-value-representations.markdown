---
layout: post
title: Pointer magic for efficient dynamic value representations
excerpt: JS implementations use some neat tricks to achieve good performance. One of those tricks is a good bit of pointer magic to make dynamically typed values more efficient.
---
What always intrigued me were the various little tricks that JavaScript implementations use to be
as fast as they are. One of those tricks is how JS implementations represent values to be more time
and space efficient.

I think it's worth the time to look at those neat little tricks for two reasons: Firstly, some of
them are also applicable in other situations (though most of them admittedly are not). And secondly,
because they remind you of how various data is represented at the low level (stuff you may have
learned in your CS courses at some point - or probably not).

So, let's start!

The trivial approach: Tagged unions
-----------------------------------

JavaScript is a dynamically typed language and as such needs some kind of "value" data structure,
which stores the current type and the value in that type. The trivial approach to this problem is to
use a tagged union. This could look roughly like this:

{% highlight cpp %}
#include <stdint.h>

class Value {
public:
    enum TypeTag {
        IntType, DoubleType, StringType, ObjectType
    }

    // constructors, methods, ...
private:
    union {
        int32_t  asInt32;
        double   asDouble;
        String  *asString;
        Object  *asObject;
    } payload;
    TypeTag type;
};
{% endhighlight %}

As you can see the idea is very simple. You have a type tag, which specifies which type the value
currently has. And a union that's used to store the various types. Examples to store integers and
objects would look like that:

{% highlight cpp %}
inline Value::Value(int32_t number) {
    type = IntType;
    payload = number;
}
inline Value::Value(Object *object) {
    type = ObjectType;
    payload = object;
}
{% endhighlight %}

This straightforward approach has several problems: The size of the above object would be 16 bytes =
128 bits (on a 64 bit machine). This is a size that you wouldn't just pass around by value. So you
need to allocate it on the heap and use a pointer to it. Allocating stuff on the heap is slow and
incurs memory overhead. When using the value later it has to be fetched from the heap, which is
again slow.

So what we want is to make values smaller and move them off the heap. Before we get to that, let's
first have a look at a technique called pointer tagging:

An interlude: Pointer tagging
-----------------------------

Have a look at the above structure again. It consists of a union, which has a size of 8 bytes, and
an integer which fits into a single byte (a `char`). So theoretically the total size should be 9
bytes. But it's not - it's 16 bytes. This is because the compiler adds padding to the data structure
in order to align it in memory.

This is done for performance reasons: The CPU can access the memory only in word sized chunks. So if
our data always starts at a word it can be fetched efficiently. If it were to start somewhere in the
middle of a word the CPU would have to apply masks and shifts in order to get the data - which would
be too slow.

A word on a 64 bit machine is 64 bit = 8 bytes (obviously ^^). So if all pointers to our object will
be aligned to 8 bytes it means that they will be multiples of 8. A pointer thus can be 8, 16, 24,
109144, etc. But it can not be 7 or 13. Or speaking in binary: 0b1000 (=8), 0b10000 (=16),
0b11000 (=24), 0b11010101001011000 (=109144).

As you can see the lower three bits are always zero. So those three bits are basically free to use.
We can use them to store a "tag" in where, which is an integer between 0 (0b000) and 7 (0b111).

    pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppTTT
    ^- actual pointer                                   three tag bits -^

Here is a sample implementation of how to do so:

{% highlight cpp %}
#include <cassert>
#include <stdint.h>

template <typename T, int alignedTo>
class TaggedPointer {
private:
    static_assert(
        alignedTo != 0 && ((alignedTo & (alignedTo - 1)) == 0),
        "Alignment parameter must be power of two"
    );

    // for 8 byte alignment tagMask = alignedTo - 1 = 8 - 1 = 7 = 0b111
    // i.e. the lowest three bits are set, which is where the tag is stored
    static const intptr_t tagMask = alignedTo - 1;

    // pointerMask is the exact contrary: 0b...11111000
    // i.e. all bits apart from the three lowest are set, which is where the pointer is stored
    static const intptr_t pointerMask = ~tagMask;

    // save us some reinterpret_casts with a union
    union {
        T *asPointer;
        intptr_t asBits;
    };

public:
    inline TaggedPointer(T *pointer = 0, int tag = 0) {
        set(pointer, tag);
    }

    inline void set(T *pointer, int tag = 0) {
        // make sure that the pointer really is aligned
        assert((reinterpret_cast<intptr_t>(pointer) & tagMask) == 0);
        // make sure that the tag isn't too large
        assert((tag & pointerMask) == 0);

        asPointer = pointer;
        asBits |= tag;
    }

    inline T *getPointer() {
        return reinterpret_cast<T *>(asBits & pointerMask);
    }
    inline int getTag() {
        return asBits & tagMask;
    }
};
{% endhighlight %}

The code is fairly straightforward I think. You could then use it like this:

{% highlight cpp %}
double number = 17.0;
TaggedPointer<double, 8> taggedPointer(&number, 5);
taggedPointer.getPointer(); // == &number
taggedPointer.getTag(); // == 5
{% endhighlight %}

The technique described above is useful in many contexts. Basically it's applicable to any situation
where you want to add a small bit of info to an aligned pointer. And this is also the situation we
are in :)

But before we get to that let's first have a look at another, very similar trick:

Storing integers in the pointer
-------------------------------

JavaScript does not have an integer type per se, it only has IEEE 754 doubles. Still people mainly
work with small integral numbers (think of loops, array indexes, etc). Thus it makes sense to store
those small integers in a real integer variable instead of a double, because many operations are
much faster this way. That's also the reason why in the `Value` class above I have a distinct int32
type and not just a double type.

Especially for integers the above `Value` approach seems like overkill. One has to allocate 16 bytes
of memory (+ overhead) and has to maintain a pointer, just to store 4 bytes of data. That's
inefficient both in terms of memory and performance.

The solution obviously is to use something similar to the tagged pointers introduced above. You can
say: "If the last bit of the pointer is 1 then it actually isn't a pointer at all, but it's an
integer."

Here is how a pointer would look:

    pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppTT0
    ^- actual pointer                                     two tag bits -^ ^- last bit 0

And this is how the integer would look like (note that as we need one bit for distinguishing the
type the integer has only 63 bits):

    iiiiiiii|iiiiiiii|iiiiiiii|iiiiiiii|iiiiiiii|iiiiiiii|iiiiiiii|iiiiiii1
    ^- actual integer                                                     ^- last bit 1

To get the actual integer value we just need to shift off the 1 bit using `integer >> 1`.

Here again a sample implementation:

{% highlight cpp %}
#include <cassert>
#include <stdint.h>

template <typename T, int alignedTo>
class TaggedPointerOrInt {
private:
    static_assert(
        alignedTo != 0 && ((alignedTo & (alignedTo - 1)) == 0),
        "Alignment parameter must be power of two"
    );
    static_assert(
        alignedTo > 1,
        "Pointer must be at least 2-byte aligned in order to store an int"
    );

    // for 8 byte alignment tagMask = alignedTo - 1 = 8 - 1 = 7 = 0b111
    // i.e. the lowest three bits are set, which is where the tag is stored
    static const intptr_t tagMask = alignedTo - 1;

    // pointerMask is the exact contrary: 0b...11111000
    // i.e. all bits apart from the three lowest are set, which is where the pointer is stored
    static const intptr_t pointerMask = ~tagMask;

    // save us some reinterpret_casts with a union
    union {
        T *asPointer;
        intptr_t asBits;
    };

public:
    inline TaggedPointerOrInt(T *pointer = 0, int tag = 0) {
        setPointer(pointer, tag);
    }
    inline TaggedPointerOrInt(intptr_t number) {
        setInt(number);
    }

    inline void setPointer(T *pointer, int tag = 0) {
        // make sure that the pointer really is aligned
        assert((reinterpret_cast<intptr_t>(pointer) & tagMask) == 0);
        // make sure that the tag isn't too large
        assert(((tag << 1) & pointerMask) == 0);

        // last bit isn't part of tag anymore, but just zero, thus the << 1
        asPointer = pointer;
        asBits |= tag << 1;
    }
    inline void setInt(intptr_t number) {
        // make sure that when we << 1 there will be no data loss
        // i.e. make sure that it's a 31 bit / 63 bit integer
        assert(((number << 1) >> 1) == number);

        // shift the number to the left and set lowest bit to 1
        asBits = (number << 1) | 1;
    }

    inline T *getPointer() {
        assert(isPointer());

        return reinterpret_cast<T *>(asBits & pointerMask);
    }
    inline int getTag() {
        assert(isPointer());

        return (asBits & tagMask) >> 1;
    }
    inline intptr_t getInt() {
        assert(isInt());

        return asBits >> 1;
    }

    inline bool isPointer() {
        return (asBits & 1) == 0;
    }
    inline bool isInt() {
        return (asBits & 1) == 1;
    }
};
{% endhighlight %}

Usage example:

{% highlight cpp %}
// either a pointer to a double (with a two bit tag) or a 63 bit integer
double number = 17.0;
TaggedPointerOrInt<double, 8> taggedPointerOrInt(&number, 3);
taggedPointerOrInt.isPointer(); // == true;
taggedPointerOrInt.getPointer(); // == &number
taggedPointerOrInt.getTag(); // == 3

taggedPointerOrInt.setInt(123456789);
taggedPointerOrInt.isInt(); // == true
taggedPointerOrInt.getInt(); // == 123456789
{% endhighlight %}

Now using the above technique we have a powerful way to optimize the original Value class. Integers
will be stored directly in the pointer; doubles, strings and objects will be stored as a pointer
with a tag identifying their type (e.g 0 = 0b00 = object, 1 = 0b01 = string, 2 = 0b10 = double).
Booleans could be stored similarly to ints (and fetched using a simple `bool >> 3`):

    00000000|00000000|00000000|00000000|00000000|00000000|00000000|0000b110
                                                    actual bool value -^  ^- last bit 0 (as it isn't an int)
                           tag = 3 = 0b11 to identify that it's a bool -^^

So what have we gained overall? Quite a bit! Integers (and bools) aren't allocated on the heap
anymore. That saves memory and probably more importantly improves performance (because the code
doesn't have to access the heap). Also the pointers to strings and objects can now be dereferenced
directly, without first fetching a value (which stores a pointer to the real string/object).

But we aren't done yet: The double still requires a pointer; wouldn't it be nice to get rid of that
too?

Pointers in the NaN-Space
-------------------------

### Recap: How do doubles look like?

Doubles in JS are following the IEEE 754 Standard for Floating-Point Arithmetic. In C++ doubles
aren't specified to follow any standard, but we assume that they use IEEE 754, too (which,
practically speaking is a pretty good assumption. In general this whole article is nothing more than
a large set of evil assumptions that any self-respecting C++ developer would murder you for.)

As you probably know IEEE 754 doubles are internally represented using the following bit sequence:

    seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
     ^- exponent ^- mantissa
    ^- sign bit

The resulting float is basically (-1)^s * m * 2^e (speaking oversimplifyingly).

What is much more important to us are some special values that doubles can have:

    Positive zero (0.0):
    seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
    00000000|00000000|00000000|00000000|00000000|00000000|00000000|00000000
    ^- sign bit 0 (= +), all other bits also 0

    Negative zero (-0.0):
    seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
    10000000|00000000|00000000|00000000|00000000|00000000|00000000|00000000
    ^- sign bit 1 (= -), all other 0

IEEE defines two zeros: +0 and -0. Both have exponent and mantissa zeroed out and are distinguished
by the value of the sign bit. This may seem pointless at first - zero is zero after all - but it
allows for some subtle distinctions. For example 1.0 / 0.0 is +INF whereas 1.0 / -0.0 is -INF. There
are also other operations where the sign of the zero makes a difference.

One interesting behavior of the zeros is that they are considered equal in comparison. I.e.
0.0 == -0.0. The only way to find out whether a number is negative zero is to check whether the
sign bit is set:

{% highlight cpp %}
inline bool isNegativeZero(double number) {
    return number == 0 && *reinterpret_cast<int64_t *>(&number) != 0;
}
{% endhighlight %}

    Positive infinity (+INF):
    seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
    01111111|11110000|00000000|00000000|00000000|00000000|00000000|00000000
     ^- exponent bits all 1                          mantissa bits all 0 -^
    ^- sign bit 0 (= +)

    Negative infinity (-INF):
    seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
    11111111|11110000|00000000|00000000|00000000|00000000|00000000|00000000
     ^- exponent bits all 1                          mantissa bits all 0 -^
    ^- sign bit 1 (= -)

Infinity has all exponent bits set and a zero mantissa. Again there is +INF and -INF, distinguished
by the sign bit.

    Signaling NaN:
    seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
    s1111111|11110ppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp
                 ^- first mantissa bit 0    everything else is "payload" -^
     ^- exponent bits all 1
    ^- any sign bit

    Quiet NaN:
    seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
    s1111111|11111ppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp
                 ^- first mantissa bit 1    everything else is "payload" -^
     ^- exponent bits all 1                 and mustn't be all-zero (as it
    ^- any sign bit                         would be INF then)

NaNs represent values that are Not a Number. E.g. 0.0/0.0 = NaN. NaNs also have interesting
comparison semantics, as any comparison with NaN will be false (including NaN == NaN).

The representations of NaNs also have all exponent bits set (like infinity) but have a non-zero
mantissa. The sign bit is irrelevant for NaNs.

There are two types of NaNs: Signaling NaNs (sNaN) and quiet NaNs (qNaN). They are distinguished by
the first bit after the exponent: If it is 1 then the NaN is quiet; if it is 0 it is signaling.
Signaling NaNs - in theory - should throw an invalid operation exception (EM_INVALID) when they are
operated upon, whereas quiet NaNs should just be left alone. Practically this doesn't seem to be
used much, at least in MSVC this is disabled by default and needs to be enabled by compiler option +
`__controlfp()` call.

Where it starts to get interesting for us is that NaNs additionally encode a 51 bit "payload" in the
mantissa. This payload was originally designed to contain error information.

But deceitful as we are we will misuse that NaN payload to stuff other things in it - like integers
and pointers.

### 64 bit is a lie

On 64 bit architectures pointers have a size of 64 bits, obviously. But think about that again: How
much is 64 bit actually? That's 2^64 = 1.84467441 * 10^19 addressable memory bytes. That's 16 EiB
(Exabytes). Or talking in more convenient terms that's 17179869184 Gigabytes. I don't know about you
but I only have 4GB of memory. I've heard of exotic server setups that have around 800 GB of memory.
But even those abominations costing hundreds of thousands of dollars still would only need a 40 bit
pointer space (2^40 = 1024 GB).

Unsurprisingly we aren't the first to notice this: the x86-64 architecture utilizes only the lower
48 bits (which still allows 256 TiB) of a pointer. Additionally bits 63 through 48 must be copies
of bit 47. Pointers that follow this pattern are called canonical.

This leads to a strange looking address space (you can find a nicer version of this image on
[Wikipedia][1]):

    0xFFFFFFFF.FFFFFFFF
             ...        <- Canonical upper "half"
    0xFFFF8000.00000000
             ...
             ...        <- Noncanonical
             ...
    0x00007FFF.FFFFFFFF
             ...        <- Canonical lower "half"
    0x00000000.00000000

The operating system normally assigns only pointers from the lower half to applications, reserving
the upper half for itself. As always there are outliers - e.g. [Solaris assigns addresses from the
upper half][2] under some circumstances (mmap). But we'll just ignore that and assume that only the
lower half from 0x00000000.00000000 to 0x00007FFF.FFFFFFFF is used. (Again, all these evil
assumptions...)

At this point you should see what I am driving at: If pointers are really only 48 bits large they
fit perfectly into our 51 bit payload space.

### Sample implementation

Here is an implementation stub for doing so (just implements doubles, ints and void pointers; a
real implementation would look similar, just with more types):

{% highlight cpp %}
#include <stdint.h>
#include <cassert>

inline bool isNegativeZero(double number) {
    return number == 0 && *reinterpret_cast<int64_t *>(&number) != 0;
}

class Value {
private:
    union {
        double asDouble;
        uint64_t asBits;
    };

    static const uint64_t MaxDouble = 0xfff8000000000000;
    static const uint64_t Int32Tag  = 0xfff9000000000000;
    static const uint64_t PtrTag    = 0xfffa000000000000;
public:
    inline Value(const double number) {
        int32_t asInt32 = static_cast<int32_t>(number);

        // if the double can be losslessly stored as an int32 do so
        // (int32 doesn't have -0, so check for that too)
        if (number == asInt32 && !isNegativeZero(number)) {
            *this = Value(asInt32);
            return;
        }

        asDouble = number;
    }

    inline Value(const int32_t number) {
        asBits = number | Int32Tag;
    }

    inline Value(void *pointer) {
        // ensure that the pointer really is only 48 bit
        assert((reinterpret_cast<uint64_t>(pointer) & PtrTag) == 0);

        asBits = reinterpret_cast<uint64_t>(pointer) | PtrTag;
    }

    inline bool isDouble() {
        return asBits < MaxDouble;
    }
    inline bool isInt32() {
        return (asBits & Int32Tag) == Int32Tag;
    }
    inline bool isPointer() {
        return (asBits & PtrTag) == PtrTag;
    }

    inline double getDouble() {
        assert(isDouble());

        return asDouble;
    }
    inline int32_t getInt32() {
        assert(isInt32());

        return static_cast<int32_t>(asBits & ~Int32Tag);
    }
    inline void *getPointer() {
        assert(isPointer());

        return reinterpret_cast<void *>(asBits & ~PtrTag);
    }
};
{% endhighlight %}

Again, the code should be easy enough to understand. As an aid, here are the three consts from the
top in binary:

    Maximum double (qNaN with sign bit set without payload):
                        seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
    0xfff8000000000000: 11111111|11111000|00000000|00000000|00000000|00000000|00000000|00000000

    32 bit integer:
                        seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
    0xfff9000000000000: 11111111|11111001|00000000|00000000|iiiiiiii|iiiiiiii|iiiiiiii|iiiiiiii
                                                            ^- integer value

    Pointer:
                        seeeeeee|eeeemmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm|mmmmmmmm
    0xfffa000000000000: 11111111|11111010|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp|pppppppp
                                          ^- 48 bit pointer value


### A few more considerations

The above code makes doubles accessible directly, without further operations and accesses pointers
using a bit mask (that's what Mozilla does). An alternative implementation could do it the other way
around and make pointers accessible directly and doubles using an offset (i.e. you would add
`0x0001000000000000` on inserting a double and subtract it when retrieving it). The latter is what
WebKit does.

On 32 bit machines one can access the lower 32 bits of the 64 bit value directly. So both pointers
and integers can be accessed directly in any case. On the other hand, whereas the tagged pointer
with embedded integer approach would need only a 32 bit value on 32 bit machines, the NaN approach
will need a 64 bit value in both 32 bit and 64 bit environments. So on 32 bit two words need to be
passed around, instead of just one (that's why Mozilla calls this approach fatvals).

Still, considering the overall wins it is worth it: Both integers and doubles (and for that matter
also bools and all other "small" values) can be stored directly in the value, without accessing the
heap.

### Links

 * [value representation in javascript implementations][3] (article similar to this one, looking
   more at the general picture than at the concrete implementation)
 * [Mozilla's New Javascript Value Representation][8] (you'll have to scroll down a bit)
 * WebKit's implementation: [declaration][4]; [inline methods][5]; [methods][7]
 * [Mozilla's implementation][6]

 [1]: http://en.wikipedia.org/wiki/X86-64#Virtual_address_space_details
 [2]: https://bugzilla.mozilla.org/show_bug.cgi?id=577056
 [3]: http://wingolog.org/archives/2011/05/18/value-representation-in-javascript-implementations
 [4]: http://trac.webkit.org/browser/trunk/Source/JavaScriptCore/runtime/JSValue.h#L250
 [5]: http://trac.webkit.org/browser/trunk/Source/JavaScriptCore/runtime/JSValueInlineMethods.h
 [6]: http://dxr.lanedo.com/mozilla-central/js/src/jsval.h.html
 [7]: http://trac.webkit.org/browser/trunk/Source/JavaScriptCore/runtime/JSValue.cpp
 [8]: http://evilpie.github.com/sayrer-fatval-backup/cache.aspx.htm