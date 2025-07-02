<img src="Assets/GrumpyLinux.jpg" alt="Grumpy Linux" width="480" align="center">

# Grumpy Linux 🦉

**(Because sometimes, you just want to build it yourself and glare at the world from your perfectly optimized terminal.)**

---

## 😠 What is Grumpy Linux?

This is not your grandma's Linux distribution. Heck, it's barely *anyone's* Linux distribution right now. Grumpy Linux is my personal, meticulously crafted, and frankly, **highly opinionated** Linux From Scratch (LFS) build, primarily for my Dell Latitude 7420 (the source of much digital angst, hence the name).

Think of it as the antithesis of "user-friendly." If it doesn't compile from source, look it up in a manual, or require at least two hours of head-scratching, it probably isn't Grumpy Linux.

It's for those of us who:
* Have tried all the other distros and found them... *wanting*.
* Believe bloatware is a personal insult.
* Enjoy the therapeutic click-clack of a keyboard compiling GCC for the 17th time.
* Prefer their operating system to be as lean, mean, and subtly judgmental as a well-exercised cat.
* ...and genuinely want to understand how the damn thing works from the ground up.

## 🛠️ Key (Grumpy) Features

* **Barely-There Base:** We started with almost nothing. And we liked it that way. Every single byte here was *earned*.
* **Optimal Disgruntlement:** Custom-tuned for the Intel Core i7-1185G7 and Iris Xe Graphics on a Dell Latitude 7420. Expect things to just *work*, grudgingly.
* **No Unnecessary Friends:** We've ruthlessly purged anything that doesn't contribute directly to its function. Your feelings? Not a function.
* **Terminal-First Mentality:** Why click when you can type? Why use a GUI when you can `vim`? We're not savages.
* **Pure, Unadulterated Control:** If you don't know why it's there, it probably isn't. And if it is, you can change it. (But don't come crying to me if you break it.)

## 😤 Why "Grumpy"?

Because sometimes, building an OS from scratch feels like wrestling a particularly stubborn badger. There are moments of frustration, the occasional syntax error that makes you question your life choices, and the sheer audacity of certain upstream projects to change their build systems.

But through it all, there's a deep, quiet satisfaction. A satisfaction that only comes from knowing you tamed the beast, even if it still gives you the occasional side-eye. This OS embodies that spirit. It works, it's efficient, but don't expect it to smile.

## 🚧 Building Grumpy Linux (If You Dare)

**WARNING: This is not for the faint of heart, the easily discouraged, or those who prefer pre-packaged solutions.**

This repository serves as a personal log and potential guide for building Grumpy Linux. It closely follows the [Linux From Scratch 12.1](http://www.linuxfromscratch.org/lfs/view/12.1/) book. I highly recommend you start there and use my notes as... well, notes.

* **You will need:** Patience, caffeine, a functioning host Linux system (I'm using Linux Mint in a VirtualBox VM for the build process – *don't judge my methods, they work*), and an unwavering belief that you are better than pre-compiled binaries.
* **Consult the `BUILD.md`:** Once I get around to properly documenting my inevitable deviations and specific hardware hacks for the Latitude 7420, it will live there.

## 📜 Contributing (Grudgingly Accepted)

Found a bug? Have a brilliant optimization? Think something could be less... *grumpy*?

* Open an issue. Try to be clear.
* Submit a pull request. Make sure it doesn't break anything.
* If your contribution makes Grumpy Linux *less* grumpy, I might look at it suspiciously.
* If it makes it *more* stable, *more* efficient, or allows for *more* righteous indignation, then perhaps we can talk.

##  License

This project is licensed under the [MIT License](LICENSE.md).
(Which basically means: do what you want with it, but don't blame me if it breaks your sanity. I already warned you.)

---
_A special thanks (and perhaps a begrudging nod of respect) to the Linux From Scratch community. Without their tireless work, I'd still be stuck trying to compile a compiler._
---

*(Image credit: My own glorious vision of a grumpy owl.)*
