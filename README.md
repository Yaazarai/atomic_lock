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

Entire single-file header code:
```C++
#pragma once
#ifndef ATOMIC_LOCK
#define ATOMIC_LOCK
	#include <mutex>

	struct atomic_mutex {
	public:
		std::atomic_bool signal;
		std::mutex lock;
	};

	class atomic_lock {
	private:
		std::atomic_bool signal;
	public:
		atomic_mutex& lock;

		atomic_lock(atomic_mutex& lock, bool wait = false, size_t timeout = 100) : lock(lock) {
			signal = static_cast<bool>(lock.signal);

			if (wait) {
				std::chrono::time_point<std::chrono::system_clock> now = std::chrono::system_clock::now();
				std::chrono::milliseconds startTime = duration_cast<std::chrono::milliseconds>(now.time_since_epoch());
				std::chrono::milliseconds endTime = duration_cast<std::chrono::milliseconds>(now.time_since_epoch());
				
				while (signal) {
					endTime = duration_cast<std::chrono::milliseconds>(now.time_since_epoch());
					if (endTime.count() - startTime.count() >= static_cast<long long>(timeout)) break;
					signal = static_cast<bool>(lock.signal);
				}
			}

			lock.signal = !signal;
			if (signal == false) lock.lock.lock();
		}

		~atomic_lock() { if (!signal) { lock.signal = false; lock.lock.unlock(); } }
		
		bool AcquiredLock() { return !signal; }
		
		void ForceUnlock() { if (!signal) { signal = true; lock.signal = false; lock.lock.unlock(); } }
	};

#endif
```
