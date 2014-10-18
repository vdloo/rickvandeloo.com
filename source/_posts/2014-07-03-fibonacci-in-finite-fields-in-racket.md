---
title: Implementing Fibonacci in finite fields, in racket
layout: post
---
A couple of days ago [Ruud van Asseldonk](https://github.com/ruud-v-a)
submitted a post about [Fibonacci numbers in finite
fields](http://ruudvanasseldonk.com/2014/07/01/fibonacci-numbers-in-finite-fields).
In the post it is first explained that [Binet's
formula](http://mathworld.wolfram.com/BinetsFibonacciNumberFormula.html)
holds up in a finite field, then an implementation of this method in c++
is presented. Since [I have started going through
SICP](http://rickvandeloo.com/2014/07/02/why-I-am-starting-the-SICP-reading-project-this-summer/),
I thought it would be a nice exercise to port this implementation to
scheme.

The author's implementation consists of five functions:

1.  addmod
2.  submod
3.  mulmod
4.  powmod
5.  fib

Porting the first two function is pretty straight forward.

The original addmod function
{% highlight cpp %}
    u64 addmod(u64 a, u64 b, u64 p)
    {
      if (p - b > a) return a + b;
      else return a + b - p;
    }
{% endhighlight %}

becomes:
{% highlight racket %}
    (define (addmod a b p)
      (if  (> (- p b) a)
          (+ a b)
          (- (+ a b) p)
      )
    )
{% endhighlight %}

and the original submod function

{% highlight cpp %}
    u64 submod(u64 a, u64 b, u64 p)
    {
      if (a >= b) return a - b;
      else return p - b + a;
    }
{% endhighlight %}

becomes:

{% highlight racket %}
    (define (submod a b p)
      (if (>= a b)
          (- a b)
          (+ (- p b) a)
      )
    )
{% endhighlight %}

The mulmod function is a bit more complicated:

{% highlight cpp %}
    u64 mulmod(u64 a, u64 b, u64 p)
    {
      u64 r = 0;

      while (b > 0)
      {
        if (b & 1) r = addmod(r, a, p);
        b >>= 1;
        a = addmod(a, a, p);
      }

      return r;
    }
{% endhighlight %}

The implementation in scheme displays more clearly the difference
between a functional programming language and the imperative c++. The
implementation in scheme is ostensibly inside out compared to the
original.

{% highlight racket %}
    (define (mulmod a b p)
      (mulmod_rec a b p 0)
    )
    (define (mulmod_rec a b p r)
      (if (> b 0)
          (mulmod_rec (addmod a a p)
                  (arithmetic-shift b -1)
                  p
                  (if (= 1 (bitwise-and b 1)) (addmod a r p) r)
          )
          r
      )
    )
{% endhighlight %}

Instead of a while loop, the scheme implementation of mulmod calls
itself. Scheme doesn't have looping constructs because of the way the
language was designed; there is no need for a special form to do this
because Scheme allows you to conveniently define iterative processes by
means of recursive procedures.

Porting this function was the moment where I decided to move over from
mit-scheme to plt-scheme, also known as Racket. In order to stay close
to the original I wanted to use the [bitwise
operations](http://en.wikipedia.org/wiki/Bitwise_operation) AND (&) and
the right arithmetic shift (\>\>=). Unfortunately to the best of my
knowledge mit-scheme doesn't offer built in bitwise operations on
anything but
[fixnums](http://web.mit.edu/scheme_v9.0.1/doc/mit-scheme-ref/Fixnum-Operations.html)
and [bit
strings](http://www.cse.iitb.ac.in/~as/mit-scheme/scheme_10.html#SEC103),
but luckily [racket
does](http://docs.racket-lang.org/reference/generic-numbers.html#%28part._.Bitwise_.Operations%29).

Next up is powmod:

{% highlight cpp %}
    u64 powmod(u64 a, u64 e, u64 p)
    {
      u64 r = 1;
      
      while (e > 0)
      {
        if (e & 1) r = mulmod(r, a, p);
        e >>= 1;
        a = mulmod(a, a, p);
      }

      return r;
    }
{% endhighlight %}

Which follows almost the exact same structure as mulmod:

{% highlight racket %}
    (define (powmod a e p)
      (powmod_rec a e p 1)
    )
    (define (powmod_rec a e p r)
      (if (> e 0)
          (powmod_rec (mulmod a a p)
                  (arithmetic-shift e -1)
                  p
                  (if (= 1 (bitwise-and e 1)) (mulmod r a p) r)
          )
          r
      )
    )
{% endhighlight %}

Note that instead of doing assigning 0 to r before the while loop in
mulmod we seed that 0 as a parameter every time mulmod is called to
mulmod\_rec. The same goes for powmod and powmod\_rec.

To tie it all together we have the fib function:

{% highlight cpp %}
    u64 fib(u64 n)
    {
      u64 p     = 0xa94fad42221f27ad;
      u64 v     = 0x0b92025517515f58;
      u64 v_inv = 0x242d231e3eb01b01;

      u64 a        = powmod(1 + v, n, p);
      u64 b        = powmod(p + 1 - v, n, p);
      u64 pow2_inv = powmod(2, p - 1 - n, p);

      u64 diff   = submod(a, b, p);
      u64 factor = mulmod(v_inv, pow2_inv, p);

      return mulmod(diff, factor, p);
    }
{% endhighlight %}

Which in Racket looks like:

{% highlight racket %}
    (define (fib n)
      (define (q)     12200160415121876909)
      (define (v)     833731445503647576)
      (define (v_inv) 2606778372125104897)
      (mulmod (submod (powmod (+ 1 (v)) n (q))
                      (powmod (- (+ (q) 1) (v)) n (q))
                      (q)
              )
              (mulmod (v_inv)
                      (powmod 2 (- (q) 1 n) (q))
                      (q)
              )
              (q)
      )
    )
{% endhighlight %}

So now we have all the components to generate a Fibonacci number. It
would of course be way more gratifying to have the program compute all
the numbers up to [the maximum of
N=93](http://ruudvanasseldonk.com/2014/07/01/fibonacci-numbers-in-finite-fields#finite-fields)
instead of simply computing one at a time. To do that we can use the
[when evaluator](http://docs.racket-lang.org/reference/when_unless.html)
in a recursive procedure. Full disclosure though: it would probably be
faster to use the regular iterative method if you are going to calculate
all the numbers anyway, but that doesn't take away the fun.

{% highlight racket %}
    (define (fib-iter n count result)
      (display "Computing fib for ")(display count)(display ": ")
      (display result)(newline)
      (when (< count n)
          (fib-iter n
                    (+ count 1)
                    (fib count)
          )
       )
    )
    (define (multifib n)
      (fib-iter n 1 0)
    )
    (multifib 93)
{% endhighlight %}

And that's all there is to it: a racket fibonacci calculator that runs
in constant time by use of finite fields. Lots of praise to the original
author for this fascinating nugget of information and for figuring out
how to translate it into a working program.

[You can find the put together version of this racket script on GitHub
Gist](https://gist.github.com/vdloo/b3eed782c0b69617d718)
