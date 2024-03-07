# Collection of code tips for Python that will be useful one day on my projects and working-related stuff

## Name mangling

This is a very confusing feature that is need to know, so we can't make mistakes in the future, I learned from [this](https://youtu.be/0hrEaA3N3lk) YouTube video and now I'll try to explain what is important for me.

When you create an underscore variable inside a function, at compilation time, python changes its name to be `_{ClassName}__{VariableName}` for example...

```python
class A:
    __count = 0 # Python will change internally the name to _A__count

    def get_count(self):
        return __count
```

That might cause some confusion because when I try to access the variable from outside, like `A.__count`, I'll not be accessing the same variable I defined inside the class.

Nevertheless, It is a useful feature because if I define a variable with the same name as one variable on the parent class, the values will not be messed up, leaving behind the need to know the implementation of the parent class. Nevertheless, it might cause some confusion. Use it carefully.

## Variable Scopes and Closures

Variable Scopes define whether to look for a variable in Python, closures are the environment where the function is utilized to store data about its values, more on that in [this video](https://youtu.be/jXugs4B3lwU). In summary, we have the following statement:

> Lookup happens at runtime, location is decided at compile time.

What does it mean? When you define a function and use a variable, it decides if it will look at local or global scopes at compile time, but, the actual value is looked at runtime. For example.

```python
x = 10
def function():
    print(x)

 x = 20

function()  # Gives error
```

The above code will return an error because `X` is found locally at compiled time, but at runtime, `x` is printed before it is referenced.

## Concurrencies and Paralelism

...

### Multiprocessing

<https://youtu.be/X7vBbelRXn0>

### Concurrencies

<https://youtu.be/GpqAQxH1Afc>

### Threading

...
