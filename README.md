# Godot Notes

An unorganized rambling of things I've had to do with Godot. Any code here is CC-0.

## Coroutine Locks
Godot implements `Mutexes` for coordinating threads. However, they're not sutable for the task of sleeping and waking up *coroutines*. Using either would simplying cause the main thread to hang. For example, lets say the following code is some utility class:

```gdscript
class Sleeper:
  extends Resource
  
  var mutex := Mutex.new()
  
  static func wait() -> void:
    mutex.lock()
    yield(...)
```

In this example, the `yield` statement will never be reached. `Mutex::lock()` would hange the main thread. Furthermore, this would freeze the entire game. However, assuming it didn't freeze the game, what would we even wait for? `Mutex` doesn't implement any signals. Additionally, `Mutex::unlock` doesn't return a `GDScriptFunctionState` objects, thus `yield` wouldn't work. 

Unfortunatly, the following wouldn't work either:

```gdscript
class Sleeper:
  extends Resource
  
  signal notified()
  
  static func wait() -> void:
    yield(self, "notified")

#... somewhere else ...
func A():
  yield(Sleeper.wait(), "completed")

func B():
  yield(Sleeper.wait(), "completed")
```

How do we know what function to wake up? Both `A` and `B` would resume if `emit_signal("notified")` was called. While you could do the following:

```gdscript
func A():
  while yield(Sleeper.wait(), "completed") != "A":
    pass
    
func B():
  while yield(Sleeper.wait(), "completed") != "B":
    pass
```
It would become rather messy, imo. Therefore, my solution for this is to create my own lock. Which is as simple as the following:

```gdscript
class Lock:
  extends Reference
	
  signal unlocked()
	
  # Help function to make things look cleaner
  func unlock():
    emit_signal("unlocked")
```

which can be used in the following way:

```gdscript
class Sleeper:
  extends Resource
  
  const locks = {}

  static func wait(name: String) -> void:
    var lock = Lock.new()
    locks[name] = lock
    yield(lock, "unlocked")

  static func notify(who: String) -> void:
    if who in locks:
      locks[who].unlock()
      locks.erase(who)

# ... somewhere else ...
func A():
  yield(Sleeper.wait("A"), "completed")

func B():
  yield(Sleeper.wait("B"), "completed")
  
# ... somewhere away ...
Sleeper.notify("B")
  
# ... somewhere else away ...
Sleeper.notify("A")
```

Now ever time `Sleeper::wait(name: String)` is invoked, we can create a new istance of a `Lock`, store it into a hashmap, and then yeild on it's `"unlocked"` signal. Then, we can use `Sleep::notify(who: String)` to resume the correct function.

