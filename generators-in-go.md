# Generators in Go

__Sven Haardiek, 2020-07-24__

Lately I played a lot with [Haskell](https://www.haskell.org/) and I started to
love [lazy evaluation](https://www.haskell.org/) as it let me define infinite
sequences without the need of infinite memory.

But at work, I mostly write programs using [Python](https://www.python.org/) and
[Go](https://golang.org/).

Python's nearest thing to lazy evaluation are
[Iterators](https://docs.python.org/3/tutorial/classes.html#iterators) and
especially
[Generators](https://docs.python.org/3/tutorial/classes.html#generators) in my
opinion and they are great.

You can simply write a function and `yield` the next generated value on in each
call, like shown [here](https://en.wikipedia.org/wiki/Iterator#Generators)

```python
def fibonacci(limit):
    a, b = 0, 1
    for _ in range(limit):
        yield a
        a, b = b, a+b

for number in fibonacci(100):  # The generator constructs an iterator
    print(number)
```

So now I thought: *Hej, what about Go? How should I write generators there?* So
I tried some different approaches to write generators for [Fibonacci
numbers](https://en.wikipedia.org/wiki/Fibonacci) and I want to share them
here.

## Native Approach

So let's dive directly into it. What do we need for an Generator? We need a
struct to store our current state in and we need a function which returns the
next Fibonacci value. Okay that should be pretty simple. Let's take a look.

```golang
type fibonacci struct {
	a, b int
}

func (f *fibonacci) Next() int {
	r := f.a
	f.a, f.b = f.a+f.b, f.a
	return r
}
```

Okay, so now our struct is holding the last two values, which are needed to
calculate the next Fibonacci number. Also `Next` should return the next
Fibonacci number on every call. Let's check, if it working with the following
code.

```golang
func main() {
	f := fibonacci{0, 1}
	for i := 0; i < 20; i++ {
		fmt.Printf("%v ", f.Next())
	}
}
```

If we now run this code, we get `0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610
987 1597 2584 4181` which are the same Fibonacci numbers as on
[Wikipedia](https://en.wikipedia.org/wiki/Fibonacci_number#Sequence_properties),
so I guess it is correct.

Our code does look nearly the same than the Python code we posted early, but we
had to cache the result, since we not `yield`, but call the function from the
beginning again. Also there is a limit in the Python code. We want to have that
too. So let's write it.

Our struct is now also holding a `limit` and we have to change the return value
of our `Next` function, since we need to have an indicator when there are no
values anymore.

```golang
type fibonacci struct {
	a, b, limit int
}

func (f *fibonacci) Next() *int {
	if f.limit == 0 {
		return nil
	}
	r := f.a
	f.a, f.b = f.b, f.a+f.b
	f.limit--
	return &r
}
```

Now the limit is stored in the generator itself. So we can change our `main` to

```golang
func main() {
	f := fibonacci{0, 1, 20}
	for {
		r := f.Next()
		if r == nil {
			return
		}
		fmt.Printf("%v ", *r)
	}
}
```

If we run this, we get `0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987 1597
2584 4181`, which is looking exactly like the result before.

Until now there were not many surprises. Writing generators like this, can be
done in nearly any language.

## Channels

One cool thing about the generator in python is that there is no constructor
and we can simply iterate over it. In Go we can achieve the same thing with
channels. Let's take a look.

```golang
func fibonacci(limit int) chan int {
	c := make(chan int)
	a := 0
	b := 1
	go func() {
		for {
			if limit == 0 {
				close(c)
				return
			}
			c <- a
			a, b = b, a+b
			limit--
		}
	}()
	return c
}
```

Instead of `yield` we write into the channel and block until someone reads the
value. After that we continue after the write, which is really similar to the
`yield` from Python.

Also this function is returning a `chan int`, so we can directly iterate over
it.

```golang
func main() {
	for r := range fibonacci(20) {
		fmt.Printf("%v ", r)
	}
}
```

If we run this, we get again `0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987
1597 2584 4181`.

So now we do have replicated the behaviour pretty good and we could stop here,
but sometimes you want to iterate and sometimes we want to call `Next`. Let's
try to combine those approaches.

## Combination

The idea here is to create an object and connect a `Next` function to it, but
also be able to iterate over the object itself. Okay, let's take a look.

```golang
type fibonacciChan chan int

func (f fibonacciChan) Next() *int {
	c, ok := <-f
	if !ok {
		return nil
	}
	return &c
}
```

Cool. Now we can call `Next` on our object and get the next result from the
channel and `nil`, if it is closed. Now we only have to adapt our `fibonacci`
function to return a `fibonacciChan`.

```golang
func fibonacci(limit int) fibonacciChan {
	c := make(chan int)
	a := 0
	b := 1
	go func() {
		for {
			if limit == 0 {
				close(c)
				return
			}
			c <- a
			a, b = b, a+b
			limit--
		}
	}()
	return c
}
```

Let's try to to either call `Next` or to iterate with this code.

```golang
func main() {
	f := fibonacci(20)
	fmt.Printf("%v ", *f.Next())
	fmt.Printf("%v ", *f.Next())
	for r := range f {
		fmt.Printf("%v ", r)
	}
}
```

If you execute it, you get again `0 1 1 2 3 5 8 13 21 34 55 89 144 233 377
610 987 1597 2584 4181`.

## Conclusion

So now you have hopefully a better understanding about generators in Go. You
can also find the code on
[GitHub](https://github.com/shaardie/golang-fibonacci-generators). If you have
further questions or comments drop mail.

Happy hacking!