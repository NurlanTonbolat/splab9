# splab9 - bugs and deadlock

[**Chapter 32**](http://pages.cs.wisc.edu/~remzi/OSTEP/threads-bugs.pdf) homework. 
The code can be found [here](http://pages.cs.wisc.edu/~remzi/OSTEP/Homework/homework.html) 
Answer all the questions.


### Q1
_First let’s make sure you understand how the programs generally work, and some of the key options. Study the code in the file called vector-deadlock.c, as well as in main-common.c and related files._
_Now, run ./vector-deadlock -n 2 -l 1 -v, which instantiates two threads (-n 2), each of which does one vector add (-l 1), and does so in verbose mode (-v). Make sure you understand the output. How does the output change from run to run?_

---

__Output doesn't change from run to run.__

```
nurlan@serverx:~/Desktop/RealDeadlock$ ./vector-deadlock -n 2 -l 1 -v
              ->add(0, 1)
              <-add(0, 1)
->add(0, 1)
<-add(0, 1)
```

### Q2
_Now add the -d flag, and change the number of loops (-l) from 1 to higher numbers. What happens? Does the code (always) deadlock?_

---

__The code does not always deadlock. In fact, I could not get the code to deadlock until I ran it with 1,000 loops (`-l 1000`), and even then it would terminate as often as not.__

### Q3
_How does changing the number of threads (-n) change the out- come of the program? Are there any values of -n that ensure no deadlock occurs?_

---

__Yes. Set `-n 1`. All other values deadlock.__

### Q4
_Now examine the code in vector-global-order.c. First, make sure you understand what the code is trying to do; do you understand why the code avoids deadlock? Also, why is there a specialcase in this vector add() routine when the source and destination vectors are the same?_

---
__The code avoids deadlock because the order the locks are taken in is a total order, defined by the virtual memory address of the vector structure.
There is a special case when the source and destination are the same because in this case only one lock needs to be acquired. Without this special case, the code would try to acquire a lock it already held, guaranteeing a deadlock.__

### Q5
_Now run the code with the following flags: -t -n 2 -l 100000 -d. How long does the code take to complete? How does the total time change when you increase the number of loops, or the number of threads?_

---
```
nurlan@serverx:~/Desktop/RealDeadlock$ ./vector-global-order -t -n 2 -l 100000 -d
Time: 0.07 seconds
```
__
It takes `0.07 seconds`.
Running `-t -n 2 -d` with loops lengths 1e4, 1e5, 1e6 and 1e7 indicates that run time scales roughly linearly with loop length.
Running `-t -l 100000 -d` with number of threads 2, 20 and 99 indicates that run time scales roughly linearly with number of threads.__

### Q6
_What happens if you turn on the parallelism flag (-p)? How much would you expect performance to change when each thread is working on adding different vectors (which is what -p enables) versus working on the same ones?_

---


### Q7
_Now let’s study vector-try-wait.c. First make sure you understand the code. Is the first call to pthread mutex trylock() really needed? Now run the code. How fast does it run compared to the global order approach? How does the number of retries, as counted by the code, change as the number of threads increases?_

---

Below are run times with the options `-t -l 100000 -d`, varying the number of threads (`-n`):
```
nurlan@serverx:~/Desktop/RealDeadlock$ ./vector-try-wait -t -l 100000 -d -n 2 [4,8,16]
```
- 2 threads: Retries: 582859     Time: 0.16 seconds
- 4 threads: Retries: 1179871    Time: 0.45 seconds
- 8 threads: Retries: 5127351    Time: 2.15 seconds
- 16 threads: Retries: 28634864  Time: 16.34 seconds


Below are the run times under high parallelism `-t -l 100000 -d -p` (by construction, all of the runs had zero retries):
```
nurlan@serverx:~/Desktop/RealDeadlock$ ./vector-try-wait -t -l 100000 -d -p -n 2 [4,8,16]
```
- 2 threads: Retries: 0     Time: 0.05 seconds
- 4 threads: Retries: 0     Time: 0.05 seconds
- 8 threads: Retries: 0     Time: 0.09 seconds
- 16 threads: Retries: 0    Time: 0.26 seconds

__Under high contention, both time taken and the number of retries scale super-linearly (it looks like time scales faster than quadratic, but slower than exponential with base 2). Although in big O notation this scaling is much worse than the global lock order solution, this solution is significantly faster in absolute time for small numbers of threads. It is at least 5 times as fast for 2 threads, and the global lock order solution only catches up at around 11 threads.
Under high parallelism (with the `-p` flag) the run times are phenomenal, and far outstrip `vector-global-order`, by first one, and then for higher numbers of threads, two orders of magnitude.__

### Q8
_Now let’s look at vector-avoid-hold-and-wait.c. What is the main problem with this approach? How does its performance compare to the other versions, when running both with -p and without it?_

---
The main problem with this approach is that it is too coarse: the global lock (which protects the acquisition of all the other locks) will be under contention even when the vectors being manipulated by each thread is different.

`-t -l 100000 -d`, varying number of threads `-n`:
```
nurlan@serverx:~/Desktop/RealDeadlock$ ./vector-avoid-hold-and-wait -t -l 100000 -d -n 2 [4,8,16,32]
```
- 2 threads: Time: 0.08 seconds
- 4 threads: Time: 0.11 seconds
- 8 threads: Time: 0.20 seconds
- 16 threads:  Time: 0.40 seconds
- 32 threads: Time: 1.03 seconds

```

`-t -l 100000 -d -p` (with parallelism), varying number of threads `-n`:
```
nurlan@serverx:~/Desktop/RealDeadlock$ ./vector-avoid-hold-and-wait -t -l 100000 -d -p -n 2 [4,8,16,32]
```
- 2 threads: Time: 0.07 seconds
- 4 threads: Time: 0.10 seconds
- 8 threads: Time: 0.22 seconds
- 16 threads:  Time: 0.39 seconds
- 32 threads: Time: 0.77 seconds

The run times under heavy contention (without the `-p` flag) very close to those for `vector-global-order`. For the high parallelism (with the `-p` flag) case, `vector-avoid-hold-and-wait` performs more or less as it does under high contention, but about twice as slow as `vector-global-order` under high parallelism. Consequently, the comparison between `vector-global-order` and `vector-try-wait` more or less holds here as well.

### Q9
_Finally, let’s look at vector-nolock.c. This version doesn’t use locks at all; does it provide the exact same semantics as the other versions? Why or why not?_

---
__No. It is only atomic with respect to adding a pair of entries, not every pair of entries.__

### Q10
_Now compare its performance to the other versions, both when threads are working on the same two vectors (no -p) and when each thread is working on separate vectors (-p). How does this no-lock version perform?_

---
Run times when working on the same two vectors (high contention):
```
nurlan@serverx:~/Desktop/RealDeadlock$ ./vector-nolock -t -l 100000 -d -n2 [4,8,16,32]
```
- 2 threads: Time: 0.49 seconds
- 4 threads: Time: 0.53 seconds
- 8 threads: Time: 1.24 seconds
- 16 threads:  Time: 2.63 seconds
- 32 threads: Time: 5.08 seconds

Run times when each thread is working on separate vectors (high parallelism):
```
nurlan@serverx:~/Desktop/RealDeadlock$ ./vector-nolock -t -l 100000 -d -p -n 2 [4,8,16,32]
```
- 2 threads: Time: 0.17 seconds
- 4 threads: Time: 0.32 seconds
- 8 threads: Time: 0.60 seconds
- 16 threads:  Time: 1.30 seconds
- 32 threads: Time: 2.34 seconds
