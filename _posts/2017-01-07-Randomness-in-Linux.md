# Randomness in Linux

Majority of web pages and blog posts [(1)](#1) I've read suggests to use `/dev/urandom`.

I would say this generalization should never be made.

Doing random things on purpose is very difficult. Especially with electronics. Things
may appear fully random but often are not in fact. Still randomness is key part in
important practices like cryptography. Thus it is important to understand terms
**random** and **randomness**.

end of TL;DR so now go and read...

Sources of randomness
---

> *Randomness is the lack of pattern or predictability in events. -[Wikipedia](https://en.wikipedia.org/wiki/Randomness)*

There are three generally accepted sources of randomness:

* environment the system is working in
* initial conditions of the system
* things generated by the system itself

Randomness generated by the system itself is called **pseudo**-randomness. There is no
mathematical source of true randomness.

The power of random things is that those things are really unpredictable. In truly
random events no one is able to predict the nature of next event. These events can be
random in time, intensity, but often called as being [stochastic](https://en.wikipedia.org/wiki/Stochastic).

Randomness in computer systems
---

Some of you might know this meme from [xkcd](http://xkcd.com/221/):

```java
int getRandomNumber()
{
  return 4 // chosen by fair dice roll
           // guaranteed to be random
}
```

While its output being truly random above function is still quite unusable when
unpredictability is needed and in context of randomness unpredictability is always needed.

In computer system high enough unpredictability is often as useful as full
unpredictability. It is only a matter of energy (or value $$$) needed to predict
next event. As long as energy needed for predicting things is higher than the
value of the achievement of successful prediction we can say system is
unpredictable enough.

Also time should be taken into account because calculating power comes cheaper day by
day. Things that are too expensive to predict today can be cheap enough tomorrow.
Thus unpredictable system must stay unpredictable enough also in the future, not
forever but long enough in relation to value.

*This is pure analogy to information security as data has always a value and if
it is encrypted using algorithms that are relaying randomness (and most often are)
it is only a matter of time and energy when the random event behind this encryption
will be predicted and data to become public.*

Randomness in operating systems
---

Instead of starting to create your own pseudo-random generator nearly all operating
systems provide the source of pseudo-randomness.

Again, randomness in Linux
---

Linux kernel maintains an **[entropy](https://en.wikipedia.org/wiki/Entropy) pool**. From where data to this pool is obtained, you can start reading:

[/drivers/char/random.c](https://github.com/torvalds/linux/blob/master/drivers/char/random.c)

*There is that `push_to_pool()` function which is called from various locations in the kernel.
I'm not going into details on this but will say that at least mouse movements and keyboard presses
are used when pushing environment related true random bits into pool.*

To check how much bits are available in this pool:

```sh
# This read-only file gives the available entropy.
# Normally, this will be 4096 (bits), a full entropy pool.
$ cat /proc/sys/kernel/random/entropy_avail
859
```

Every time when the content of this entropy pool is used to derive pseudo-random data that content is
removed from the pool and more content needs to be pushed back with `push_to_pool()` to maintain decent
pool size.

External random sources
---

Using [different software utilities](https://wiki.archlinux.org/index.php/Rng-tools) Linux kernel can become
aware of [external hardware](https://en.wikipedia.org/wiki/Hardware_random_number_generator) random bit sources and use these as a source of the entropy pool itself. This
provides even better entropy in the pool and thus to create better (more unpredictable) pseudo-random data.

The case of /dev/random
---

Linux kernel provides you pseudo-random data derived from the content of this entropy pool. There is
file called `/dev/random` which is not a static file but a file provided by the kernel itself. Every
time you read from this file kernel will give you pseudo-random data derived from the content of its
entropy pool.

```sh
# Read first 32 bytes and present as base64 encoded
$ head -c 32 /dev/random | base64
UN8ENY95SVz2rBr0KmmhuP3yffm2qd8zqbX8MQSDnYQ=
```

There is one nasty behavior of `/dev/random`; every time entropy pool drains itself empty all read
operations to `/dev/random` will **block** (io-wait) until enough data is pushed and becoming available
from the entropy pool to derive more random data.

The quality (unpredictability) of pseudo-randomness is better when blocking until enough data is pushed
and becoming available from the entropy pool.

The case of /dev/urandom
---

When **lower quality** (more predictable) pseudo-random data can be used there is `/dev/urandom` available
for that purpose. Data is again derived from the content of this entropy pool but read operations to this
file will not block (io-wait) when entropy pool drains itself empty. Instead kernel will create
**lower quality** pseudo-random data to substitute better quality data derived from the content of its
entropy pool. This is achieved by re-using already used bits obtained from entropy pool.

Which one to use?
---

Majority of web pages and blog posts I've read suggests to use `/dev/urandom`. I would say this generalization should never be made.

When the best available quality is needed `/dev/random` should be used always even with the cost of blocking. [(2)](#2)
One example is when generating keys **to be used in cryptographic algorithms**.

When there is the absolute need to be non-blocking (in cryptography) even with the cost of lower quality randomness `/dev/urandom` can be used. But still be very careful when making the decision.

For whatever else from temporary filename creation to generating "almost unique" data blobs you can use `/dev/urandom` as a source but note that you should neither waste the valuable bits inside Linux's entropy pool for tasks that doesn't require randomness but only uniqueness.

*Some versions of man page [random(4)](https://linux.die.net/man/4/random) states usage limitations while [some other](http://man7.org/linux/man-pages/man4/random.4.html) versions effectively does not. Be on the watch of what version/from what source you are reading.*

*2017-01-08 edit*
---

**Majority of web pages and blog posts I've read suggests to use `/dev/urandom`.**

##### 1
Most of them refer to [this article](http://www.2uo.de/myths-about-urandom/)

**When the best available quality is needed `/dev/random` should be used always...**

##### 2
For sure `/dev/random` should NOT be used always but only "when the best available ~~quality~~ entropy is needed". Largest variation in people opinions seems to be how big "quality difference" there is between `/dev/random` and `/dev/urandom`. Some would say there is none but I would say there is enough to write this.

**The absence of term [CSPRNG](https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator)**

As both `/dev/random` and `/dev/urandom` [backs](https://en.wikipedia.org/wiki//dev/random#Linux) to the [same](https://en.wikipedia.org/wiki/Salsa20#ChaCha_variant) (CS)PRNG at kernel (version 4.8 and newer) level it is left out of the scope. I would say it is a complete different story whether you should run your pseudo-random data through some algorithm X before using it in cryptography.

You should be more focused to the fact that even CSPRNG -backed sources does not magically create better entropy than the entropy of the source it uses has. At least `/dev/random` ensures there is enough estimated entropy at the time of read. Linux should not remove it as a legacy thingie.
