---
layout: post
title: "AI without runtime: Building a neural network in TypeScript's type system"
author: Alexandra
tags:
  - TypeScript
  - machine learning
---

TypeScript's type system turns out to be a Turing-complete programming language.
However, because it is a type system, your IDE might just perform the type checking without you ever having to run any code.
Let's exploit that and build an AI&trade; that can be trained and do predictions without a runtime.

<!--more-->

The goal of this blog post is to torture TypeScript's type system into performing the dark magic known as machine learning.
Some familiarity with programming maybe even with TypeScript or a cool functional programming language will help.
I will explain the features of the type system that are used when they appear.
But be aware that the type system is actually much more powerful than what is exploited in the following text.

So how do we implement a neural network in a type system?
We don't have arithmetics over any form of numbers. And on a more fundamental level, we also don't have boolean arithmetics.
So we simply have to start by constructing boolean arithmetics, use these to construct bit sequences, turn these into signed integers and
then into fixed-width decimals. Once we got an approximation of real numbers going, we can construct vectors and implement operations on these vectors.
Neural networks are nothing more than vector-churning monstrosities, so at that point we have unlocked all the parts to build and train one.

The table of contents agrees with the previous description with a tiny hiccup, first we need to engage in some trickery in the first section:
* toc
{:toc}

You can find the source code at [https://github.com/AlexandraBaier/type-system-neural-network](https://github.com/AlexandraBaier/type-system-neural-network).
If you have comments, feedback or did something similar, feel free to create an issue in the repo to tell me about it. (Alternatively, I have an email address at the bottom of the page.)

## Preliminary trickery

First, we will need a way to check if our types actually do what we think they do.
Since I refuse to actively engage in running any code, this needs to run within the type system:

{% highlight TypeScript %}
{% raw %}
const typeEqual = <T1 extends T2, T2 extends Q, Q = T1>(): void => undefined as any;
{% endraw %}
{% endhighlight %}

The function itself does not do anything, its only purpose is to act as a vessel of our will to compare the two
type parameters `T1` and `T2`. We achieve this by declaring that the two type parameters extend each other.
Since we cannot directly declare this, we introduce a helper generic default parameter `Q`, which is assigned the parameter `T1`.

Next, we need to be able to index array types (not actual arrays).
To do that we need the ability to construct positive numbers and zero 
and increment/decrement these numbers.

{% highlight TypeScript %}
{% raw %}
type ListOfLength<Len extends number, List extends unknown[] = []> =
    List extends { length: Len }
        ? List
        : ListOfLength<Len, [unknown, ...List]>;
{% endraw %}
{% endhighlight %}

In TypeScript each array has a field `length`, similarly the array type also has a field `length`.
Using a conditional type, we can check whether a provided generic type parameter extends a type with a `length` field and also check whether it has an exact length `Len`.
If our current array type stored in `List` does not have the required length yet, we add an `unknown` type to `List` and perform a recursion on `ListOfLength`.

{% highlight TypeScript %}
{% raw %}
type Length<List extends any[]> =
    List extends { length: infer Len extends number }
        ? Len
        : never;
{% endraw %}
{% endhighlight %}

This type uses another feature of conditional types called `infer`. 
We can perform a structural pattern match with the conditional type and extract the exact type of a component  using `infer`. Here we use that to get the length of the array type `List`.
If `List` does not have a field `length`, we return `never`.
This is the default type, if a type cannot be constructed. 
For us `never` indicates an error, since we never intend to construct `never`.

{% highlight TypeScript %}
{% raw %}
type Increment<Number extends number> = Length<[...ListOfLength<Number>, unknown]>;

type Decrement<Number extends number> =
    ListOfLength<Number> extends [unknown, ...ListOfLength<infer NewNumber>]
        ? NewNumber
        : never;
{% endraw %}
{% endhighlight %}

`Increment` takes as input a positive integer or zero as the generic type parameter `Number`. 
It then turns this into an array of the length of `Number`, appends another value to the array and extracts the length of the new array.
`Decrement` is a little bit trickier. An array of length `Number` can be pattern matched with an array consisting of a head element and a tail of length `Number - 1`. We infer `Number - 1` as `NewNumber`. If `Number` is smaller than 1, we get `never`.

With that we can define arbitrary indexes and go up and down these indexes. We also need to be able to compare them:

{% highlight TypeScript %}
{% raw %}
// Assume positive integers.
type Max<Number1 extends number, Number2 extends number> =
    ListOfLength<Number1> extends [...ListOfLength<Number2>, ...ListOfLength<infer _>]
        ? Number1
        : Number2;

// Assume positive integers.
type Greater<Num1 extends number, Num2 extends number> =
    ListOfLength<Num1> extends [...ListOfLength<Num2>, ...ListOfLength<infer Difference extends number>]
        ? Difference extends 0
            ? false
            : true
        : false;

typeEqual<Greater<4, 3>, true>();
typeEqual<Greater<3, 3>, false>();
typeEqual<Greater<0, 3>, false>();
{% endraw %}
{% endhighlight %}

Both of these types use the same trick as `Decrement`. An array of length `Number1` is greater or equal in length to an array of length `Number2` if it can be represented by an array of length `Number2` and zero or more additional elements. For `Greater`, we need to consider the special case of `0`, since equal is not the same as greater.

With that we have already almost all tricks that we will also use in the following sections.

Now someone boring might ask: "Why are there three more sections just to define numbers, 
when you already have numbers using array types?".
There are two reasons:

1. This is more fun.
2. Already tried that. It is called the Peano axioms and those damn us to recursion hell.

So let's ignore the critics and start with some good old booleans.

## Booleans

Boolean arithmetics are pretty straightforward. 
`true` and `false` are both literal types that extend the type `boolean`.
`true` also extends `true` and `false` extends `false`.
We have already seen how conditional types allow us to check whether a generic type parameter extends a literal. 
So implementing the different logical operations is easy-peasy:

{% highlight TypeScript %}
{% raw %}
type Not<B extends boolean> =
    B extends true
        ? false
        : true;

type And<B1 extends boolean, B2 extends boolean> =
    B1 extends true
        ? B2 extends true
            ? true
            : false
        : false;

type Xor<B1 extends boolean, B2 extends boolean> =
    B1 extends true
        ? B2 extends true
            ? true
            : false
        : B2 extends true
            ? false
            : true;

type Or<B1 extends boolean, B2 extends boolean> =
    B1 extends true
        ? true
        : B2 extends true
            ? true
            : false;

typeEqual<And<false, false>, false>();
typeEqual<And<true, false>, false>();
typeEqual<And<false, true>, false>();
typeEqual<And<true, true>, true>();

typeEqual<Xor<false, false>, true>();
typeEqual<Xor<true, false>, false>();
typeEqual<Xor<false, true>, false>();
typeEqual<Xor<true, true>, true>();

typeEqual<Or<false, false>, false>();
typeEqual<Or<true, false>, true>();
typeEqual<Or<false, true>, true>();
typeEqual<Or<true, true>, true>();
{% endraw %}
{% endhighlight %}

And now we engage in building a logical circuit to construct a full adder, 
which will allow us two add two bits together.
Wikipedia has a good description of how that works 
([https://en.wikipedia.org/wiki/Adder_(electronics)#Full_adder](https://en.wikipedia.org/wiki/Adder_(electronics)#Full_adder)).
So here is just the implementation:

{% highlight TypeScript %}
{% raw %}
type FullAdder<B1 extends boolean, B2 extends boolean, Carry extends boolean> =
    { sum: Xor<Carry, Xor<B1, B2>>, carry: Or<Or<And<B1, B2>, And<B2, Carry>>, And<B1, Carry>> };

typeEqual<FullAdder<false, false, false>, { sum: false, carry: false }>();
typeEqual<FullAdder<false, true, false>, { sum: true, carry: false }>();
typeEqual<FullAdder<true, false, false>, { sum: true, carry: false }>();
typeEqual<FullAdder<false, false, true>, { sum: true, carry: false }>();
typeEqual<FullAdder<false, true, true>, { sum: false, carry: true }>();
typeEqual<FullAdder<true, true, false>, { sum: false, carry: true }>();
typeEqual<FullAdder<true, true, true>, { sum: true, carry: true }>();
{% endraw %}
{% endhighlight %}

`FullAdder` is a type with two fields `carry` and `sum`, which contain the carry bit of the addition and the sum bit of the addition, respectively.

Now we will extend the humble boolean or bit to the mighty bit sequence.

## Bits

A bit sequence is an array type consisting of boolean types `boolean[]`.
Adding two bit sequences together involves recursion and our lovely `FullAdder`.
We make the simplifying assumption that the two bit sequences are of equal length:

{% highlight TypeScript %}
{% raw %}
type AddBits<Number1 extends boolean[], Number2 extends boolean[], Carry extends boolean = false, Acc extends boolean[] = []> =
    Number1 extends { length: infer NBits }
        ? Number2 extends { length: NBits }
            ? Number1 extends [...(infer Rem1 extends boolean[]), infer Bit1 extends boolean]
                ? Number2 extends [...(infer Rem2 extends boolean[]), infer Bit2 extends boolean]
                    ? FullAdder<Bit1, Bit2, Carry> extends {
                            sum: infer BitSum extends boolean,
                            carry: infer BitCarry extends boolean
                        }
                        ? AddBits<Rem1, Rem2, BitCarry, [BitSum, ...Acc]>
                        // Unreachable
                        : never
                    // Unreachable
                    : never
                : { sum: Acc, carry: Carry }
            : never
        : never;

typeEqual<AddBits<[true], [true]>, { sum: [false], carry: true }>();
typeEqual<AddBits<[true, false, true], [true, false, false]>, { sum: [false, false, true], carry: true }>();
{% endraw %}
{% endhighlight %}

`AddBits` simply performs long addition. 
Adding two (or three including the carry) is easy and recursion over an array is also easy. 
This kind of recursion over arrays will be extremely common, 
so understanding it now will be helpful in the near future.
By pattern matching the input array into a tail array and a head element, 
we make the array shorter with each recursion step.
Our base case occurs, when the array is empty. 
We can then return the accumulated results stored in the default parameters `Carry` and `Acc`.

Because this was so much fun, let's do it again. 
But this time without assuming the two sequences are of the same length:

{% highlight TypeScript %}
{% raw %}
// Assume Bits is shorter than TargetLength.
type PadLeft<Bits extends boolean[], TargetLength extends number> =
    Bits extends { length: TargetLength }
        ? Bits
        : PadLeft<[false, ...Bits], TargetLength>;

typeEqual<PadLeft<[true, false], 4>, [false, false, true, false]>();

type AddBitsVariableSize<Number1 extends boolean[], Number2 extends boolean[]> =
    Number1 extends { length: infer NBits1 extends number }
        ? Number2 extends { length: infer NBits2 extends number }
            ? AddBits<
                PadLeft<Number1, Max<NBits1, NBits2>>,
                PadLeft<Number2, Max<NBits1, NBits2>>
            > extends { sum: infer Sum extends boolean[], carry: infer Carry extends boolean }
                ? [Carry, ...Sum]
                : never
            : never
        : never;

typeEqual<AddBitsVariableSize<[true, true], [true]>, [true, false, false]>();
typeEqual<AddBitsVariableSize<[false, true], [true]>, [false, true, false]>();
{% endraw %}
{% endhighlight %}

The trick to adding two bit sequences of different lengths is to make them have the same length. 
We achieve this with `PadLeft`, which prepends `False` until the shorter array is as long as the longer array.

We got addition. What is the cooler version of addition? Multiplication, obviously.
Let's go ahead and implement it. But how?
Well, my memory is not very good, but I still remember long multiplication from school.
Basically, take two multipliers called A and B. 
Multiply the first bit  of B with the entirety of A and add that to our result, which starts at 0.
Then multiply the second bit of B with A and shift the result by 1 to the left and add that to the result.

Repeat this for each bit and shift the product one bit further to the left each time.

We already have the ability to add two bit sequences.
To fully implement multiplication, we will also need to multiply a bit sequence with a single bit
and a way to shift a bit sequence to the left by `n` bits.

We will also introduce some helpers for later that can truncate our bit sequences. This will be useful,
once we want our bit sequences to have a fixed-size. Here, it will help us in writing
our `typeEqual` statements.

{% highlight TypeScript %}
{% raw %}
type MultiplyBitsWithBit<Left extends boolean[], Right extends boolean, Product extends boolean[] = []> =
    Left extends [infer Bit extends boolean, ...(infer Remainder extends boolean[])]
        ? MultiplyBitsWithBit<Remainder, Right, [...Product, And<Bit, Right>]>
        : Product;

typeEqual<MultiplyBitsWithBit<[true, false, false], true>, [true, false, false]>();
typeEqual<MultiplyBitsWithBit<[true, false, false], false>, [false, false, false]>();

type ShiftLeft<Bits extends boolean[]> = [...Bits, false];

typeEqual<ShiftLeft<[true, false, true]>, [true, false, true, false]>();

type ShiftLeftNTimes<Bits extends boolean[], N extends number> =
    N extends 0
        ? Bits
        : ShiftLeftNTimes<ShiftLeft<Bits>, Decrement<N>>;

typeEqual<ShiftLeftNTimes<[true, false], 3>, [true, false, false, false, false]>();

type TruncateFalseLeft<Bits extends boolean[]> =
    Bits extends [false]
        ? Bits
        : Bits extends [false, ...(infer RemainingBits extends boolean[])]
            ? TruncateFalseLeft<RemainingBits>
            : Bits;

typeEqual<TruncateFalseLeft<[false, false, true, false]>, [true, false]>();
typeEqual<TruncateFalseLeft<[true, false, true, false]>, [true, false, true, false]>();

type TruncateLeft<Bits extends boolean[], TargetLength extends number> =
    Bits extends { length: infer Length extends number }
        ? Greater<Length, TargetLength> extends true
            ? Bits extends [boolean, ...(infer Remainder extends boolean[])]
                ? TruncateLeft<Remainder, TargetLength>
                : never
            : Bits
        : never;

typeEqual<TruncateLeft<[false, false, true, false], 1>, [false]>();
typeEqual<TruncateLeft<[true, false, true, false], 3>, [false, true, false]>();

// Assume bits represent unsigned integers. Algorithm cannot overflow.
type MultiplyBits<Number1 extends boolean[], Number2 extends boolean[], Product extends boolean[] = [false], CurrentIndex extends number = 0> =
    Number2 extends [...(infer Remainder extends boolean[]), infer Bit extends boolean]
        // The extends helps with "infinitely deep" recursion.
        ? ShiftLeftNTimes<MultiplyBitsWithBit<Number1, Bit>, CurrentIndex> extends (infer Summand extends boolean[])
            ? MultiplyBits<Number1, Remainder, AddBitsVariableSize<Product, Summand>, Increment<CurrentIndex>>
            : never
        : Product

typeEqual<TruncateFalseLeft<MultiplyBits<[false, true, true], [false, true, false]>>, [true, true, false]>();
typeEqual<TruncateFalseLeft<MultiplyBits<[false, false, true, true], [false, false, true, false]>>, [true, true, false]>();
typeEqual<TruncateFalseLeft<MultiplyBits<[false, false, true, true], [false, false, true, true]>>, [true, false, false, true]>();
typeEqual<TruncateFalseLeft<MultiplyBits<[false, false, false, true, true, false, false, false], [false, false, false, true, true, false, false, false]>>, [true, false, false, true, false, false, false, false, false, false]>();
typeEqual<TruncateFalseLeft<MultiplyBits<[false, false, false, false, false, false, false, false], [false, false, false, true, true, false, false, false]>>, [false]>();
{% endraw %}
{% endhighlight %}

We will also want to compare numbers by magnitude, so lets quickly implement a greater type for bit sequences:

{% highlight TypeScript %}
{% raw %}
type GreaterBits<Number1 extends boolean[], Number2 extends boolean[]> =
    Number1 extends { length: infer Len }
        ? Number2 extends { length: Len }
            ? Len extends 0
                ? false
                : Number1 extends [infer Bit1 extends boolean, ...infer Rem1 extends boolean[]]
                    ? Number2 extends [infer Bit2 extends boolean, ...infer Rem2 extends boolean[]]
                        ? Xor<Bit1, Bit2> extends true
                            ? GreaterBits<Rem1, Rem2>
                            : Bit1
                        : never
                    : never
            : never
        : never;

typeEqual<GreaterBits<[], []>, false>();
typeEqual<GreaterBits<[false, false], [false, false]>, false>();
typeEqual<GreaterBits<[true, false, false], [false, false]>, never>();
typeEqual<GreaterBits<[true, false, false], [false, true, false]>, true>();
typeEqual<GreaterBits<[true, true, false], [true, true, true]>, false>();
typeEqual<GreaterBits<[true, true, true], [true, true, false]>, true>();
{% endraw %}
{% endhighlight %}

We got bit sequences and can operate on the positive numbers including zero. Negative numbers would be pretty cool, too.
So let's go ahead and implement integers next.

## Int8

The classic way to implement signed integers as opposed to unsigned integers is the two's complement 
([https://en.wikipedia.org/wiki/Two%27s_complement](https://en.wikipedia.org/wiki/Two%27s_complement)).
Unfortunately, the two's complement expects us to make a sacrifice. 
Our bit sequence needs to have a fixed length.
Consequently, we define our new type `Int8` as a bit sequence type with a fixed length of `8`.
We could do bigger numbers like `32`, `64` or `999`, 
but there is a balance between precision and TypeScript's ability to crawl through recursive types.

{% highlight TypeScript %}
{% raw %}
type BinaryNumberType<Size extends number, Result extends boolean[] = []> =
    Result extends { length: Size }
        ? Result
        : BinaryNumberType<Size, [boolean, ...Result]>;

typeEqual<BinaryNumberType<3>, [boolean, boolean, boolean]>();

type Int8 = BinaryNumberType<8>;
{% endraw %}
{% endhighlight %}

What makes `Int8` so special is its ability to represent negative integers using the two's complement.
So let's get started with that.
Making a positive number negative with two's complement just involves two things: 
Invert the bits and add 1. 
Adding is free using `AddBits`, but we need to implement the inversion.
We can then recognize a negative number by the first bit from the left. 
If it is 1, we got us a negative integer otherwise a positive one.

{% highlight TypeScript %}
{% raw %}
type InvertBits<Number extends boolean[], Result extends boolean[] = []> =
    Number extends [infer Bit extends boolean, ...(infer Remainder extends boolean[])]
        ? Bit extends true
            ? InvertBits<Remainder, [...Result, false]>
            : Bit extends false
                ? InvertBits<Remainder, [...Result, true]>
                : never
        : Result;

typeEqual<InvertBits<[true, false]>, [false, true]>();

// Assume B is a byte with a value between 1 and 128.
type TwosComplement<B extends Int8> =
    AddBits<InvertBits<B>, [false, false, false, false, false, false, false, true]>['sum'];
{% endraw %}
{% endhighlight %}

It would also be handy if we could convert these integers into numbers and the other way around.
That might make testing our types easier.

{% highlight TypeScript %}
{% raw %}
type BitsToInt8<Num extends boolean[]> =
    Num extends { length: 8 }
        ? Num
        : Greater<8, Length<Num>> extends true
            ? PadLeft<Num, 8>
            : TruncateLeft<Num, 8>;

typeEqual<BitsToInt8<[false, true, true, true, true, true, true, true]>, [false, true, true, true, true, true, true, true]>();
typeEqual<BitsToInt8<[false, true, false, true, true, true, true, true, true, true]>, [false, true, true, true, true, true, true, true]>();
typeEqual<BitsToInt8<[true, true, true, true, true]>, [false, false, false, true, true, true, true, true]>();
typeEqual<BitsToInt8<[false]>, [false, false, false, false, false, false, false, false]>();

type IsInteger<Num extends number> =
    `${Num}` extends `${number}.${number}`
        ? false
        : `${Num}` extends `${number}e-${number}`
            ? false
            : true;

type PositiveDigit = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
type Digit = 0 | PositiveDigit;
type OneDigitSignedIntDecimal = `${Digit}` | `-${PositiveDigit}`;
type TwoDigitSignedIntDecimal = `${PositiveDigit}${Digit}` | `-${PositiveDigit}${Digit}`;
type ThreeDigitInt8Decimal =
    `1${1 | 2}${0 | 1 | 2 | 3 | 4 | 5 | 6 | 7}`
    | `-1${1 | 2}${0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8}`;
type Int8Decimal = OneDigitSignedIntDecimal | TwoDigitSignedIntDecimal | ThreeDigitInt8Decimal;

type CanBeRepresentedAsByte<Number extends number> =
    `${Number}` extends Int8Decimal
        ? true
        : false;

typeEqual<CanBeRepresentedAsByte<5>, true>();
typeEqual<CanBeRepresentedAsByte<-6>, true>();
typeEqual<CanBeRepresentedAsByte<-35>, true>();
typeEqual<CanBeRepresentedAsByte<42>, true>();
typeEqual<CanBeRepresentedAsByte<127>, true>();
typeEqual<CanBeRepresentedAsByte<-128>, true>();
typeEqual<CanBeRepresentedAsByte<128>, false>();
typeEqual<CanBeRepresentedAsByte<-129>, false>();

type Int8PositionIndex = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8; // 8 represents "0" (kind of).
type Int8PositionValue = [128, 64, 32, 16, 8, 4, 2, 1];

type IncrementIndex<Index extends Int8PositionIndex> = Increment<Index> & Int8PositionIndex;

type PositiveIntegerToInt8<Integer extends number, Result extends boolean[] = [], Remainder extends unknown[] = ListOfLength<Integer>, StepIdx extends Int8PositionIndex = 0> =
    StepIdx extends 8
        ? Result
        : Remainder extends [...ListOfLength<Int8PositionValue[StepIdx]>, ...ListOfLength<infer NewRemainderValue>]
            // Able to subtract something.
            ? PositiveIntegerToInt8<Integer, [...Result, true], ListOfLength<NewRemainderValue>, IncrementIndex<StepIdx>>
            // Not able to subtract something.
            : PositiveIntegerToInt8<Integer, [...Result, false], Remainder, IncrementIndex<StepIdx>>;

// Assume Integer is an integer between -128 and 127.
type IntegerToInt8<Integer extends number> =
    `${Integer}` extends `-${infer PositiveInteger extends number}`
        ? TwosComplement<PositiveIntegerToInt8<PositiveInteger>>
        : PositiveIntegerToInt8<Integer>;

typeEqual<IntegerToInt8<127>, [false, true, true, true, true, true, true, true]>();
typeEqual<IntegerToInt8<7>, [false, false, false, false, false, true, true, true]>();
typeEqual<IntegerToInt8<-1>, [true, true, true, true, true, true, true, true]>();
typeEqual<IntegerToInt8<-128>, [true, false, false, false, false, false, false, false]>();
typeEqual<IntegerToInt8<0>, [false, false, false, false, false, false, false, false]>();
typeEqual<IntegerToInt8<-0>, [false, false, false, false, false, false, false, false]>();

type NumberToInt8<Number extends number> =
    IsInteger<Number> extends false
        ? never
        : CanBeRepresentedAsByte<Number> extends false
            ? never
            : IntegerToInt8<Number>;

typeEqual<NumberToInt8<127>, [false, true, true, true, true, true, true, true]>();
typeEqual<NumberToInt8<128>, never>();
typeEqual<NumberToInt8<-128>, [true, false, false, false, false, false, false, false]>();
typeEqual<NumberToInt8<-129>, never>();
{% endraw %}
{% endhighlight %}

We take a number and first make sure it can be represented by `Int8` using `IsInteger` and `CanBeRepresentedAsByte`.
Then we can check whether the number starts with a minus sign using a template literal type and extract
the actual positive number from it (`IntegerToInt8`).
We then convert the positive integer into an `Int8` using `PositiveIntegerToInt8`.
If the original number was negative, we have to convert it into a negative `Int8` using
the `TwosComplement` afterward.
The actual mechanism of turning a number into a bit sequence in `PositiveIntegerToInt8` basically
just tries to subtract smaller and smaller powers of 2 from the input number until nothing is left.
For each matching power of 2 a true is added to the bit sequence, for each failed matched a false is added.

Now that we can construct positive and negative integers and represent them without having to type out all the bits each time,
we can move on to the good part: arithmetics. And because we were so good about it before with the bit sequences,
this is getting really easy:

{% highlight TypeScript %}
{% raw %}
type Int8IsNegative<Num extends Int8> =
    Num[0] extends true
        ? true
        : false;

type AbsoluteInt8<Num extends Int8> =
    Int8IsNegative<Num> extends true
        ? TwosComplement<Num>
        : Num;

typeEqual<AbsoluteInt8<NumberToInt8<0>>, NumberToInt8<0>>();
typeEqual<AbsoluteInt8<NumberToInt8<-2>>, NumberToInt8<2>>();
typeEqual<AbsoluteInt8<NumberToInt8<2>>, NumberToInt8<2>>();

// Lots of additional extends to avoid "infinite depth" recursion errors.
// We have to be more explicit to help the type inference.
type MultiplyInt8<Num1 extends Int8, Num2 extends Int8> =
    MultiplyBits<AbsoluteInt8<Num1>, AbsoluteInt8<Num2>> extends (infer Product extends boolean[])
        ? BitsToInt8<Product> extends (infer ProductInt8 extends Int8)
            // If both are negative or positive, then the product is positive, otherwise it is negative.
            ? Xor<Int8IsNegative<Num1>, Int8IsNegative<Num2>> extends true
                ? ProductInt8
                : TwosComplement<ProductInt8>
            : never
        : never;

typeEqual<MultiplyInt8<NumberToInt8<5>, NumberToInt8<4>>, NumberToInt8<20>>();
typeEqual<MultiplyInt8<NumberToInt8<5>, NumberToInt8<-4>>, NumberToInt8<-20>>();
typeEqual<MultiplyInt8<NumberToInt8<-5>, NumberToInt8<-4>>, NumberToInt8<20>>();
typeEqual<MultiplyInt8<NumberToInt8<-5>, NumberToInt8<4>>, NumberToInt8<-20>>();
typeEqual<MultiplyInt8<NumberToInt8<-5>, NumberToInt8<0>>, NumberToInt8<0>>();
typeEqual<MultiplyInt8<NumberToInt8<-5>, NumberToInt8<1>>, NumberToInt8<-5>>();

type GreaterInt8<Num1 extends Int8, Num2 extends Int8> =
    Num1 extends [true, ...boolean[]]
        // Num1 negative
        ? Num2 extends [true, ...boolean[]]
            // Num1 negative, Num2 negative
            ? Not<GreaterBits<TwosComplement<Num1>, TwosComplement<Num2>>>
            // Num1 negative, Num2 positive
            : false
        : Num2 extends [true, ...boolean[]]
            // Num1 positive, Num2 negative
            ? true
            // Num1 positive, Num2 positive
            : GreaterBits<Num1, Num2>;

typeEqual<GreaterInt8<NumberToInt8<8>, NumberToInt8<4>>, true>();
typeEqual<GreaterInt8<NumberToInt8<8>, NumberToInt8<-4>>, true>();
typeEqual<GreaterInt8<NumberToInt8<-8>, NumberToInt8<4>>, false>();
typeEqual<GreaterInt8<NumberToInt8<-8>, NumberToInt8<-4>>, false>();
typeEqual<GreaterInt8<NumberToInt8<2>, NumberToInt8<2>>, false>();
{% endraw %}
{% endhighlight %}

Nice, this almost feels like an actual programming language and not a type system.

For neural networks, we need real numbers. We got integers and implementing genuine floating point numbers is not my vibe.
So let's do the next best thing and turn towards fixed-width decimals.

## Q4.3

Fixed-width decimals are a perfectly good and accurate representation of the real numbers, don't let any haters tell you otherwise.
We will implement the very cool decimal representation called Q4.3 ([https://en.wikipedia.org/wiki/Q_(number_format)](https://en.wikipedia.org/wiki/Q_(number_format))).
This fixed-width decimal representation requires 8 bits, which we luckily have with our `Int8` type.
The first bit is a sign bit, the next three bit stand for the integer component of the number and the last 4 bits stand for
the fractional component of the number. This gives us an amazingly large range from `-16` to `15 + 7/8`.
The cool thing is the two's complement works for Q4.3 in the same way. So one less thing to worry about.

Defining this number is really easy. The hard (more like fiddly really) part will be turning TypeScript numbers into the corresponding bit sequence.
Both of these things are coming up:

{% highlight TypeScript %}
{% raw %}
type FixedWidthDecimal = Int8;

type FractionalComponentToBinary<Num extends number>
    = `${Num}` extends '125'
    ? [false, false, true]
    : `${Num}` extends '25'
        ? [false, true, false]
        : `${Num}` extends '375'
            ? [false, true, true]
            : `${Num}` extends '5'
                ? [true, false, false]
                : `${Num}` extends '625'
                    ? [true, false, true]
                    : `${Num}` extends '75'
                        ? [true, true, false]
                        : `${Num}` extends '875'
                            ? [true, true, true]
                            : never;

type IntegerComponentToBinary<Num extends number> =
    TruncateFalseLeft<PositiveIntegerToInt8<Num>> extends (infer Bits extends boolean[])
        ? Bits
        : never;

// The type system needs a lot of help to do any kind of work here.
type NumberToFixedWidthDecimal<Number extends number> =
    `${Number}` extends `-${infer PositiveIntegerComponent extends number}.${infer FractionalComponent extends number}`
        // Negative decimal
        ? IntegerComponentToBinary<PositiveIntegerComponent> extends (infer IntegerBits extends boolean[])
            ? FractionalComponentToBinary<FractionalComponent> extends (infer FractionalBits extends boolean[])
                ? BitsToInt8<[...IntegerBits, ...FractionalBits]> extends (infer PositiveBits extends Int8)
                    ? TwosComplement<PositiveBits>
                    : never
                : never
            : never
        // Positive decimal
        : `${Number}` extends `${infer IntegerComponent extends number}.${infer FractionalComponent extends number}`
            ? IntegerComponentToBinary<IntegerComponent> extends (infer IntegerBits extends boolean[])
                ? FractionalComponentToBinary<FractionalComponent> extends (infer FractionalBits extends boolean[])
                    ? BitsToInt8<[...IntegerBits, ...FractionalBits]>
                    : never
                : never
            // Negative integer
            : `${Number}` extends `-${infer PositiveIntegerComponent extends number}`
                ? IntegerComponentToBinary<PositiveIntegerComponent> extends (infer IntegerBits extends boolean[])
                    ? BitsToInt8<[...IntegerBits, false, false, false]> extends (infer PositiveBits extends Int8)
                        ? TwosComplement<PositiveBits>
                        : never
                    : never
                // Positive integer
                : `${Number}` extends `${infer IntegerComponent extends number}`
                    ? IntegerComponentToBinary<IntegerComponent> extends (infer IntegerBits extends boolean[])
                        ? BitsToInt8<[...IntegerBits, false, false, false]>
                        : never
                    : never;

typeEqual<NumberToFixedWidthDecimal<1.5>, [false, false, false, false, true, true, false, false]>();
typeEqual<NumberToFixedWidthDecimal<1>, [false, false, false, false, true, false, false, false]>();
typeEqual<NumberToFixedWidthDecimal<-1>, [true, true, true, true, true, false, false, false]>();
typeEqual<NumberToFixedWidthDecimal<-1.5>, [true, true, true, true, false, true, false, false]>();
typeEqual<NumberToFixedWidthDecimal<0.5>, [false, false, false, false, false, true, false, false]>();
typeEqual<NumberToFixedWidthDecimal<3>, [false, false, false, true, true, false, false, false]>();
typeEqual<NumberToFixedWidthDecimal<9>, [false, true, false, false, true, false, false, false]>();
typeEqual<NumberToFixedWidthDecimal<-9>, [true, false, true, true, true, false, false, false]>();
typeEqual<NumberToFixedWidthDecimal<-0.5>, [true, true, true, true, true, true, false, false]>();
{% endraw %}
{% endhighlight %}

The fractional component has only 3 bits, accordingly there are only 7 distinct decimals (excluding zero) we can represent with Q4.3. 
This mapping is defined by `FractionalComponentToBinary`. The case for `0` does not have to represented since stringify-ing
numbers like `1.0` in TypeScript results in `'1'`. Converting the integer component is already a solved problem with our
type `PositiveIntegerToInt8`, which we use in `IntegerComponentToBinary`. These two mappings are combined in `NumberToFixedWidthDecimal`,
which essentially uses template literal types to check whether a stringify-ed number starts with a `-` and whether it has a decimal point.
Numbers that cannot be represented within Q4.3 are mapped to `never`. The alternative would have been some more complicated rounding logic,
which is really not needed for our purposes.

Same as before, we got our numbers now, so let's make them do some arithmetics. 
We only really have to watch out for the multiplication, where we have to put the fixed-decimal point back into its
place after multiplication:

{% highlight TypeScript %}
{% raw %}
// Addition and subtraction is the same as for Int8.
type AddFixedWidthDecimal<Num1 extends FixedWidthDecimal, Num2 extends FixedWidthDecimal> = AddInt8<Num1, Num2>;
type SubtractFixedWidthDecimal<Num1 extends FixedWidthDecimal, Num2 extends FixedWidthDecimal> = SubtractInt8<Num1, Num2>;

typeEqual<AddFixedWidthDecimal<NumberToFixedWidthDecimal<2>, NumberToFixedWidthDecimal<1>>, NumberToFixedWidthDecimal<3>>();
typeEqual<AddFixedWidthDecimal<NumberToFixedWidthDecimal<-2>, NumberToFixedWidthDecimal<1>>, NumberToFixedWidthDecimal<-1>>();
typeEqual<AddFixedWidthDecimal<NumberToFixedWidthDecimal<-2.5>, NumberToFixedWidthDecimal<1>>, NumberToFixedWidthDecimal<-1.5>>();
typeEqual<AddFixedWidthDecimal<NumberToFixedWidthDecimal<-2>, NumberToFixedWidthDecimal<1.5>>, NumberToFixedWidthDecimal<-0.5>>();

typeEqual<SubtractFixedWidthDecimal<NumberToFixedWidthDecimal<2>, NumberToFixedWidthDecimal<1>>, NumberToFixedWidthDecimal<1>>();
typeEqual<SubtractFixedWidthDecimal<NumberToFixedWidthDecimal<-2>, NumberToFixedWidthDecimal<1>>, NumberToFixedWidthDecimal<-3>>();
typeEqual<SubtractFixedWidthDecimal<NumberToFixedWidthDecimal<-2.5>, NumberToFixedWidthDecimal<1>>, NumberToFixedWidthDecimal<-3.5>>();

type MultiplyFixedWidthDecimal<Num1 extends FixedWidthDecimal, Num2 extends FixedWidthDecimal> =
    MultiplyBits<AbsoluteInt8<Num1>, AbsoluteInt8<Num2>> extends (infer Product extends boolean[])
        ? Product extends [...(infer Remainder extends boolean[]), boolean, boolean, boolean]
            ? BitsToInt8<Remainder> extends (infer ProductInt8 extends Int8)
                ? Xor<Int8IsNegative<Num1>, Int8IsNegative<Num2>> extends true
                    ? ProductInt8
                    : TwosComplement<ProductInt8>
                : never
            : never
        : never;

typeEqual<MultiplyFixedWidthDecimal<NumberToFixedWidthDecimal<2>, NumberToFixedWidthDecimal<1>>, NumberToFixedWidthDecimal<2>>();
typeEqual<MultiplyFixedWidthDecimal<NumberToFixedWidthDecimal<1>, NumberToFixedWidthDecimal<10>>, NumberToFixedWidthDecimal<10>>();
typeEqual<MultiplyFixedWidthDecimal<NumberToFixedWidthDecimal<1.5>, NumberToFixedWidthDecimal<3>>, NumberToFixedWidthDecimal<4.5>>();
typeEqual<MultiplyFixedWidthDecimal<NumberToFixedWidthDecimal<-1.5>, NumberToFixedWidthDecimal<3>>, NumberToFixedWidthDecimal<-4.5>>();
typeEqual<MultiplyFixedWidthDecimal<NumberToFixedWidthDecimal<-3>, NumberToFixedWidthDecimal<3>>, NumberToFixedWidthDecimal<-9>>();
typeEqual<MultiplyFixedWidthDecimal<NumberToFixedWidthDecimal<-2>, NumberToFixedWidthDecimal<1.5>>, NumberToFixedWidthDecimal<-3>>();
typeEqual<MultiplyFixedWidthDecimal<NumberToFixedWidthDecimal<-2>, NumberToFixedWidthDecimal<0>>, NumberToFixedWidthDecimal<0>>();
typeEqual<MultiplyFixedWidthDecimal<NumberToFixedWidthDecimal<0>, NumberToFixedWidthDecimal<2>>, NumberToFixedWidthDecimal<0>>();
{% endraw %}
{% endhighlight %}

When we multiply two Q4.3 numbers, our fixed decimal travels to the left leaving a total of six bits to the right of the decimal point.
We don't want to represent these 3 additional bits of precision, so we just truncate them. We achieve that by pattern matching
on an array consisting of our resulting product `Remainder` and a tail of 3 booleans.
The product in `Remainder` might also be bigger than our 8 bits allow for. Therefore, we truncate it using 
`BitsToInt8`. This is a very violent way of handling overflow, but as long as the bits don't form a union, we will be fine.

As you can see, implementing the fixed-width decimals was not particularly difficult thanks to our functioning integer type.
Now let's proceed to the last puzzle piece, before we can assmble our neural network.

## Vectors

We got real numbers now! (Don't let anyone tell you that Q4.3 is not a perfect 100% accurate representation of real numbers.)

Neural networks just need one more thing: Vectors! (Some might say matrices or tensors are needed, too, but those are lies by mathematicians. 
We use recursion or loops for that.)

Let's define our `Vector` type and some ways to define specific vectors:

{% highlight TypeScript linenos %}
{% raw %}
type Vector = FixedWidthDecimal[];

type ArrayToVector<Array extends number[], Vec extends Vector = []> =
    Array extends [infer Num extends number, ...infer Rem extends number[]]
        ? ArrayToVector<Rem, [...Vec, NumberToFixedWidthDecimal<Num>]>
        : Vec;

typeEqual<ArrayToVector<[3, -2]>, [NumberToFixedWidthDecimal<3>, NumberToFixedWidthDecimal<-2>]>();
typeEqual<ArrayToVector<[3, -2]>, [NumberToFixedWidthDecimal<3>, NumberToFixedWidthDecimal<-2>]>();
typeEqual<ArrayToVector<[-2, 1.5]>, [NumberToFixedWidthDecimal<-2>, NumberToFixedWidthDecimal<1.5>]>();

// Assume Size is a positive integer or 0.
type ZeroVector<Size extends number, Vec extends Vector = []> =
    Vec extends { length: Size }
        ? Vec
        : ZeroVector<Size, [...Vec, NumberToFixedWidthDecimal<0>]>;

typeEqual<ZeroVector<3>, ArrayToVector<[0, 0, 0]>>();

// Assume Size is a positive integer greater 0. Assume index is in {0, ..., Size - 1}.
type UnitVector<Size extends number, Index extends number, Vec extends Vector = []> =
    Vec extends { length: (infer CurrentSize extends number) }
        ? CurrentSize extends Size
            ? Vec
            : CurrentSize extends Index
                ? UnitVector<Size, Index, [...Vec, NumberToFixedWidthDecimal<1>]>
                : UnitVector<Size, Index, [...Vec, NumberToFixedWidthDecimal<0>]>
        : never;

typeEqual<UnitVector<3, 1>, ArrayToVector<[0, 1, 0]>>();
typeEqual<UnitVector<5, 4>, ArrayToVector<[0, 0, 0, 0, 1]>>();
{% endraw %}
{% endhighlight %}

We got our `Vector` type and, same as before, we do arithmetics with them:

{% highlight TypeScript %}
{% raw %}
type ScalarVectorMultiply<Scalar extends FixedWidthDecimal, Vec extends Vector, Result extends Vector = []> =
    Vec extends [infer Element extends FixedWidthDecimal, ...infer Rem extends FixedWidthDecimal[]]
        ? ScalarVectorMultiply<Scalar, Rem, [...Result, MultiplyFixedWidthDecimal<Scalar, Element>]>
        : Result;

typeEqual<ScalarVectorMultiply<NumberToFixedWidthDecimal<2>, ArrayToVector<[1, 2.5, 3]>>, ArrayToVector<[2, 5, 6]>>();

type VectorAdd<Vec1 extends Vector, Vec2 extends Vector, Sum extends Vector = []> =
    Vec1 extends [infer Field1 extends FixedWidthDecimal, ...(infer Rem1 extends FixedWidthDecimal[])]
        ? Vec2 extends [infer Field2 extends FixedWidthDecimal, ...(infer Rem2 extends FixedWidthDecimal[])]
            ? VectorAdd<Rem1, Rem2, [...Sum, AddFixedWidthDecimal<Field1, Field2>]>
            : never
        : Sum;

typeEqual<VectorAdd<[], []>, []>();
typeEqual<VectorAdd<ArrayToVector<[5, -2]>, ArrayToVector<[-0.5, 1.5]>>, [NumberToFixedWidthDecimal<4.5>, NumberToFixedWidthDecimal<-0.5>]>();

type VectorPointwiseMultiply<Vec1 extends Vector, Vec2 extends Vector, Product extends Vector = []> =
    Vec1 extends [infer Field1 extends FixedWidthDecimal, ...(infer Rem1 extends FixedWidthDecimal[])]
        ? Vec2 extends [infer Field2 extends FixedWidthDecimal, ...(infer Rem2 extends FixedWidthDecimal[])]
            ? VectorPointwiseMultiply<Rem1, Rem2, [...Product, MultiplyFixedWidthDecimal<Field1, Field2>]>
            : never
        : Product;

typeEqual<VectorPointwiseMultiply<[], []>, []>();
typeEqual<VectorPointwiseMultiply<ArrayToVector<[3, -2]>, ArrayToVector<[1, 0]>>, ArrayToVector<[3, 0]>>();
typeEqual<VectorPointwiseMultiply<ArrayToVector<[5, -2]>, ArrayToVector<[-0.5, 1.5]>>, [NumberToFixedWidthDecimal<-2.5>, NumberToFixedWidthDecimal<-3>]>();

type VectorSum<Vec extends Vector, Sum extends FixedWidthDecimal = NumberToFixedWidthDecimal<0>> =
    Vec extends [infer Field extends FixedWidthDecimal, ...infer Rem extends Vector]
        ? VectorSum<Rem, AddFixedWidthDecimal<Field, Sum>>
        : Sum;

typeEqual<VectorSum<[]>, NumberToFixedWidthDecimal<0>>();
typeEqual<VectorSum<ArrayToVector<[3.5, -2.5, 1]>>, NumberToFixedWidthDecimal<2>>();

type InnerProduct<Vec1 extends Vector, Vec2 extends Vector> = VectorSum<VectorPointwiseMultiply<Vec1, Vec2>>;

typeEqual<InnerProduct<ArrayToVector<[3, -2]>, ArrayToVector<[1, 0]>>, NumberToFixedWidthDecimal<3>>();
typeEqual<InnerProduct<ArrayToVector<[3, -2]>, ArrayToVector<[1, -1]>>, NumberToFixedWidthDecimal<5>>();
{% endraw %}
{% endhighlight %}

Nothing special to see here. Using recursion to iterate over arrays is standard fare by now.

Addition and inner products are the core ingredients of a neural network. We have them, so let's proceed to the finale.

## Neural Network

Our journey of building a neural network begins by constructing an affine layer.
That is just a \[vector, ..., vector\]-vector multiplication plus some intercept term (for mathematicians: matrix = \[vector, ... vector\]).
Additionally, derivatives are really important to neural networks, something about gradient descent. Let's also implement those, while we are at it:

{% highlight TypeScript %}
{% raw %}
type NeuronWeights<InputSize extends number> = {
    weight: { length: InputSize } & Vector,
    bias: FixedWidthDecimal
}

type AffineNeuron<Input extends Vector, Weights extends NeuronWeights<number>> =
    AddFixedWidthDecimal<InnerProduct<Weights['weight'], Input>, Weights['bias']>;

type AffineNeuronDerivativeWeight<Input extends Vector> = Input;

type AffineNeuronDerivativeBias = NumberToFixedWidthDecimal<1>;

type AffineNeuronDerivative<Input extends Vector> = {
    weight: AffineNeuronDerivativeWeight<Input>,
    bias: AffineNeuronDerivativeBias
}

type ScalarAffineNeuronMultiply<Scalar extends FixedWidthDecimal, Neuron extends NeuronWeights<number>> = {
    weight: ScalarVectorMultiply<Scalar, Neuron['weight']>,
    bias: MultiplyFixedWidthDecimal<Scalar, Neuron['bias']>
}

type AffineLayer<Input extends Vector, Weights extends NeuronWeights<number>[], Output extends Vector = []> =
    Weights extends [infer W extends NeuronWeights<number>, ...infer Rem extends NeuronWeights<number>[]]
        ? AffineLayer<Input, Rem, [...Output, AffineNeuron<Input, W>]>
        : Output;
{% endraw %}
{% endhighlight %}

A neuron takes an input vector and generates a single scalar (`AffineNeuron`), its derivative is just its weight vector (`AffineNeuronDerivative`).
An affine layer is then just a bunch of such neurons arranged neatly in a row (`AffineLayer`).
We don't calculate the derivative of the layer because it would turn into this unspeakable mathematical object (for mathematicians: a matrix).

Neural networks need non-linearity in addition to linearity (life is about balance after all).
Most nonlinear activation functions are very unpleasant to implement requiring exponents and strange symbols.
Luckily, Rectified Linear Unit (ReLU) was invented by some smart beings (according to [Wikipedia](https://en.wikipedia.org/wiki/Rectifier_(neural_networks)#cite_note-10), first used by Alstin S. Householder in 1941) and that is just a maximum function:

{% highlight TypeScript %}
{% raw %}
type ReLU<Input extends Vector, Result extends Vector = []> =
    Input extends [infer Element extends FixedWidthDecimal, ...infer Rem extends Vector]
        ? GreaterInt8<Element, NumberToFixedWidthDecimal<0>> extends true
            ? ReLU<Rem, [...Result, Element]>
            : ReLU<Rem, [...Result, NumberToFixedWidthDecimal<0>]>
        : Result;

typeEqual<ReLU<ArrayToVector<[1.5, -0.5, 0]>>, ArrayToVector<[1.5, 0, 0]>>();

type ReLuDerivativeScalar<Input extends FixedWidthDecimal> =
    GreaterInt8<Input, NumberToInt8<0>> extends true
        ? NumberToFixedWidthDecimal<1>
        : NumberToFixedWidthDecimal<0>;
{% endraw %}
{% endhighlight %}

Another nice thing about ReLU is that the derivative is either 0 or 1.

To learn is to make mistakes... almost. You also need to know in which direction to change to make fewer mistakes.
Accordingly, we will need the derivative of an error function for our neural network.
The mean squared error is the easiest to implement that I am aware of and having a background in time-series regression, it is my favorite:

{% highlight TypeScript %}
{% raw %}
type SquaredErrorDerivative<Predicted extends FixedWidthDecimal, Output extends FixedWidthDecimal> =
    MultiplyFixedWidthDecimal<NumberToFixedWidthDecimal<-2>, SubtractFixedWidthDecimal<Output, Predicted>>;
{% endraw %}
{% endhighlight %}

Now, let's assemble a neural network. Building an entire auto-differentiation system is certainly possible. For brevity's sake, this is left as an exercise to the reader.
Instead, we will define a specific feedforward neural network architecture consisting of a single hidden layer and a single output dimension.
The output is linear. Accordingly,
we are dealing with a regression model. What luck that we also implemented the mean squared error loss before, 
which is well suited to this kind of prediction task. 
Let's implement both a type for the feedforward neural network and its forward pass:

{% highlight TypeScript %}
{% raw %}
type NeuralNetworkWeights<InputSize extends number, HiddenSize extends number> = {
    inputToHidden: NeuronWeights<InputSize>[] & { length: HiddenSize },
    hiddenToOutput: NeuronWeights<HiddenSize>
}

type ForwardPass<Input extends Vector, Weights extends NeuralNetworkWeights<number, number>> =
    AffineNeuron<ReLU<AffineLayer<Input, Weights['inputToHidden']>>, Weights['hiddenToOutput']>

type HiddenLayerDerivativeNeuron<Input extends Vector, ErrorGradient extends FixedWidthDecimal, OutputWeight extends FixedWidthDecimal, NeuronActivation extends FixedWidthDecimal> =
    ScalarAffineNeuronMultiply<MultiplyFixedWidthDecimal<MultiplyFixedWidthDecimal<ErrorGradient, OutputWeight>, ReLuDerivativeScalar<NeuronActivation>>, AffineNeuronDerivative<Input>>;

type InputToHiddenDerivative<Input extends Vector, ErrorGradient extends FixedWidthDecimal, OutputWeights extends NeuronWeights<number>['weight'], HiddenActivation extends Vector, Result extends NeuronWeights<number>[] = []> =
    OutputWeights extends [infer W extends FixedWidthDecimal, ...infer RemWeights extends NeuronWeights<number>['weight']]
        ? HiddenActivation extends [infer H extends FixedWidthDecimal, ...infer RemOutputs extends Vector]
            ? InputToHiddenDerivative<Input, ErrorGradient, RemWeights, RemOutputs, [...Result, HiddenLayerDerivativeNeuron<Input, ErrorGradient, W, H>]>
            : never
        : Result;

type HiddenToOutputDerivative<ErrorGradient extends FixedWidthDecimal, HiddenOutput extends Vector> =
    ScalarAffineNeuronMultiply<ErrorGradient, AffineNeuronDerivative<HiddenOutput>>;

type NeuralNetworkDerivative<Input extends Vector, PredictedOutput extends FixedWidthDecimal, TrueOutput extends FixedWidthDecimal, Weights extends NeuralNetworkWeights<number, number>> =
    SquaredErrorDerivative<PredictedOutput, TrueOutput> extends (infer ErrorGradient extends FixedWidthDecimal)
        ? AffineLayer<Input, Weights['inputToHidden']> extends (infer HiddenActivation extends  Vector)
            ? ReLU<HiddenActivation> extends (infer HiddenOutput extends Vector)
                ? {
                    inputToHidden: InputToHiddenDerivative<Input, ErrorGradient, Weights['hiddenToOutput']['weight'], HiddenActivation>,
                    hiddenToOutput: HiddenToOutputDerivative<ErrorGradient, HiddenOutput>
                }
                : never
            : never
        : never;
{% endraw %}
{% endhighlight %}

I basically hand-calculated the backpropagation graph of this neural network and then wrote it the gradients down here.
There is no magic left anymore. We used that all up when we conjured numbers from conditional types and some spit.

Now that we have the forward pass and its corresponding derivative, we can implement the backward pass:

{% highlight TypeScript %}
{% raw %}
type UpdateWeightsNeuron<Weights extends NeuronWeights<number>, Derivative extends NeuronWeights<number>, LearningRate extends FixedWidthDecimal> =
    {
        weight: VectorAdd<Weights['weight'], ScalarVectorMultiply<MultiplyFixedWidthDecimal<NumberToFixedWidthDecimal<-1>, LearningRate>, Derivative['weight']>>,
        bias: SubtractFixedWidthDecimal<Weights['bias'], MultiplyFixedWidthDecimal<LearningRate, Derivative['bias']>>
    }

typeEqual<
    UpdateWeightsNeuron<
        { weight: ArrayToVector<[1, -2.5]>, bias: NumberToFixedWidthDecimal<0.5> },
        { weight: ArrayToVector<[-0.5, 1]>, bias: NumberToFixedWidthDecimal<1> },
        NumberToFixedWidthDecimal<2>
    >,
    { weight: ArrayToVector<[2, -4.5]>, bias: NumberToFixedWidthDecimal<-1.5> }
>();

type UpdateWeightsLayer<Weights extends NeuronWeights<number>[], Derivatives extends NeuronWeights<number>[], LearningRate extends FixedWidthDecimal, Result extends NeuronWeights<number>[] = []> =
    Weights extends [infer W extends NeuronWeights<number>, ...infer RemWeights extends NeuronWeights<number>[]]
        ? Derivatives extends [infer D extends NeuronWeights<number>, ...infer RemDerivatives extends NeuronWeights<number>[]]
            ? UpdateWeightsLayer<RemWeights, RemDerivatives, LearningRate, [...Result, UpdateWeightsNeuron<W, D, LearningRate>]>
            : never
        : Result;

type BackwardPass<Input extends Vector, TrueOutput extends FixedWidthDecimal, Weights extends NeuralNetworkWeights<number, number>, LearningRate extends FixedWidthDecimal> =
    NeuralNetworkDerivative<Input, ForwardPass<Input, Weights>, TrueOutput, Weights> extends {
            inputToHidden: infer InputToHiddenDerivative extends NeuronWeights<number>[],
            hiddenToOutput: infer HiddenToOutputDerivative extends NeuronWeights<number>
        }
        ? {
            inputToHidden: UpdateWeightsLayer<Weights['inputToHidden'], InputToHiddenDerivative, LearningRate>,
            hiddenToOutput: UpdateWeightsNeuron<Weights['hiddenToOutput'], HiddenToOutputDerivative, LearningRate>
        }
        : never;
{% endraw %}
{% endhighlight %}

A backward pass (`BackwardPass`) updates the weights in each layer according to the error gradient and some learning rate (`UpdateWeightsLayer`).

We are so very close now and my brain is starting to dissolve and ooze out of my ears (the latter fact is not related to this blog post).

We just need to initialize a neural network with random weights, toss some training data at it, and then perform a prediction.

Since we have no access to side effects, generating good random numbers is hard and pseudo-random numbers are not actually random.
The correct approach then clearly is to throw some dice and use those for weight initialization. 
So this is what I did and here are the truly certifiably random initial weights:

{% highlight TypeScript %}
{% raw %}
type ExampleWeights = {
    inputToHidden: [
        { weight: ArrayToVector<[0.5, 1.0]>, bias: NumberToFixedWidthDecimal<1.0> },
        { weight: ArrayToVector<[-1.0, 2.0]>, bias: NumberToFixedWidthDecimal<0> },
        { weight: ArrayToVector<[0.5, 1]>, bias: NumberToFixedWidthDecimal<-0.5> }
    ],
    hiddenToOutput: { weight: ArrayToVector<[-0.5, 1.0, 0.25]>, bias: NumberToFixedWidthDecimal<0.5> }
}

typeEqual<ExampleWeights extends NeuralNetworkWeights<2, 3> ? true : false, true>();
typeEqual<ForwardPass<ArrayToVector<[1, -1]>, ExampleWeights>, NumberToFixedWidthDecimal<0.25>>();
{% endraw %}
{% endhighlight %}

As we can see before training the neural network produces an output of `0.25` for an input of `[1, -1]`.
As everyone knows the correct answer for that input is `4`. Our very human brains just instinctively know that.
So let's teach the neural network this universal truth of human existence by performing a single training iteration and then predicting again.

{% highlight TypeScript %}
{% raw %}
type TrainOnceAndPredict<Input extends Vector, TrueOutput extends FixedWidthDecimal, Weights extends NeuralNetworkWeights<number, number>, LearningRate extends FixedWidthDecimal> =
BackwardPass<Input, TrueOutput, Weights, LearningRate> extends (infer NewWeights extends NeuralNetworkWeights<number, number>)
? ForwardPass<Input, NewWeights>
: never;

typeEqual<TrainOnceAndPredict<ArrayToVector<[1, -1]>, NumberToFixedWidthDecimal<4>, ExampleWeights, NumberToFixedWidthDecimal<0.5>>, NumberToFixedWidthDecimal<4.25>>();
{% endraw %}
{% endhighlight %}

As we can see the new output after a single training iteration is `4.25`, which is very close to the truth. For me that is sufficient evidence that we built a functioning neural network.
And according to a lot of tech bro hype online, neural networks are capable of achieving artificial general intelligence (AGI). So clearly what we achieved here today is
the first step towards AGI in the type system and therefore artificial intelligence without a runtime.

Thank you for reading until this point or scrolling this far without reading :)

