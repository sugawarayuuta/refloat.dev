# Float parser that sacrifices nothing

## Introduction

It's been a while...

Hello.  My name is Sugawara Yuuta. it has been 5 or 6 months since the last time I wrote an article.

Many things happened since and I just graduated from highschool few days ago. I don't really have plans on what to do next, but I'm going to sharpen my skills even more.

So, it's a bit different from what I have written in the past but I want to introduce my parser that converts strings to floating point numbers.

I'm pretty bad at math, so I hope it improved my viewpoint of math.

## Why does it matter?

Who cares? Who doesn't. It's such an interesting topic to cover.

floats are used all over the place. Even though the idea behind it is simple, conversion between them and decimals are really complicated. For example, a parser might require arbitrary precision math, just to get ~16 digits.

More importantly, it's a slow operation, there aren't many algorithms that conver this functionality completely, even.

Today's goal is to make little bit less painful, faster, and simpler. You may ask, "Is it possible to have both of simplicity and speed at the same time?"

If I say the conclusion first, it is! I hope I'll cover your questions by the end of an article, and we're going to "rediscover" what I've found, hopefully.

## How floats work

> I'm going to explain how floating point numbers work. if you know how it works, or you have read IEEE-754 in the past, you may skip this section.

There are many ways you can represent numbers in a computer, the most common one, of course, is an integer. I think you have heard it by now, but in computers binary is used, and it counts up like below.

```
┌─┬─┬─┬─┐              
│ │ │ │ │      14=8+4+2
│ │ │ │ │      5 =4+1       
└─┴─┴─┴─┘      7 =4+2+1
 ▲ ▲ ▲ ▲       ...     
 │ │ │ └─────1                    
 │ │ └─────2                             
 │ └─────4                              
 └─────8               
```

But there is a problem! We can't represent fractions with it. in real life, you use 0.1, or 0.25, etc.

Let's say you were given 8 bits, what would you do to implement this?

One easy way to do this, is to put a decimal point at the middle. How you count, basically doesn't change. Before the decimal point, you count it like 1, 2, 4, 8 like usual, and after that you do an inverse. (1/2, 1/4, 1/8...)

Wait, but that greatly reduces the maximum number we can represent!

OK, let's write a program to see how it was reduced.

```go:main.go
// https://go.dev/play/p/12BNqmE2Wi4
package main

import "fmt"

func main() {
	printFrac(^uint8(0))
}

func printFrac(u8 uint8) {
	fmt.Print(u8>>4, ".")
	u8 &= 0xf
	for u8 != 0 {
		u8 *= 10
		fmt.Print(u8 >> 4)
		u8 &= 0xf
	}
	fmt.Println()
}
```
`^` is NOT, it just makes ones zero and zeros one. That means, `^uint8(0)` is just a max number we have.

Normally, the max value for `uint8` is 255, so we can say for certain that 15.9375 is incredibly small compared to that.

So the question becomes, "can we represent wide range of numbers while allowing fractions? If so, how?"... to answer that question, we need to see the standard people came up with, called IEEE-754.

### IEEE-754

Do you know about scientific notation? It's the name of a representation of numbers like `6.02 * 10^23`. The standard that's based on it is called IEEE-754.

![](https://storage.googleapis.com/zenn-user-upload/7f7956446efe-20240306.png)
*By Fresheneesz at the English Wikipedia project, CC BY-SA 3.0, https://commons.wikimedia.org/w/index.php?curid=3357169*

The leftmost bit is the sign bit, it is set to 1 if it's negative and 0 otherwise. Something to care about is that even if the mantissa is 0, there are still two patterns of signs. (0, -0)

though most signed integer implementations are on two's complement, which doesn't have this issue.

Next, the middle part is the exponent. That's 10^23 in the example I just showed. of course, it's 2^x when it comes to computers.

The last part is the mantissa. When you normalize the integer part, it should be non-zero 1 digit. By the way, the people realized that in binary, non-zero 1 digit can only be 1, so we actually don't store this bit if we follow the standard.

## Problem statement

So far we have talked about binary system, but we all use decimal in the real life.

That means, we need to convert between them someway. More specifically,

- convert from 10^x to 2^e

That's what we need. In an equation,

- 2^e = 10^x

Let's take logarithm from both sides...

- log_2(2^e) = log_2(10^x)

Since log_base(base^e) = e,

- e = log_2(10^x)

But we still need to evaluate 10^x, which might be expensive. 

However, using log_base_0(base_1^e) = e * log_base_0(base_1) gives

- e = x * log_2(10)

Yay! Oh, one thing to note, we can only use integers in exponents. so...

- m * 2^floor(e) = 2^e

OK, it's not that difficult. Using base^e_0 / base^e_1 being base^(e_0 - e_1) after rearranging brings 

- m = 2^(e - floor(e))

Now, the problem is a lot easier to work with.

- Calculate 2^x (0 <= x < 1)

### Attempt of 2^x: 1. "exponentiation by squaring"

The first thing I came up with is "exponentiation by squaring" algorithm. it's *usually* used for cases where x is an integer, but we can still use that by inverting the whole thing. 

How it works is not difficult to understand. If you haven't heard about it. try reading [wikipedia](https://en.wikipedia.org/wiki/Exponentiation_by_squaring) or something similar.

But because of how we're in a special case, we are going to need square roots. Probably the easiest way to do so is newton's method.

Newton's method, finds a root of a function f(x). The great thing about this method is that you get quadratic convergence! That means, the number of correct digits doubles in each iteration. The implementation goes like this.

- x_n+1 = x_n - f(x_n) / f'(x_n)

By the way, it's more complicated, but it's possible to get cubic convergence by using [Halley's method](https://en.wikipedia.org/wiki/Halley%27s_method).

In this case of x = sqrt(a), f(x) = x^2 - a becomes 0. The derivative of this function is f'(x) = 2x, and you'll be able to calculate a root using that. I don't remember exactly, but this method is slow. The cause is obviously the square root, as newton's method requires multiple iterations and getting high precision doesn't scale well (n^2). We can also use the hardware instruction if the precision required is lower.

### Attempt of 2^x: 2. "Taylor expansion"

The idea behind Taylor expansion is to make non-polynomial functions polynomials up to certain x value, centered at some value (usually 0). Polynomials are much easier to work with.

Taylor expansion is defined as below.

- sum(f^n(a)/n! * (x-a)^n) (1 <= n <= inf)

f^n(x) is nth derivative of f(x). differentiating 2^x gives 2^x * log(2)^n,

- 2^x ~= 1 + x\*log(2) + x^2\*log(2)^2/2! + x^3\*log(2)^3/3!

To reduce the cost of multiplications, we can use [Horner's method](https://en.wikipedia.org/wiki/Horner%27s_method). 

And using Taylors theorem gives us the maxium absolute error of the approximation, we can check how many terms we have to use to reach certain precision *for sure*.

By the way, doing argument reduction can make the convergence faster. the maximum input is 1 (r = 1), so applying the Lagrange remainder formula becomes

- R(r) = log(2)^k * r^k / (k+1)! * r^(k+1)

Which shows you need to use 18 multiplications to get 64bits. However there is a trick; iterating some top bits and multiplying 2^nth root of 2 for non-zero bit, and doing Taylor expantion for the remaining bits gets us to just 13 multiplications.

### Attempt of 2^x: 3. "Remez Algorithm"

Is it possible to reduce the number of terms further? Yes. There is an approach called minimax in approximation theory. Minimax constructs polynomials such that the maximum absolute error becomes the smallest. This field is actually well studied, So I used a tool called [sollya](https://www.sollya.org/) for that.

I'll construct it so that the maximum absolute error is around 2^-64. I used 10th order for 64bit and 5th order for 32bit.

Also, currently I'm rounding to the nearest exponent integer instead of flooring to always round down. (input, instead of 0 <= x < 1, is currently 0 <= x <= 1/2) Adding this to the Horner method I described ealier gives what I use for now.

## What I made while studying this

This method itself doesn't use lookup tables, at all, while the current `strconv` implementation uses a large table. In addition to that, it's able to calculate floats faster than the current `strconv` on most cases.

Also, the functions follow the standard library closely, so for `ParseFlost` you can migrating by just changing the import statement. It passes standard libraries' tests too. The project is published on GitHub which is under BSD license.

[https://github.com/sugawarayuuta/refloat](https://github.com/sugawarayuuta/refloat)

## By the way...

I did fuzzing while I was sleeping and it mismatched with the standard library after 4 hours. Actually, it was their unexpected behavior. but since the input is quite large (800+digits), It might also make sense to ignore. Their slow-path has been using 800 digit decimal for a while.

## Conclusion

I explained how my float parser worked. Ask me anything if it was unclear for you, feel free to comment, etc. contributions are welcomed, too. Thank you for reading. Bye.