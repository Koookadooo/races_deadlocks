# 
# Addressing Race Conditions and Deadlocks

## Part 1: Race Conditions

### Example 1: Incrementing x
#### Pseudocode
```
 1│ if x == 12:
 2│     x++
```
#### What can go wrong:
* T1 checks x (x == 12) and gets preempted before incrementing.
* T2 also checks x (x == 12) and increments x to 13.
* T3 does the same before T1 resumes.
* T1 resumes and increments x again (x = 14).
* Expected behavior: x should be 13 after all threads run once, but it ends up being 14 (or more, depending on timing and number of   threads).

#### Solution:
```
 1| lock(m1)
 2│ if x == 12:
 3│     x++
 4| unlock(m1)
```


### Example 2: Conditional Locking Increment
#### Psuedocode
```
 1│ if x == 12:
 2│     lock(m1)
 3│     x++
 4│     unlock(m1)
```

#### What can go wrong:
* T1 evaluates the condition if x == 12 and finds it true.
* Before T1 can execute lock(m1), it is preempted, and T2 starts executing.
* T2 also evaluates the condition if x == 12 and finds it true (since T1 has not yet incremented x).
* T2 acquires the lock m1, increments x to 13, and releases the lock.
* T1 resumes, acquires the lock m1 (now that it's released by T2), and increments x again to 14.
* Expected behavior: x should be incremented only once to become 13, assuming the initial value of x is 12 and that the condition is checked only once per thread. However, due to concurrent evaluation of the condition, x ends up being 14.

#### Solution:
```
 1│ lock(m1)
 2│ if x == 12:
 3│     x++
 4│ unlock(m1)
```


### Example 3: Hash Map Access
#### Psuedocode
```
 1│ if y not in hash:
 2│     hash[y] = 12
 3│ else
 4│     hash[y]++
```

#### What can go wrong:
* T1 checks if y is not in hash and finds it true.
* T2 also checks if y is not in hash simultaneously and finds it true because T1 has not yet added y to hash.
* Both T1 and T2 proceed to set hash[y] = 12.
* T3 checks, finds y in hash, and increments hash[y] to 13.
* Expected behavior: hash[y] should be 14 (initially set to 12, then incremented twice), but due to simultaneous access, it ends up being either 12 or 13.

#### Solution:
```
 1│ lock(m1)
 2│ if y not in hash:
 3│     hash[y] = 12
 4│ else
 5│     hash[y]++
 6│ unlock(m1)
```


### Example 4: Compound Addition
#### Psuedocode
```
 1| x += 12
```

#### What can go wrong:
* T1 reads x (0) and calculates x + 12 (12).
* Before T1 writes back 12, T2 reads x (0) and also calculates x + 12 (12).
* T3 does the same, each thinking x is still 0 when they read it.
* All three write 12 to x consecutively.
* Expected behavior: x should be 36 (0 + 12 + 12 + 12) after all three threads run, but it ends up being just 12.

#### Solution:
```
 1│ lock(m1)
 2│ x += 12
 3│ unlock(m1)
```


### Example 5: Semaphore Implementation
#### Psuedocode
```
 1│ semaphore_init(value):
 2│     x = value
 3│
 4│ semaphore_signal():
 5│     x++
 6│
 7│ semaphore_wait():
 8│     while x == 0:
 9│         do nothing  # spinlock
10│
11│     x--
```

#### What can go wrong:
* T1 calls semaphore_wait() and checks x (line 8), which is initially 1.
* T1 proceeds beyond the spinlock to decrement x (line 11).
* Almost simultaneously, T2 also calls semaphore_wait(), and since there hasn't been a context switch or memory update delay, it also sees x as 1 (passing the spinlock condition at line 8).
* T1 decrements x to 0.
* T2, unaware that T1 has already decremented x, also decrements x, now making x -1.
* Expected behavior: The semaphore's counter (x) should accurately reflect the number of available resources and should never fall below zero, but in this case we get -1.

#### Solution:
```
 1│ semaphore_init(value):
 2│     x = value
 3│     lock = False  # Initialize the lock as free (False)
 4│
 5│ semaphore_signal():
 6│     while test_and_set(lock):  # Attempt to acquire the lock
 7│         do nothing  # Busy-wait if lock is already True (locked)
 8│     x += 1  # Safely increment x
 9│     lock = False  # Release the lock
10│
11│ semaphore_wait():
12│     while True:  # Continuously attempt to decrement x safely
13│         while test_and_set(lock):  # Attempt to acquire the lock
14│             do nothing  # Busy-wait if lock is already True
15│         if x > 0:  # Check if there are resources to consume
16│             x -= 1  # Consume a resource
17│             lock = False  # Release the lock
18│             break  # Exit loop after successful operation
19│         lock = False  # Release the lock if no resource to consume
```

# 
## Part 2: Deadlocks

### Example 1: Out of order
#### Psuedocode
```
 1│ function1():
 2│     lock(m1)
 3│     lock(m2)
 4│
 5│     unlock(m2)
 6│     unlock(m1)
 7│
 8│ function2():
 9│     lock(m1)
10│     lock(m2)
11│
12│     unlock(m1)
13│     lock(m1)
14│
15│     unlock(m2)
16│     unlock(m1)
```

#### What can go wrong:
* T1 locks m1 and m2 in sequence and then releases both, one after the other.
* T2 locks m1 and then m2. After doing its operations, it releases m1 and tries to lock it again.
* Before T2 re-locks m1, T1 starts again and locks m1.
* T1 tries to lock m2 but can't because T2 has it locked.
* T2 is also stuck waiting for T1 to release m1.
* Both threads are waiting for the other to release a lock they need, neither can proceed, resulting in a deadlock.

#### Solution:
```
 1│ function1():
 2│     lock(m1)
 3│     lock(m2)
 4│
 5│     unlock(m2)
 6│     unlock(m1)
 7│
 8│ function2():
 9│     lock(m1)
10│     lock(m2)
11│
12│     unlock(m1)
13│     temp = try_lock(m1)  # Attempt to reacquire m1 without blocking
14│     if not temp:
15│         unlock(m2)
16│         return  # Exit if m1 cannot be immediately reacquired
17│     unlock(m1)
18│
19│     unlock(m2)
```


### Example 2: Twisting little passages, all different...
#### Psuedocode
```
 1│ function1(m1, m2):  # Mutexes are passed in by caller
 2│     lock(m1)
 3│     lock(m2)
 4│
 5│     unlock(m2)
 6│     unlock(m1)
```

#### What can go wrong:
* Assume T1 calls function1(m1, m2) and T2 calls function1(m2, m1).
* T1 locks m1 and then m2.
* Simultaneously, T2 locks m2 (while T1 holds m1 but before locking m2) and then tries to lock m1, resulting in a deadlock. Each thread holds one mutex and waits for the other, neither able to proceed.

#### Solution: 
```
 1│ function1(m1, m2):
 2│     if id(m1) < id(m2):  # Ensure the lower memory address mutex is always locked first
 3│         first, second = m1, m2
 4│     else:
 5│         first, second = m2, m1
 6│     lock(first)
 7│     lock(second)
 8│
 9│     unlock(second)
10│     unlock(first)
```