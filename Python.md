input() is used for taking user input. It always returns a string, you'll need to typecast for other data types. For example, need to use int for age.

age = int(input("Enter your age : "))
print(age)

Python doesn't have static, const keyword

Exceptions in abc.
Garbage Collection in Python 

x = 10 
y = 10 .... Both are same object
z = [10,10]

reference count of 10 increased by 4.

In python each object stores 3 things, reference count, value, type

### Garbage Collection in Python
There are two ways of garbage collection in python.
1. Reference Count
     As soon as reference count of an object reaches zero, it's memory is claimed.
     When we use del keyword for an object reference, then it decreases the reference count to that object, and doesn't delete the object itself. It's GC that actually deletes the object
     Reference count is not thread safe.
2. Tracing
As soon as reference count of an object reache