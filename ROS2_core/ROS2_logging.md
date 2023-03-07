# Study of rclcpp logging with focus on realtime-compliance

## Introduction

This study started with a simple question: "Is it ok to include log macro calls in realtime code, so you can log during debugging but disable it through the loglevel during normal operation?". The result is an almost complete walktrough through the code, describing all aspects of the current ROS 2 logging subsystems in rcl, rcutils and rclcpp.</br>
I hope it can be useful to many, and maybe even incite some to also start looking into the ROS 2 core implementation. I'm looking forward to reading your notes!

With respect to the realtime-compliance of the code, I mainly looked for:
- Heap allocations, and
- Blocking calls (e.g. locking a mutex).

Any other things that might be relevant, please let me know!

Furthermore, some conclusions are based on assumptions, which obviously I think are correct but which I did not formally check (e.g. the assumption that lambda functions are in stack memory (or some other preallocated region) and hence don't allocate, unless of course if there would be an actual allocation inside the lambda function).</br>
</br>
If there are inaccuracies or incorrect statements in below description, please correct me!

</br>

## TL;DR

Logging is _obviously_ not realtime-compliant.</br>
It is possible however to use log macro calls in your code, and disable them at runtime through the loglevel.</br>
This could be useful e.g. for debug output while testing, without compromising realtime behavior during normal operation.


</br>

However, even when disabled some function calls are still made, so <span style="color:red">**following rules are necessary conditions**</span> for realtime-compliance:

- Don't use the streaming macros (e.g. RCLCPP_DEBUG_STREAM), since these allocate using plain `malloc`, **before** the loglevel is evaluated and logging is discarded.</br>

- Call the logger macros with:
  - `node::get_logger()`,
    - E.g.&emsp;`RCLCPP_DEBUG(this->get_logger(), "Log string")`,
  - Or:&ensp;pass a logger member variable that was instantiated outside the realtime loop,
    - E.g.&emsp;`RCLCPP_DEBUG(my_logger_, "Log string")`,
  - But do **not** use `rclcpp::get_logger("some_name")`,
    - E.g.&emsp;`RCLCPP_DEBUG(rclcpp::get_logger("some_name"), "Log string")`, since this allocates on each call, using plain `malloc`.
    </br></br>


- Each log macro call eventually calls `rcutils_logging_get_logger_effective_level(name)` which ([on some occasions](#ancestorallocation)) allocates memory. Hence:
  - rclcpp should be initialized with a realtime-compliant memory allocator,
  - **This seems currently not possible due to [this](https://github.com/ros2/rcl/issues/1036) and [this](https://github.com/ros2/rcl/issues/1037),**
  - As a workaround: don't use child loggers (i.e. with dot separated names such as "parent_logger.child_logger", or by call of `Logger::get_child("name")`) for log macros in a realtime part of your code.

</br>

### A few peculiarities about the /rosout publisher

- A publisher is created **only** for _node_ loggers, and **not** for _non-node_ loggers.</br>
  An example:
  - `RCLCPP_INFO(node_->get_logger(), "Message")` is published to /rosout, but
  - `RCLCPP_INFO(rclcpp::get_logger("name"), "Message")` is not published to /rosout.</br>
  </br>

- Child loggers publish using the publisher of the parent logger.</br>
This is configured in the call to `Logger::get_child("child_name")`, and is active for the lifetime of the child logger.</br>
  This implies that:
  - Calling `get_child` on a _non-node_ logger throws an exception and exits your program,
  - Log calls with the full name of a child logger, e.g. `get_logger("node_name.child_name")`, will only publish to /rosout if there still is an instance of `Logger::get_child("child_name")` concurrently valid in some scope.

</br>

E.g. this will **not** output "Log 2" to /rosout:</br>
```c++
    auto node = std::make_shared<rclcpp::Node>("LoggerNode");
    RCLCPP_INFO(node->get_logger().get_child("child"), "Log 1");   // child logger only exists in scope of this log call
    RCLCPP_INFO(rclcpp::get_logger("LoggerNode.child"), "Log 2");  // publishing no longer configured
```
</br>

But this will:
```c++
    auto node = std::make_shared<rclcpp::Node>("LoggerNode");
    auto childlogger = node->get_logger().get_child("child");
    RCLCPP_INFO(node->get_logger().get_child("child"), "Log 1");  // new instance of child logger in local scope, publishing is already configured
    RCLCPP_INFO(rclcpp::get_logger("LoggerNode.child"), "Log 2"); // publishing still active as childlogger still exists
```
  
</br>

Note though, that using the full name of children for node loggers is not recommended anyway, as the node name could be remapped at launch.

</br>
</br>

The rest of this document gives a detailed overview of the code path and logic wrt. logging from rclcpp / rcutils / rcl.

</br>
</br>




## rclcpp Logger class

Function calls such as `this->get_logger()` or `rclcpp::get_logger("logger_name")` are generally being described as "giving you _the node's logger_" or "giving you _the named logger_". Because of this, one would expect that these instantiations of the logger class perform the actual logging.

This is however not the case: rclcpp "loggers" don't do any logging: logging is done solely through the macro calls, that eventually resolve to rcutils function calls.


The [rclcpp::Logger](https://github.com/ros2/rclcpp/blob/b1e834a8dfb0e2058fbf0e3adcc06cd8eda86243/rclcpp/include/rclcpp/logger.hpp#L51) itself has little functionality:
- It holds a name,
- It has a method [`set_level(Level level)`](https://github.com/ros2/rclcpp/blob/b1e834a8dfb0e2058fbf0e3adcc06cd8eda86243/rclcpp/src/rclcpp/logger.cpp#L111) which passes the name and loglevel to `rcutils_logging_set_logger_level(name, level)`.
  - This sets the loglevel into a hash table (contains logger name hash and corresponding loglevel value, for lookup on each log call). More on this below.
- It has some other functions, such as:
  - [`get_name()`](https://github.com/ros2/rclcpp/blob/b1e834a8dfb0e2058fbf0e3adcc06cd8eda86243/rclcpp/include/rclcpp/logger.hpp#L140): returns the name,
  - [`get_child(child_name)`](https://github.com/ros2/rclcpp/blob/b1e834a8dfb0e2058fbf0e3adcc06cd8eda86243/rclcpp/src/rclcpp/logger.cpp#L71): returns a Logger instance with name "logger_name.child_name" and configures the rosout logging system for publishing output to /rosout.</br>

</br>

Re. realtime compliance:
- The [logger constructor](https://github.com/ros2/rclcpp/blob/3062dec77e4695dc0dc8a2efe764dd18c2ad47da/rclcpp/include/rclcpp/logger.hpp#L122-L123) allocates a `std::string` with standard allocator (`malloc`)
  - `rclcpp::get_logger(name)` calls the constructor and returns by value,
  - `rclcpp::get_node_logger` idem,
- [`rclcpp::get_logging_directory`](https://github.com/ros2/rclcpp/blob/b1e834a8dfb0e2058fbf0e3adcc06cd8eda86243/rclcpp/src/rclcpp/logger.cpp#L57) calls `rcl_logging_get_logging_directory` explicitly with default allocator (i.e. `malloc`),
- `Logger::get_child()` also allocates memory
- So basically these are all <span style="color:red">**not realtime-compliant**</span>
- [`node::get_logger()`](https://github.com/ros2/rclcpp/blob/33dae5d679751b603205008fcb31755986bcee1c/rclcpp/src/rclcpp/node_interfaces/node_logging.cpp#L19-L33) returns a Logger by value,
  - Given a Logger holds its name as an `std::shared_ptr<const std::string>`, which upon copy by the copy constructor does not allocate new memory for the string, I assume this is ok.
- Conclusion:
  - Calling the logger macros with `rclcpp::get_logger("some_name")` is <span style="color:red">**not realtime-compliant**</span>
  - Calling the logger macros with `this->get_logger()` <span style="color:red">**is realtime-compliant**</span> (wrt the logger object that is; for the inner workings of the macros see below).


</br>
</br>

## rclcpp logger macro's

Below description covers the 'DEBUG' loglevel macros, but it is obviously identical for the other loglevels (INFO / WARNING / etc).
</br>
The rclcpp logger macros are in `logging.hpp`, which is generated at compile time from `rclcpp/resource/logging.hpp.em` (i.e. `logging.hpp` is in the build and install folder, but not in the ROS 2 source).

</br>

RCLCPP_DEBUG / RCLCPP_DEBUG_EXPRESSION / RCLCPP_DEBUG_FUNCTION / RCLCPP_DEBUG_SKIPFIRST
- These macros have a compile-time assert that 'logger' is an `rclcpp::Logger`
- Followed by call of an **rcutils** macro, respectively: 
    - `RCUTILS_LOG_DEBUG_NAMED`
    - `RCUTILS_LOG_DEBUG_EXPRESSION_NAMED`
    - `RCUTILS_LOG_DEBUG_FUNCTION_NAMED`
    - `RCUTILS_LOG_DEBUG_SKIPFIRST_NAMED`

</br>

RCLCPP_DEBUG_THROTTLE / RCLCPP_DEBUG_SKIPFIRST_THROTTLE
- Similar to above, but also define a lambda function to return the current time
  - <span style="color:red">**Assumption: stack allocated --> realtime-compliant**, but I did not check the time functions.</span>
- Respective rcutils macro calls:
  - `RCUTILS_LOG_DEBUG_THROTTLE_NAMED`
  - `RCUTILS_LOG_DEBUG_SKIPFIRST_THROTTLE_NAMED`

</br>

RCLCPP_DEBUG_STREAM  / 
RCLCPP_DEBUG_STREAM_ONCE / etc
- Similar to above, but these macros first stream the arguments into a `std::stringstream` and then call the regular macros, passing on the combined string retrieved from the stringstream.
  - <span style="color:red">**Heap allocation, no custom allocator</br>
    --> not realtime-compliant, even if log is disabled through the loglevel**</span>

</br>

The rclcpp logging macros can be disabled at compile time:
 - Define `RCLCPP_LOG_MIN_SEVERITY=RCLCPP_LOG_MIN_SEVERITY_[DEBUG|INFO|WARN|ERROR|FATAL]`</br>
   in your build options to compile out anything below that severity.
 - Use `RCLCPP_LOG_MIN_SEVERITY_NONE` to compile out all macros.

</br>
</br>

## rcutils log functionality

All rcutils logging macros start with a call to `RCUTILS_LOGGING_AUTOINIT`, which initializes the logging if it was not initialized yet.</br>
 - By default however, `rclcpp::init()` initializes logging before any log call, more on that [below](#rclcpp_init). 

</br>

 ### RCUTILS_LOGGING_AUTOINIT

- [RCUTILS_LOGGING_AUTOINIT](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/include/rcutils/logging.h#L560) first checks if logging is already initialized, and, if not:
- Calls [`rcutils_logging_initialize(void)`](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L129-L132) which calls [`rcutils_logging_initialize_with_allocator(rcutils_allocator_t)`](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L513) with the default allocator
  - Default allocator uses standard `malloc/free/realloc/calloc` (stdlib.h)
  - Question: Is there a way to pass a different allocator?</br>
    &emsp;&emsp;Is this done by manually calling `rcutils_logging_initialize_with_allocator(rcutils_allocator_t)` with a custom `rcutils_allocator_t` before using any logger macro or function?</br>
    ---> ANSWER: it should be possible through the `InitOptions`, though there currently are certain issues, see description [below](#rclcpp_init) wrt. initialisation from `rclcpp::init`.

</br>

### `rcutils_logging_initialize_with_allocator(allocator)`

</br>

- `g_rcutils_logging_allocator` is set to the given `allocator` if the allocator is valid ([code](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L519-L523))

- `g_rcutils_logging_output_handler` is a **function pointer** to the current output handler ([code](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L525))
  - It is set here to `rcutils_logging_console_output_handler()`
  - This is lateron changed by `rclcpp::init()` to an `rclcpp_logging_multiple_output_handler`, which calls this  `rcutils_logging_console_output_handler()` as well as the rosout and external log library handler.
  </br>
  </br> 
- Environment variables:
  - `RCUTILS_CONSOLE_STDOUT_LINE_BUFFERED`: no longer used, outputs an error message and is ignored - [code](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L528-L541)
  - `RCUTILS_LOGGING_USE_STDOUT`: to output to stdout instead of stderr - [code](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L543-L563)
  - `RCUTILS_LOGGING_BUFFERED_STREAM` - [code](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L570-L591):
    - Empty = default of the stream,
    - 0 = force unbuffered,
    - 1 = force line buffered.
  - Correspondingly a call to `setvbuf` to configure the buffering - [code](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L580)
  
  - `RCUTILS_COLORIZED_OUTPUT` - [code](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L593-L612)
    - Empty = auto,
    - 0 = force disable, 
    - 1 = force enable.
    
  - `RCUTILS_CONSOLE_OUTPUT_FORMAT`: custom output format string - [code](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L614-L633)
    - Output format string is copied to `g_rcutils_logging_output_format_string` but truncated to MAX_OUTPUT_FORMAT_LEN,
    - If the environment variable was not set or could not be read, the default is used: `"[{severity}] [{time}] [{name}]: {message}"`

</br>

- `g_rcutils_logging_severities_map` is initialized ([code](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L635-L646)). This is a static `rcutils_hash_map_t`.
  - This maps logger names to loger severities.
    - Logger names are stored as hashes instead of strings as this is more efficient.
    - On each logging call:
      - It is checked if the map contains a hash corresponding to that log call name, and
      - If so: the corresponding severity is read from the map,
      - If not: the default severity is used,
      - If the severity of the macro call >= the hashmap severity, the message is logged.
  - First, an empty hashmap is requested,
  - Then it is initialized with following arguments:
    - Initial size is 2 (comments state that this has to be power of 2 due to some optimization on lookup),
    - Key size: sizeof(const char*),
    - Data size: sizeof(int),
    - Hash function,
    - Compare function,
    - Allocator
    </br>
    </br>
- `parse_and_create_handlers_list()` is called<a id="parse_and_create_handlers_list"></a>
  - This parses `g_rcutils_logging_output_format_string`. This variable:
    - Either has been read from environment variable `RCUTILS_CONSOLE_OUTPUT_FORMAT` (see above), 
    - Or was set to the default output format string: `"[{severity}] [{time}] [{name}]: {message}"`,
  - The parser installs handler functions:
    - That add the corresponding values to the output log string, according to each valid `{item}`,
    - And to copy all other parts of the format string to the output string (i.e. those parts that do not correspond to valid `{items}`).
  - Implementation is imo hard to read as it has little comments:
    - [These lines](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L452-L453) appear to handle the case where there is no start delimiter left:
      - `rcutils_find()` does not find a start delimiter and hence returns `SIZE_MAX`
      - Hence `chars_to_copy` is set to the remaining characters 
    - [This line](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L454) adds a handler `copy_from_orig` to the array of handlers, with `start_offset = i` and `end_offset =  i + chars_to_copy`
      - When formatting a log string, this handler will copy the corresponding part of the original log formatting string into the output.
    - [This](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L464) then jumps back to the start of the while loop (I think, but it seems not necessary??), 
      - `rcutils_find` is called again for start delimiter and returns zero, so the if clause is skipped
    - We arrive at [another call](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L472) to `rcutils_find`, this time for the end delimiter,
      - [If no end delimiter is found](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L475-L483) in the remainder of the formatting string (same logic with `SIZE_MAX`), add a handler to copy the remainder of this string,
      - [Else](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L485-L506):
        - Copy that part of the string into `token`,
        - Check if `token` actually is a token (`find_token_handler`)
          - If so: add the corresponding handler,
          - If not: add the `copy_from_orig` handler
</br>

- And finally, `g_rcutils_logging_severities_map_valid` and `g_rcutils_logging_initialized` are set to `true`.</br>

This concludes `RCUTILS_LOGGING_AUTOINIT`.</br>

</br>
</br>
</br>

Following to the `RCUTILS_LOGGING_AUTOINIT` call, each of the rcutils macros resolves to a call of:</br>
```c++
RCUTILS_LOG_COND_NAMED(severity, condition_before, condition_after, name, ...)
```
Functional differences between the macros (e.g. throttle, skipfirst, etc) is realized through the `condition_before` and `condition_after` macro arguments of that macro (see below).


</br>
</br>
</br>

### RCUTILS_LOG_COND_NAMED

Initializes static struct with the calling function name, file name and line number ([code](https://github.com/ros2/rcutils/blob/f0bee9e66830ababfe1e0f2414d46e36a0b60ec5/resource/logging_macros.h.em#L69)), 
<span style="color:red">**memory preallocated since it is static --> realtime-compliant**</span> 

Then, it checks if logging is enabled for that name and severity, by calling `rcutils_logging_logger_is_enabled_for(name, severity)` (see below).</br>
If so, it concatenates:

```c++ 
    condition_before  rcutils_log_internal()  condition_after
```

In this,&ensp;`condition_before`&ensp;and&ensp;`condition_after`&ensp;are&ensp;`#define`&ensp;statements, which add conditional code around the call of `rcutils_log_internal()`.

</br>

E.g. for `CONDITION_EXPRESSION` the defines are as follows:

```c++
    #define RCUTILS_LOG_CONDITION_EXPRESSION_BEFORE(expression)  if (expression) {
  
    #define RCUTILS_LOG_CONDITION_EXPRESSION_AFTER }
```
so the concatenation results in:

```c++
    if (expression) {   rcutils_log_internal();   }
```

</br>

Similar expressions are defined for the other log macros (throttle, skip first, etc).

As an example, this is the implementation of `RCLCPP_DEBUG_ONCE`:

```c++
#define RCLCPP_DEBUG_ONCE(logger, ...) \
  do { \
    static_assert( \
      ::std::is_same<typename std::remove_cv_t<typename std::remove_reference_t<decltype(logger)>>, \
      typename ::rclcpp::Logger>::value, \
      "First argument to logging macros must be an rclcpp::Logger"); \
 \
    RCUTILS_LOG_DEBUG_ONCE_NAMED( \
      (logger).get_name(), \
      __VA_ARGS__); \
  } while (0)
```

`RCUTILS_LOG_DEBUG_ONCE_NAMED` calls `RCUTILS_LOG_COND_NAMED`:

```c++
  # define RCUTILS_LOG_DEBUG_ONCE_NAMED(name, ...) \
  RCUTILS_LOG_COND_NAMED( \
    RCUTILS_LOG_SEVERITY_DEBUG, \
    RCUTILS_LOG_CONDITION_ONCE_BEFORE, RCUTILS_LOG_CONDITION_ONCE_AFTER, name, \
    __VA_ARGS__)
```

```c++
#define RCUTILS_LOG_COND_NAMED(severity, condition_before, condition_after, name, ...) \
  do { \
    RCUTILS_LOGGING_AUTOINIT; \
    static rcutils_log_location_t __rcutils_logging_location = {__func__, __FILE__, __LINE__}; \
    if (rcutils_logging_logger_is_enabled_for(name, severity)) { \
      condition_before \
      rcutils_log_internal(&__rcutils_logging_location, severity, name, __VA_ARGS__); \
      condition_after \
    } \
  } while (0)
```

with following conditions:

```c++
#define RCUTILS_LOG_CONDITION_ONCE_BEFORE \
  { \
    static int __rcutils_logging_once = 0; \
    if (RCUTILS_UNLIKELY(0 == __rcutils_logging_once)) { \
      __rcutils_logging_once = 1;

#define RCUTILS_LOG_CONDITION_ONCE_AFTER } \
}
```

The compiler combines those `#define` statements, resulting in the following expression: </br>
(the static assert was ommitted as it is a compile-time check)</br>


```c++
 do { 
    do { 
      RCUTILS_LOGGING_AUTOINIT; 
      static rcutils_log_location_t __rcutils_logging_location = {__func__, __FILE__, __LINE__}; 
      if (rcutils_logging_logger_is_enabled_for(name, severity)) { 
      { 
        static int __rcutils_logging_once = 0; 
        if (RCUTILS_UNLIKELY(0 == __rcutils_logging_once)) { 
          __rcutils_logging_once = 1;
          rcutils_log_internal(&__rcutils_logging_location, severity, name, __VA_ARGS__); 
        }
      }
    } while (0)
} while (0)
```

In other words: in _the scope of the macro call_, a static variable is defined, to track if _that particular_ macro call was already called or not. </br>
So this effectively results in each log message being logged once and only once by its macro call.
</br>
</br>
There's a specific reason for using the `do { .... } while (0)` construction, see the [reference in the code](https://github.com/ros2/rclcpp/blob/1a9b117d5357f4d37ecd457843ee27e8ba56b65c/rclcpp/resource/logging.hpp.em#L97-L99) for more info.
</br>
</br>

For realtime execution, the logging macros are disabled by setting the loglevels so that `rcutils_logging_logger_is_enabled_for(name, severity)` returns `false`.</br>
Hence, the conditions and `rcutils_log_internal()` function are not called.

</br>
</br>

### rcutils_logging_logger_is_enabled_for(name, severity)

</br>

- Calls `rcutils_logging_get_logger_effective_level` to get the 'effective  loglevel' for logger 'name',

- Returns true if `severity >= effective_loglevel`. 
  - `severity` is the severity of the macro call (e.g. RCLCPP_DEBUG),
  - `effective_loglevel` is the loglevel which was set for this logger name or its ancestors (see below), or the default loglevel if no loglevel was set.

</br>
</br>

### rcutils_logging_get_logger_effective_level(name)<a id="ancestorallocation"></a>

Looks up 'name' in the hash map:
- [If the hash map has no entries](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L866-L884): return `g_rcutils_logging_default_logger_level`

- Else it looks up if there is a [hash for 'name'](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L888-L892) or, if not, for the ['ancestors' of 'name'](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L904-L939) and:
  - If found:&emsp;returns the corresponding severity,
  - Else:&emsp;returns `g_rcutils_logging_default_logger_level`</br>
  </br>
- The 'ancestor' hierarchy is signified by logger names being separated by dots:
   - A logger named `x` is an ancestor of `x.y`,
   - Both `x` and `x.y` are ancestors of `x.y.z`, etc.</br>
   </br>
   
   <span style="color:red">**To search for ancestors, a copy of 'name' is made, allocated by g_rcutils_logging_allocator --> realtime-compliant allocator mandatory**</span></br>
_Idea for possible code change to avoid this allocation: add extra search and hash function that takes a parameter 'up_to_nth_delimiter', so instead of hashing until the end of the string (\0), the hashing is done until the specified number of occurences of `RCUTILS_LOGGING_SEPARATOR_CHAR` (i.e. '.') are encountered. This way no copy has to be made._

</br>
</br>

### rcutils_logging_set_logger_level(name, level)

Above hashmap is populated by calls to `rcutils_logging_set_logger_level`.</br>
Loglevels are typically specified through the command line arguments, which are processed in `rclcpp::init` (see [below](#rclcpp_init)).</br>

`rcutils_logging_set_logger_level` does the following:
- [Check](https://github.com/ros2/rcutils/blob/1db1f16a49378f4693b74296449ee920ff53cffe/src/logging.c#L978-L980) that the log level is in the range `[0, number of loglevels - 1]`.
- [Get](https://github.com/ros2/rcutils/blob/1db1f16a49378f4693b74296449ee920ff53cffe/src/logging.c#L985) the corresponding string,
- [Check](https://github.com/ros2/rcutils/blob/1db1f16a49378f4693b74296449ee920ff53cffe/src/logging.c#L993) if an entry exist in the hashmap for that logger name,
- If so: remove it, and iterate over the hash map to see if there are cached values for child loggers (i.e. not manually set) that also should be removed. Cached values are an [optimization that currently is disabled](https://github.com/ros2/rcutils/blob/1db1f16a49378f4693b74296449ee920ff53cffe/src/logging.c#L946-L959).
- Finally [add the new key](https://github.com/ros2/rcutils/blob/1db1f16a49378f4693b74296449ee920ff53cffe/src/logging.c#L1052-L1057) or change the [default loglevel](https://github.com/ros2/rcutils/blob/1db1f16a49378f4693b74296449ee920ff53cffe/src/logging.c#L1059-L1062).

</br>
</br>
</br>

### `rcutils_log_internal(location, severity, name, format, ... )`

For realtime execution, the logging macros are disabled by setting the loglevels so that `rcutils_logging_logger_is_enabled_for(name, severity)` returns `false`. Hence, the conditions and `rcutils_log_internal()` function are not called.</br>
However, realtime compliance of `rcutils_log_internal(location, severity, name, format, ... )` could still be relevant if somebody (yes, that's you!) decides to write a realtime-compliant output handler (e.g. a non-locking buffered one? So the realtime calls never block, and a non-realtime thread reads from the buffer and does the actual logging output.).

Anyway, `rcutils_log_internal(location, severity, name, format, ... )` [is called](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1113-L1121).

  - The `va_list`, `va_start` and `va_end` calls deal with converting the "`...`" part of the function call into a variable named "`args`" that can be passed on to the call of `vrcutils_log_internal`.</br>
  <span style="color:red">**Does va_start allocate?** How does it convert "..." into args?</span>
  - Then in [`vrcutils_log_internal`](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1083-L1097):
    - The [current time is retrieved](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1088) `rcutils_system_time_now()`. This [boils down to](https://github.com/ros2/rcutils/blob/c83c4c7b197e72527fc9e0b438f9f1e6225f70af/src/time_unix.c#L65) `clock_gettime()` which <span style="color:red">**I think is realtime-compliant?**</span>, and then
    - The output handler function pointer is dereferenced to [call the ouput handler](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1095) with the arguments.

</br>

For rclcpp logging, an output handler is set in `rclcpp::init()`, that outputs to `stderr`, a `logfile` and publishes to `/rosout`.</br>
`stderr` is handled by the [rcutils_logging_console_output_handler](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1256).

</br>
</br>
</br>

## Recap

It starts with an rclcpp macro call:

`RCLCPP_INFO(node->get_logger(), "Some log message");`

</br>

After some rcutils macro magic, this resolves to:
- A call of `RCUTILS_LOGGING_AUTOINIT` (which only does things when logging is not initialized yet),
- Followed by `rcutils_logging_logger_is_enabled_for(name, severity)`.
</br>

Here it stops if logging is disabled through the loglevel.

</br>

Otherwise a call is made to `rcutils_log_internal()`.


This basically passes on to `vrcutils_log_internal()`, which calls the ouput handler.

</br>

So let's now check the rclcpp initialization, as this sets the output handler.


</br>
</br>

## rclcpp logger initialization

Rclcpp logging is typically initialized from `rclcpp::init`. <a id="rclcpp_init"></a>

</br>

`rclcpp::init()` is implemented in [`rclcpp/utilities.cpp`](https://github.com/ros2/rclcpp/blob/33dae5d679751b603205008fcb31755986bcee1c/rclcpp/src/rclcpp/utilities.cpp#L33-L44) and basically calls:</br>
 &emsp;&emsp;`get_global_default_context()->init(argc, argv, init_options);`


`get_global_default_context()` [returns a shared_ptr](https://github.com/ros2/rclcpp/blob/33dae5d679751b603205008fcb31755986bcee1c/rclcpp/src/rclcpp/contexts/default_context.cpp#L22-L27) to a `DefaultContext`, which inherits from `rclcpp::Context`. 

The `Context` constructor is empty apart from zero-initialising some member variables.

Then, `Context::init()` is called. It has several functional blocks:

- [Initialize the context](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L203-L213):
  
  - It [locks](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L198) an `init_mutex_`, which is a `std::recursive_mutex`,
  - It [allocates](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L203) a new `rcl_context_t`,
  - It [calls](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L207) `rcl_get_zero_initialized_context()`,
  - It [calls](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L208) `rcl_init` on the context (description below), and
  - If all went well, it [stores](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L213) `context` and a corresponding destructor function `__delete_context()` into the `std::shared_ptr` called `rcl_context_`.
   </br>
- [Initialize the logging](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L215-L234)&ensp;(can be disabled through the `InitOptions`):
  - [Locks](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L216-L217) the global logging mutex (shared_prt to static `std::recursive_mutex` )
  - [If the reference count is zero](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L218-L219) (i.e. no other contexts initialized the logging yet), a [call is made](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L220-L223) to `rcl_logging_configure_with_output_handler`
    - With the `rcl_context_` global arguments,
    - With the [allocator](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L222) defined in the `init_options`,
    - With the `rclcpp_logging_output_handler`
  - For a further description, see below.
  </br>
- Then [it checks](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L239-L241) for unparsed arguments (there shouldn't be any):
  - If there are: [catch the exception](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L247-L257), clean up (`rcl_shutdown` etc) and rethrow,
  </br>
- Finally [save the `initoptions`](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L243) and [add the context](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L245-L246) as a weak pointer to the global `WeakContextsWrapper`.

</br>
</br>

**Regarding above call of `rcl_init`:**
- [It does some checks](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/init.c#L54-L69):
  - If `argc`and `argv` correspond (i.e. `argc` non-null `argv` values),
  - If the options and context are not null and the allocator is valid,

- **There's a [call to logging](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/init.c#L71-L73) before logging is initalized** ---> [bugreport](https://github.com/ros2/rcl/issues/1037)

- [This comment](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/init.c#L87) is interesting:</br>
  &emsp;`// use zero_allocate so the cleanup function will not try to clean up uninitialized parts later`</br>
  I assume this is part of the rationale for the "two-step initialization" procedure that is often used:
    - First get a zero-initialized _something_,
    - Then initialize the _something_,
    - If initialization fails, then everything which is still zero should not be cleaned up.
    </br>
    </br>
- It [calls](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/init.c#L99) `rcl_init_options_copy`
  - This seems to be RMW-related, as it [resolves to](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/init_options.c#L106) a call of `rmw_init_options_copy`.
  </br>
  </br>
- `argc` and `argv` are [copied into the context](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/init.c#L105-L124)
- The command line arguments [are parsed](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/init.c#L127):
  - Do some [error checks](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/arguments.c#L254-L266)
  - [Allocate](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/arguments.c#L271) an `rcl_arguments_t`
    - It does not have `zero` in the name (`_rcl_allocate_initialized_arguments_impl`) this time, but it is also zero-initialized.
    - [Here](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/arguments.c#L2055-L2070) is the complete list of members. Wrt logging, following ar relevant:
      - `args_impl->log_levels`
      - `args_impl->external_log_config_file`
      - `args_impl->log_stdout_disabled`
      - `args_impl->log_rosout_disabled`
      - `args_impl->log_ext_lib_disabled`
      </br></br>
  - [Allocate the members](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/arguments.c#L282-L315) of the `args_impl` of the `rcl_arguments_t`.</br>
  It states in the comment that it is "over-allocating".</br>
  Indeed, each allocation is done for `argc` values (i.e. the maximum possible amount). After parsing, the non-needed allocations are [deallocated](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/arguments.c#L596-L667)</br>
  Wrt logging: the [call to](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/arguments.c#L311) `rcl_log_levels_init`&ensp;[allocates](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/log_level.c#L52-L57) memory for `rcl_logger_setting_t`.
  </br></br>
  - Then it starts [parsing](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/arguments.c#L331-L594). The definition of the known command-line arguments is [here](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/include/rcl/arguments.h#L41-L84).</br>
  &emsp;&emsp;(**There's several more calls to logging here, though logging still is uninitalized** --> [bugreport](https://github.com/ros2/rcl/issues/1037))</br>
  Amongst others, it parses:
    - Command-line [loglevel arguments](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/arguments.c#L420-L444),
    - Command-line specified [logger file](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/arguments.c#L446-L476)</br>
      --> This simply reads the file: there are currently no checks in the [code](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/arguments.c#L1976-L1992)
      </br></br>
- The rest of `rcl_init` mainly seems to setup RMW.


</br>
</br>

**After `rcl_init` the call to `rcl_logging_configure_with_output_handler` is made to initialize the logging:**


- It starts with a [call to](https://github.com/ros2/rcl/blob/bc979c6f22d6af54ad473cd65cd81708a4da8a93/rcl/src/rcl/logging.c#L65) `RCUTILS_LOGGING_AUTOINIT`,
  - This is strange, as `RCUTILS_LOGGING_AUTOINIT` uses the standard allocator.
  - Filed a [bug report](https://github.com/ros2/rcl/issues/1036)
  </br></br>
- Then, the parsed command-line arguments from `rcl_init()` are taken and corresponding [loglevels are set](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/logging.c#L68-L90)<a id="loglevels_set"></a>
</br></br>
- Following three output handlers [are initialized](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/logging.c#L91-L113) and [added](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/logging.c#L91-L113) (unless disabled through the command-line arguments):
  1. `rcutils_logging_console_output_handler`,
  2. `rcl_logging_rosout_output_handler`
  3. `rcl_logging_ext_lib_output_handler`
</br></br>
- The log config file is passed solely to the [external logger](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/logging.c#L103).
</br></br>
- Finally `rcutils_logging_set_output_handler` is called, so the `rclcpp_logging_multiple_output_handler` is used.
</br>
</br>
</br>

The `rclcpp_logging_multiple_output_handler`:

- [Locks](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L131-L133) the rclcpp global logging mutex, and
- [Calls](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L134-L135) `rcl_logging_multiple_output_handler`, which simply [calls](https://github.com/ros2/rcl/blob/bc979c6f22d6af54ad473cd65cd81708a4da8a93/rcl/src/rcl/logging.c#L157-L161) each of the three  output handlers consecutively.

</br>
</br>
</br>


## rcl output handlers

</br>

As mentioned, rcl installs three output handlers, console, rosout and external library:


</br>

### rcutils_logging_console_output_handler

</br>

The `rcutils_logging_console_output_handler` does the following:

- [Check](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1264) for initialized,
- [Check](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1270-L1281) if severity is correct, else return,
- [Check](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1283-L1289) if output is colorized,
- [Initialize](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1291-L1307) `msg_array` and `output_array`
  - Initial buffer size is 1024 characters (including terminating `\0`)&emsp; <span style="color:red">**Fixed-size array in stack memory**</span>
  - Both use the `g_rcutils_logging_allocator` if expansion beyond 1024 characters is needed.&emsp;  <span style="color:red">**Can be realtime-compliant if realtime-compliant allocator is used, or stay below 1024 characters.**</span></br>
  </br>

- [Call](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1309-L1311) to `SET_OUTPUT_COLOR_WITH_SEVERITY(status, severity, color)`.</br>
This calls [SET_COLOR_WITH_SEVERITY](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1241-L1246) and [SET_OUTPUT_COLOR_WITH_COLOR](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1241-L1246).

`SET_COLOR_WITH_SEVERITY` resolves to:
  ```c++
    { 
    const char * color = NULL;                           // color is char* in stack memory
      { 
        switch (severity) { 
          case RCUTILS_LOG_SEVERITY_DEBUG:
            color = "\033[32m";                          // color points to static read-only string  --> ok
            break;
          case RCUTILS_LOG_SEVERITY_INFO:
            color = "\033[0m";
            break;
          
          /* rest of code ommitted for clarity */

        }
      }
  ```

And `SET_OUTPUT_COLOR_WITH_COLOR(status, color, output_array)` does [the following](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1230-L1240):
- Calls `rcutils_char_array_strncat`
  - This appends the color string to `output_array`,
  - Will not allocate as `output_array` is still empty and its capacity (1024 - 1) is larger than the color string that is copied into it.
</br>
</br>

- If all above went ok, `args` is cloned into a new `va_list` and passed on to `rcutils_char_array_vsprintf`, see [this code](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1313-L1322)</br>
<span style="color:red">**Does va_copy allocate? It's a copy?**</span>

- `rcutils_char_array_vsprintf`: <a id="rcutils_vspintf"></a>
  - Calls `vsnprintf` from `<stdio.h>`. This converts a variable argument list `va_list` into a formatted string in a sized buffer. <span style="color:red">**Does this allocate?**</span>
  - If buffer is too small, `vsnprintf` returns the needed buffer size to fit.
  - If needed, `msg_array` is resized accordingly and `vsnprintf` called a 2nd time. &emsp;  <span style="color:red">**Can be realtime-compliant if realtime-compliant allocator is used, or stay below 1024 characters.**</span></br>
  </br>

- If all is ok, `rcutils_logging_format_message` [is called](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1324-L1331):
  - This [populates](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1128-L1134) a `logging_input_t` data member and
  - Iterates accross the formatting handlers that were set by [parse_and_create_handlers_list()](#parse_and_create_handlers_list)
    - Recall: these handlers populate `output_array` according to the formatspecification (e.g. `"[{severity}] [{time}] [{name}]: {message}"`)
    - Similar to above, `output_array` is resized as needed. &emsp;  <span style="color:red">**Can be realtime-compliant if realtime-compliant allocator is used, or stay below 1024 characters.**</span></br>
    </br>

- Then `SET_STANDARD_COLOR_IN_BUFFER` [is called](https://github.com/ros2/rcutils/blob/61018b2f88e55ac81edda4a45a02634493c999ed/src/logging.c#L1334) which appends "\033[0m" to the end of the buffer to reset the color output, 

- and finally ...

- An `fprintf` call is made to output the `output_array`:
```c
     if (RCUTILS_RET_OK == status) {
       fprintf(g_output_stream, "%s\n", output_array.buffer);
     }
``` 


</br>
</br>

### rosout

This publishes log info to topic /rosout.</br>
Configuration of this output handler is spread accross a few steps:
- A call to `rcl_logging_rosout_init` before the handler is registered,
- Some logic upon creation of a node,
- Some logic upon call of `Logger::get_child()`
</br>
</br>

**Upon `rcl_logging_rosout_init`:**
  - This [initializes two hashmaps](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L89-L119):  `__logger_map` and `__sublogger_map`.</br>
    - Hashmap allocator is the one which was passed to `rcl_logging_configure_with_output_handler` (i.e. the one defined in the `InitOptions`, see [here](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L222)).
  - `__logger_map` links full logger names (e.g. `"my_logger"` or `"other_logger.child.grandchild"`) to `rosout_map_entry_t` instances, which hold a reference to the corresponding node publisher.
  - `__sublogger_map` links full names to a reference counter, to track when cleanup is needed.

</br>


**Upon creation of a node:**

- On creation of a node, a [NodeBase is instantiated](https://github.com/ros2/rclcpp/blob/432bf21f0261bab209b37ccfe6da550f02751a22/rclcpp/src/rclcpp/node.cpp#L162), which:
  - [Locks](https://github.com/ros2/rclcpp/blob/33dae5d679751b603205008fcb31755986bcee1c/rclcpp/src/rclcpp/node_interfaces/node_base.cpp#L54-L58) the global logging mutex, and
  - [Calls](https://github.com/ros2/rclcpp/blob/33dae5d679751b603205008fcb31755986bcee1c/rclcpp/src/rclcpp/node_interfaces/node_base.cpp#L63-L67) `rcl_node_init`.
  </br></br>
- This does a lot of things, and then [calls](https://github.com/ros2/rcl/blob/4eccc3cbe65f5b58981f22efaf612a65731cb2f0/rcl/src/rcl/node.c#L287-L294) `rcl_logging_rosout_init_publisher_for_node`, which:
  - [Checks the hash map](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L243-L261)
    - If an entry already exists for this name (due to another node with the same name), a warning is issued.</br>
    (This is another rcutils log macro call, but this is ok here as logging should have be enabled by now.)
    </br></br>
  - [Creates a publisher](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L263-L275)
    - This [allocates](https://github.com/ros2/rcl/blob/60d80648c2a7cdbadcc7f22bdbc3c358c04dff7c/rcl/src/rcl/publisher.c#L65) using the allocator defined in the `rcl_publisher_options_t`.
    - The provided publisher options are the [default ones](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L266), i.e. `malloc`. [--> bugreport](https://github.com/ros2/rcl/issues/1040)
    - Note following [TODO](https://github.com/ros2/rcl/blob/60d80648c2a7cdbadcc7f22bdbc3c358c04dff7c/rcl/src/rcl/publisher.c#L109) in the code:</br>
    `  // TODO(wjwwood): pass along the allocator to rmw when it supports it`</br>
    I did not further look into this. Relevant [link](https://github.com/ros2/rmw/issues/159)?
    </br></br>

  - [Adds it](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L277-L289) to the `__logger_map`
</br></br>

Note that this is the only place where rosout publishers are created.</br>
This implies that only log messages associated to a node logger are published to `/rosout`.</br>
Log messages from log macro calls with a non-node logger (i.e. a "named logger", e.g. `rclcpp::get_logger("some_name")`) are **not** published to `/rosout`.

</br></br>

**Upon call of `Logger::get_child()`:**

- [Locks](https://github.com/ros2/rclcpp/blob/399f4df7396d67b42ccaea651bbd87d66b0d62b3/rclcpp/src/rclcpp/logger.cpp#L78-L81) the global logging mutex

- [Configures](https://github.com/ros2/rclcpp/blob/399f4df7396d67b42ccaea651bbd87d66b0d62b3/rclcpp/src/rclcpp/logger.cpp#L82) the subnode publishing by a call to `rcl_logging_rosout_add_sublogger`:
  - This first [gets the full name](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L434) (i.e. "logger" + "child" --> "logger.child").
    - This [allocates memory](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L408-L410) using `__rosout_allocator` (which is the allocator given to `rcl_logging_configure_with_output_handler()`, i.e. the one defined in the `InitOptions`, see [here](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L222)).
    </br></br>

  - Then [checks if there is an entry](https://github.com/ros2/rcl/blob/12e32fefdfc628224087cfd250760a2d3d57981e/rcl/src/rcl/logging_rosout.c#L435) for the parent node in the `__logger_map`,
    - If not, [throws an error](https://github.com/ros2/rcl/blob/12e32fefdfc628224087cfd250760a2d3d57981e/rcl/src/rcl/logging_rosout.c#L436-L440)
      - Note that this implies that, if the rosout publisher is enabled, a call to `get_child()` on a non-node logger (i.e. a "named logger") will **throw an exception**.
      - This **terminates your program**.
      </br></br>
    - Else:
      - If there is already a key in the `__logger_map` for the full name ("parent.child"):
        - [Augment the reference counter](https://github.com/ros2/rcl/blob/12e32fefdfc628224087cfd250760a2d3d57981e/rcl/src/rcl/logging_rosout.c#L442-L453) in the `__sublogger_map`,</br>
      - Else:
        - [Add](https://github.com/ros2/rcl/blob/12e32fefdfc628224087cfd250760a2d3d57981e/rcl/src/rcl/logging_rosout.c#L456-L461) a new entry in `__logger_map` for the sublogger full name, with the same `entry` as the parent (i.e. using the publisher of the parent),
        - [Allocate and add](https://github.com/ros2/rcl/blob/12e32fefdfc628224087cfd250760a2d3d57981e/rcl/src/rcl/logging_rosout.c#L463-L474) a new sublogger entry to `__sublogger_map`.
        </br></br>
- Finally [return, or cleanup](https://github.com/ros2/rcl/blob/12e32fefdfc628224087cfd250760a2d3d57981e/rcl/src/rcl/logging_rosout.c#L487-L493) if something went wrong along the way.
  

</br>
</br>

**The output handler itself:**</br>
The `rcl_logging_rosout_output_handler` has a similar functionality as the console output handler:

- Get node/publisher info from the hashmap for that logger name - [code](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L340)
  - The `if` clause spans the whole output handler function body, so if no hash is found (i.e. for a named logger) it just returns. 
  </br></br>
- Initialize an empty message buffer and call `rcutils_char_array_vsprintf` - [code](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L342-L356)
  - For the description of `rcutils_char_array_vsprintf`, see [above description](#rcutils_vspintf) of the console output handler.
  </br></br>
- Then it [creates](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L363) a message by calling `rcl_interfaces__msg__Log__create()`.</br>
Afaik, the code implementing this function is generated upon build time from the [Log.msg interface](https://github.com/ros2/rcl_interfaces/blob/rolling/rcl_interfaces/msg/Log.msg), yielding following source file:</br>
&emsp;&emsp;`build/rcl_interfaces/rosidl_generator_c/rcl_interfaces/msg/detail/log__functions.c`</br>
which allocates using the **default allocator** and initializes the message:
```c
rcl_interfaces__msg__Log *
rcl_interfaces__msg__Log__create()
{
  rcutils_allocator_t allocator = rcutils_get_default_allocator();
  rcl_interfaces__msg__Log * msg = (rcl_interfaces__msg__Log *)allocator.allocate(sizeof(rcl_interfaces__msg__Log), allocator.state);
  if (!msg) {
    return NULL;
  }
  memset(msg, 0, sizeof(rcl_interfaces__msg__Log));
  bool success = rcl_interfaces__msg__Log__init(msg);
  if (!success) {
    allocator.deallocate(msg, allocator.state);
    return NULL;
  }
  return msg;
}
```
</br>

- Then, the message is [populated](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L365-L372) and [published](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L373)
  - The publish call resorts to a call of [rwm_publish](https://github.com/ros2/rcl/blob/60d80648c2a7cdbadcc7f22bdbc3c358c04dff7c/rcl/src/rcl/publisher.c#L262),
    - This code path was not further studied for this document.
  - Note that the last argument is `NULL`(no pre-allocted memory to be used).
  </br></br>
- Finally the message is [finalized and deallocated](https://github.com/ros2/rcl/blob/d15594effa63065a19a9f69960ea80f5ac5be8bd/rcl/src/rcl/logging_rosout.c#L381):
```c
void
rcl_interfaces__msg__Log__destroy(rcl_interfaces__msg__Log * msg)
{
  rcutils_allocator_t allocator = rcutils_get_default_allocator();
  if (msg) {
    rcl_interfaces__msg__Log__fini(msg);
  }
  allocator.deallocate(msg, allocator.state);
}
```

  


</br>
</br>

### External library (spdlog)

An interface for external logging libraries is defined in `rcl_logging`.</br>

This declares [following functions](https://github.com/ros2/rcl_logging/blob/rolling/rcl_logging_interface/include/rcl_logging_interface/rcl_logging_interface.h), to be implemented by the library:
- `rcl_logging_external_initialize`
- `rcl_logging_external_set_logger_level`
- `rcl_logging_external_log`
- `rcl_logging_external_shutdown`

And following function, [implemented](https://github.com/ros2/rcl_logging/blob/cdef749a304f734ba275f8466a4883db6400dc75/rcl_logging_interface/src/logging_dir.c#L25) by the interface itself:
- `rcl_logging_get_logging_directory`</br>
This returns the full path to the logging directory. It is:
  - Either the value of environment variable `ROS_LOG_DIR`,
  - Or the default of `$ROS_HOME/log`,
    - `$ROS_HOME` is another environment variable that can be set, or defaults to `$HOME/.ros`.
  
</br>

Currently, there are [two implementations](https://github.com/ros2/rcl_logging):
- `rcl_logging_spdlog`, using the `spdlog` library,
- `rcl_logging_noop`, which is an empty implementation (does nothing).
<br>

For reference: there used to be also a `log4cxx` implementation, but that was [removed](https://github.com/ros2/rcl_logging/pull/78) at some point.

</br>

I did not check in detail how to select a specific external library implementation.</br>
This [Discourse post](https://discourse.ros.org/t/ros2-logging/6469/34) states the following:

> The external logging library is statically linked when rcl is compiled, but I believe that the rcl library is dynamically linked during runtime. That should mean that you only need to recompile rcl_logging_spdlog and rcl and make sure they’re sourced as an overlay to the base ROS install. 

So I assume it is a compile-time option.

Note that [this anwer](https://discourse.ros.org/t/ros2-logging/6469/36) to above post states that "overlaying didn’t work for me reliably". So ymmv...


</br></br>

`rcl_logging_external_initialize`
- This first locks a mutex
  - It is strange that this implementation uses a mutex at this level, whereas the others don't.
  - In any case, in the context of rclcpp logging, initialization calls are serialized anyway by the rclcpp mutexes, so an extra mutex is not necessary.

</br>

- It [errors](https://github.com/ros2/rcl_logging/blob/cdef749a304f734ba275f8466a4883db6400dc75/rcl_logging_spdlog/src/rcl_logging_spdlog.cpp#L106-L111) if a configuration file was provided.


- It [reads](https://github.com/ros2/rcl_logging/blob/cdef749a304f734ba275f8466a4883db6400dc75/rcl_logging_spdlog/src/rcl_logging_spdlog.cpp#L115-L121) environment variable `RCL_LOGGING_SPDLOG_EXPERIMENTAL_OLD_FLUSHING_BEHAVIOR` which probably no one still knows about ;-),

- It [gets](https://github.com/ros2/rcl_logging/blob/cdef749a304f734ba275f8466a4883db6400dc75/rcl_logging_spdlog/src/rcl_logging_spdlog.cpp#L127) the path to the log directory (default `$HOME/.ros/log/`) and [creates](https://github.com/ros2/rcl_logging/blob/cdef749a304f734ba275f8466a4883db6400dc75/rcl_logging_spdlog/src/rcl_logging_spdlog.cpp#L137) the directory if needed.

- It [retrieves and concatenates](https://github.com/ros2/rcl_logging/blob/cdef749a304f734ba275f8466a4883db6400dc75/rcl_logging_spdlog/src/rcl_logging_spdlog.cpp#L143-L175) all parts of the log file name,

- Then it [creates and registers](https://github.com/ros2/rcl_logging/blob/cdef749a304f734ba275f8466a4883db6400dc75/rcl_logging_spdlog/src/rcl_logging_spdlog.cpp#L177-L191) a spdlog sink and logger, configured to flush every 5 seconds and on error.

</br></br>

**The other interface functions**

Refer to their [implementation](https://github.com/ros2/rcl_logging/blob/cdef749a304f734ba275f8466a4883db6400dc75/rcl_logging_spdlog/src/rcl_logging_spdlog.cpp#L197-L217).


</br></br>

**The output handler**

The `rcl_logging_ext_lib_output_handler` is very similar to the console output handler:
- It also [calls](https://github.com/ros2/rcl/blob/12e32fefdfc628224087cfd250760a2d3d57981e/rcl/src/rcl/logging.c#L193-L196) `rcutils_char_array_vsprintf` and `rcutils_logging_format_message` to populate the message string,
- And then is [passes](https://github.com/ros2/rcl/blob/12e32fefdfc628224087cfd250760a2d3d57981e/rcl/src/rcl/logging.c#L210) this to `rcl_logging_external_log()`.

</br></br></br>


## Conclusion

This concludes this study of the rcl / rcutils / rclcpp logging implementation.

There only parts (I think) that are not covered here, are the cleanup routines called from [`context::shutdown`](https://github.com/ros2/rclcpp/blob/1fd5a96561166ad2ad557bbc1a297cd652883cb2/rclcpp/src/rclcpp/context.cpp#L339-L353).</br>
However, if you have come this far, I think you're capable to study these on your own. ;-)
</br></br>
Thanks for reading, I hope this document is useful to you.
</br></br>
Now back to the robots.

