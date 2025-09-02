---
tags:

- python
- tips
- advanced
- programming

---

# Python Tips & Tricks

A collection of Python code tips and tricks that will be useful for projects and work-related development.

## Name Mangling

This is a very confusing feature that is important to understand to avoid mistakes in the future. I learned from [this YouTube video](https://youtu.be/0hrEaA3N3lk) and here's what's important to know.

When you create a double underscore variable inside a class, at compilation time, Python changes its name to be `_{ClassName}__{VariableName}`. For example:

```python
class A:
    __count = 0  # Python will change internally the name to _A__count

    def get_count(self):
        return __count
```

This might cause confusion because when trying to access the variable from outside, like `A.__count`, you'll not be accessing the same variable defined inside the class.

Nevertheless, it's a useful feature because if you define a variable with the same name as one in the parent class, the values will not be mixed up, removing the need to know the implementation details of the parent class. However, it might cause some confusion. Use it carefully.

## Variable Scopes and Closures

Variable scopes define where to look for a variable in Python. Closures are the environment where functions store data about their values. More details in [this video](https://youtu.be/jXugs4B3lwU).

The key principle is:

> Lookup happens at runtime, location is decided at compile time.

What does this mean? When you define a function and use a variable, Python decides whether to look at local or global scopes at compile time, but the actual value lookup happens at runtime. For example:

```python
x = 10

def function():
    print(x)
    x = 20  # This makes x local to the function

function()  # Gives UnboundLocalError
```

The above code will return an error because `x` is found to be local at compile time, but at runtime, `x` is accessed before it's assigned locally.

## Concurrency and Parallelism

Understanding the difference between concurrency and parallelism is crucial for writing efficient Python programs.

### Multiprocessing

Learn more about multiprocessing in Python: [https://youtu.be/X7vBbelRXn0](https://youtu.be/X7vBbelRXn0)

### Concurrency

Understanding concurrency concepts: [https://youtu.be/GpqAQxH1Afc](https://youtu.be/GpqAQxH1Afc)

### Threading

Threading in Python allows for concurrent execution of code, though with limitations due to the Global Interpreter Lock (GIL).
