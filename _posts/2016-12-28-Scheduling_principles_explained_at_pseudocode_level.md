# Scheduling principles explained at pseudocode level

*TL;DR Most operating systems (like Linux) provides you the scheduler and signaling out of the box but you really should take these hints when dealing with embedded systems like RISC microcontrollers where there is no operating system present.*

<!--more-->

A goal is to gain as much profitable thread time with most little code complexity as possible.

* thread time is calculated during one full led blink cycle (on-off).
* led blinking interval must be stationary (no variation, 1sec on, 1sec off)

**Profitable thread time or not?**

* we assume `led_` operations takes 1 milliseconds thread time for each >> profitable
* `something_useful()` takes all thread time it gets without interfering with leds during one full cycle >> profitable
* sleeping is waste of thread time >> non-profitable

Blocking, no scheduling
---

```C
void main() {
  while(true) {
    led_on()
    ms_sleep(999)
    led_off()
    ms_sleep(999)
  }
}
```

Problem here is there is no way to do something other useful in this thread. Nearly all time is wasted for thread sleeping.

Profitable thread time: ~2 ms

Blocking, very simple scheduling
---

```C
// Flip-flop variable for software interrupt
bool soft_interrupt = false

// Hardware interrupt, run every 1000 milliseconds
void hard_interrupt() {
  soft_interrupt = true
}

void main() {
  while(true) {
    if(soft_interrupt) {
      soft_interrupt = false
      led_on()
      ms_sleep(999)
      led_off()
    }
    
    // Time to do something useful
    something_useful()
  }
}
```

Problem here is that one call to `something_useful()` cannot take much time to complete or else it will interfere with led blinking.. Also other part of that 1000 milliseconds is always wasted for thread sleeping.

Profitable thread time: ~999 ms

Non-blocking, state-based simple scheduling
---

```C
// Flip-flop variables for software interrupts
bool soft_interrupt_1, soft_interrupt_2 = false

// Hardware interrupt, run every 1000 milliseconds
void hard_interrupt() {
  soft_interrupt_1 = true
}

void main() {
  while(true) {
    if(soft_interrupt_1) {
      soft_interrupt_1 = false
      
      if(soft_interrupt_2) {
        soft_interrupt_2 = false
        led_off()
      } else {
        soft_interrupt_2 = true
        led_on()
      }
    }
    
    // Time to do something useful
    something_useful()
  }
}
```

Now as there is no thread sleeping at all there is nearly 2000 milliseconds profitable time to do something useful. Having same problem about interference as above example there is also new problem; if program's complexity grows new flip-flop variables are needed and managing those variables and if-else pairs will become a huge job.

Profitable thread time: ~1998 ms

Non-blocking, time-based simple scheduling
---

```C
// Flip-flop variables for software interrupts
bool soft_interrupt_1, soft_interrupt_2 = false
int ms_counter = 0

// Hardware interrupt, run every 1 milliseconds
void hard_interrupt() {
  if(ms_counter == 1000) {
    soft_interrupt_1 = true
  }
  
  if(ms_counter >= 2000) {
    ms_counter = 0
    soft_interrupt_2 = true
  }
  
  ms_counter ++
}

void main() {
  while(true) {
    if(soft_interrupt_1) {
      soft_interrupt_1 = false
      led_on()
    }
    
    if(soft_interrupt_2) {
      soft_interrupt_2 = false
      led_off()
    }
    
    // Time to do something useful
    something_useful()
  }
}
```

Now as there is again no thread sleeping, nearly 2000 milliseconds profitable time is gained. There is again new problem; time is wasted for incrementing and checking `ms_counter` value.

Profitable thread time: ~1995 ms

Non-blocking, more advanced scheduling (round-robin-like)
---

In this example we assume `led_blinker` and `whatever_work` operations are complete tasks containing **signaling** support from and to the scheduler and these tasks always:

* informs the scheduler when work is done by raising `DONE` flag
* informs the scheduler when willing to sleep by raising `SLEEP` flag
* obeys `INTERRUPT` signal from the scheduler thus deciding to continue sleeping or raising `DONE` flag
* obeys `KILL` signal from the scheduler thus stopping all ongoing sleeping/work immediately and raising `DONE` flag

There is also `run_for_ms()` operation present which is capable of running a given task at a given amount of time only and then continuing to next task leaving previous task to an unfinished state (no flags).

```C
const int MAX_TASKS = 2
const int MAX_TIME = 1000
const int TIMESLOT = 10

int *task[MAX_TASKS]
int task_runtime_ms[MAX_TASKS]

// Hardware interrupt, run every 1 milliseconds
void hard_interrupt() {
  for (k = 0; k < MAX_TASKS; k++) {
    // Increase runtime counter of this task if running
    if(task_runtime_ms[k] > 0) {
      task_runtime_ms[k] ++
    }
    
    // To detect deadlocks where there is no
    // SLEEP nor DONE flag seen in MAX_TIME milliseconds
    if(task_runtime_ms[k] > MAX_TIME) {
      signal(task[k], KILL)
    }

    // If task is done then no further timeslots are
    // needed until next run()
    if(has_flag(task[k], DONE)) {
      task_runtime_ms[k] = 0
    }
  }
}

void led_blinker() {
  led_on()
  flag(SLEEP, 999)
  led_off()
  flag(SLEEP, 999)
}

void whatever_work() {
  something_useful()
  flag(DONE)
}

void run(int k) {
  task_runtime_ms[k] = 1
}

void main() {
  // Introduce all tasks to the scheduler
  task[0] = &led_blinker
  task[1] = &whatever_work

  // Run all tasks
  // Tasks can be run in any other part on code as well
  run(0)
  run(1)

  while(true) {
    for (k = 0; k < MAX_TASKS; k++) {
      if(task_runtime_ms[k] > 0) {
        if(has_flag(task[k], SLEEP)) {
          // Reset runtime to prevent task killing
          // during deadlock detection
          task_runtime_ms[k] = 1
          signal(task[k], INTERRUPT)
        } else {
          // Run task for TIMESLOT amount of time
          run_for_ms(task[k], TIMESLOT)
        }
      }
    }
  }
}
```

That ampersand notation in `&led_` is just a pointer which is stored and used later with `run_for_ms()`

Now look at `void led_blinker()` operation from the last code snippet and compare it to the first code snippet in this article. See any similarities? Once you have implemented a proper scheduler you don't have to poison your actual application code with all kinds of "scheduling" attempts presented above in this article.

Epilogue
---

As you see led behaviour is getting more and more unpredictable. You cannot say for sure how many milliseconds led is on and off but at least it is something close to 1000ms +/- 10ms. This is the cost of scheduling as there is always variance present.

There are real-time operating systems present (like RTLinux). Schedulers of those systems are more predictable in behaviour than in non-real-time operating systems. There is also real-time versions of normal Linux kernel available. Still predictability is not always the requirement and other than real-time schedulers can be used.

Last pseudo-scheduler example in this article is far from being perfect. Do not use it. **Make it better.**

*TL;DR Most operating systems (like Linux) provides you the scheduler and signaling out of the box but you really should take these hints when dealing with embedded systems like RISC microcontrollers where there is no operating system present.*

What next
---

You should now understand how scheduling with signaling works.

Now go away and maybe read more advanced scheduler code from Linux kernel who knows why:

https://github.com/torvalds/linux/blob/master/kernel/sched/core.c

Also take a look at different scheduling disciplines from Wikipedia:

https://en.wikipedia.org/wiki/Scheduling_(computing)#Scheduling_disciplines

and Posix signals:

https://en.wikipedia.org/wiki/Unix_signal#POSIX_signals
