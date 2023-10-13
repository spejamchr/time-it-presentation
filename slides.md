---
theme: seriph
background: https://source.unsplash.com/collection/94734566/1920x1080
class: text-center
highlighter: shiki
lineNumbers: false
info: A lightning talk on benchmarking
drawings:
  persist: true
title: TimeIt
mdc: true
fonts:
  sans: 'Arial'
  serif: 'Robot Slab'
  mono: 'Fira Code'
transition: slide-up
---

# TimeIt

A benchmarking story

---

# Why?

Benchmarking should have a reason

Our `EnterprisePortal::UseCasePersistenceService#save` was timing out, and [@salsa] wanted to know
which part exactly was the slow bit.

[@salsa]: https://execonline.slack.com/team/U010TE2D1R9

---
transition: slide-left
---

# First Idea

Write some code to time all the functions real quick

<v-clicks>

- I can store of all the times in an instance variable
- I can write a simple method that captures the body of a function in a block
- The capturing method runs the block and stores the time to my list of times

</v-clicks>

---
transition: slide-left
---

# Block Capture 

Implementation (directly  in the class)

```ruby {3,6-12,15-17}
def initialize(arguments)
  # ... Skipping some stuff
  @benchmarks = []                                      # Store the data here.
end

def timeit(method)                                      # Use this method to time other methods.
  a = Time.now
  result = yield                                        # Call `yield` to call the passed block.
  b = Time.now
  @benchmarks << { method => b - a }                    # Store the timing data.
  result                                                # Return the original result.
end
```

---
transition: slide-left
---

# Block Capture 

Usage

```ruby {2-5}
def save
  timeit :save do                                       # Call `#timeit` inside the method, and pass it a block.
    # Actual method here                                # The block contains the original method's definition.
  end
end
```

---

# Block Capture

Thoughts

<v-click>

Pros:

- Easy to implement the `timeit` method

</v-click>
<v-click>

Cons:

- If the method has an early return, the `return` needs to be replaced with `next` so that it
  behaves as expected inside the block.

```ruby {3,5,6}
def validate_permission_type_change(params)
  timeit __method__ do
    next if new_record                                                          # This
    permission_type_param = params.fetch(:permission_type)
    next if use_case.permission_type == permission_type_param                   # was
    next if user.can_update_permission_type?(use_case)                          # annoying.
    use_case.errors.add(:permission_type, "Permission type cannot be changed")
  end
end
```

</v-click>

---
transition: slide-left
---

# Second Idea

Use a decorator like `Memorb`, which lets us memoize a method by prefixing it with `memoize`

<v-clicks>

- I can still store of all the times in an instance variable
- I can write a decorating method that aliases the original method
- The decorator method runs the original method and stores the time in my list of times

</v-clicks>

---
transition: slide-left
---

# Decorator

Implementation

```ruby {3,6-17}
def initialize(arguments)
  # ... Skipping some stuff
  @benchmarks = []                                      # Continue storing the data here.
end

def self.timeit(method)                                 # Now this is a class method.
  aliased = "aliased_#{method}".to_sym                  # Make an alias version of the method 
  alias_method aliased, method                          # (like a backup of the original).

  define_method(method) do |*args, &block|              # Redefine the given method.
    a = Time.now
    result = send(aliased, *args, &block)               # Time the alias/backup version with `#send`.
    b = Time.now
    @benchmarks << { method => b - a }                  # Store the timing data.
    result                                              # Return the original result.
  end
end
```

---
transition: slide-left
---

# Decorator

Usage

```ruby
timeit def save                                         # Call `timeit` outside the method now.
  # Actual method here                                  # The inside of the method stays the same.
end
```

---
transition: slide-left
---

# Decorator

Thoughts

<v-click>

Pros

- Don't have to change the method body
- Don't have to write the name of the method I'm timing

</v-click>
<v-click>

Cons

- Figuring out the method aliasing was confusing
- I wanted to time *all* the methods in the class, and prefixing each one was annoying.

</v-click>

---

# Decorator

Results

This second implementation was plenty to get the results I needed. The slow method, `#save`, took
about 13.5s in my test case, and the timing code showed that the particularly slow portion was
`#assign_programs_and_extensions` (13.4s) and within that `#assign_programs` (10.3s).

<v-click>

But I still wanted a more ergonomic timing solution...

</v-click>

---
transition: none
---

# Third Idea

Make a `TimeIt` module that times *all* methods just by including it in a class

<v-clicks>

- When the module is included I can list all the class's instance methods & decorate them
- How to run code when a module is included?
- Use the `included` hook
- But at that point I only have access to methods already defined...

</v-clicks>

<v-after>

```ruby {3,7,9}
class Demo

  def first_method                                      # TimeIt#included can see this
    # stuff
  end

  include TimeIt

  def second_method                                     # TimeIt#included can't see this :(
    # other stuff
  end
end
```

</v-after>

---
transition: slide-left
---

# Third Idea

Make a `TimeIt` module that times *all* methods just by including it in a class

- When the module is included I can list all the class's instance methods & decorate them
- How to run code when a module is included?
- Use the `included` hook
- But at that point I only have access to methods already defined...
- Can I run code when a new method on the class is defined?

<v-clicks>

- Yes, Ruby has a `method_added` hook
- But my decorator defines method aliases, which will trigger `method_added` and recursively call my
  decorator...
- So I need to keep track of which methods I've decorated
- If I'm decorating *all* methods, how do I handle `#send`? (My decorator has been using it to  
  call the aliased original method.)
- I can use Ruby's `::Method` class to get an unbound `#send` method and bind it to the  
  decorated object

</v-clicks>


---
transition: slide-left
---

# Module

Implementation (highlights)

- `#included`
```ruby
def included(base)
  Helpers.decoratable_methods(base).each do |method|    # All instance methods on the base
    Helpers.decorate_as_timed(base, method) }           # Implementation on next slide...
  end
end
```

<v-click>

- `#method_added`

```ruby
def method_added(method)                                # Runs any time a new method is defined.
  super                                                 # Be polite. Call `super` when appropriate.
  Helpers.decorate_as_timed(self, method)               # Reusing this logic here.
end
```

</v-click>

---
transition: slide-left
---

# Module

Implementation (more highlights)

- `#decorate_as_timed`

```ruby
def decorate_as_timed(base, method)
  return if already_timed?(base, method)                # Prevent re-decorating methods...

  record_as_timed(base, method)                         # ...by keeping a list.

  aliased = [PREFIX, method].join.to_sym                # Create a method alias
  base.alias_method aliased, method                     # of the original method
  base.__send__(:private, aliased)                      # and make it private.
  define_decorated_method(base, method, aliased)        # Implementation on next slide...
end
```

---
transition: slide-left
---

# Module

Implementation (last highlights)

- `#define_decorated_method`

```ruby
def define_decorated_method(base, method, aliased)
  # Don't call any methods on `self` while defining the decorated method,
  # including `#__send__`, since they could be decorated and cause a
  # recursive stack overflow. Use this `unbound_send` object to call the
  # original `#__send__` method.
  unbound_send = ::BasicObject.method(:__send__).unbind

  base.define_method(method) do |*args, &block|         # Define the decorated method.
    bound_send = unbound_send.bind(self)                # Bind our `#__send__` method.
    data = Helpers.timer(method) do                     # Use a `timer` helper to time the original
      bound_send.call(aliased, *args, &block)           # method and give us helpful metadata.
    end
    (@__timings ||= []) << data.fetch(:timing)          # Store the timing metadata.
    data.fetch(:result)                                 # Return the original result of the method.
  end
end
```

---
transition: slide-left
---

# Module

Implementation (not shown)

Several helper methods for doing little jobs:

- Get timing info when running some block
- Get all instance methods of a class
- Keep track of which methods have already been decorated

---
transition: slide-left
---

# Module

Usage

```ruby {2}
class UseCasePersistenceService
  include TimeIt                                        # So simple!

  # The rest of the class...
end
```

---

# Module

Thoughts

<v-click>

Pros:

- Easy to use
- Fairly easy to extend the code
- Fun to implement

</v-click>
<v-click>

Cons:

- Difficult to implement
- More complex, so more prone to bugs

</v-click>

---

# Notes

- [@ecopony] came to the same timing conclusions and fixed the slowdown in the
  `UseCasePersistenceService` in [#7755]. Tanks!

[@ecopony]: https://execonline.slack.com/team/U0BRFK73P

[#7755]: https://github.com/execonline-inc/exec_online/pull/7755
