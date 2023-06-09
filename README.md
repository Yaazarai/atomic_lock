# atomic_mutex & atomic_lock
Memory Order safe solution to std::lock_guard using atomics. On some platforms when using `std::lock_gaurd` you can experience a crash due to mutexes not being memory-order safe on multi-threaded calls (I experienced this on AMD APUs). Atomics force memory ordering, so we use atomics to signal and check for the usage of the mutex lock prior to actually locking, making calls memory order safe. A simple optional timeout check using `std::chrono` is also provided to force waiting for a lock and exiting/failing after a certain amount of time (in milliseconds) has passed.

Usage is pretty straight forward, the same as `std::lock_guard`:
```C++
atomic_mutex amutex;

void some_threadedfunc(...) {
    atomic_lock alock(amutex);
    
    if (alock.AcquiredLock()) {
        ...
    }
}
// OR using timeout:
void some_threadedfunc(...) {
    atomic_lock alock(amutex, true, 1000 /* 1,000 ms */);
    
    if (alock.AcquiredLock()) {
        ...
    }
}
```
