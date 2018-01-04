# Moving to python 3 with pleasure
## A short guide on migrating from python2 to python3 for data scientists

Most probably, you already know about the problems caused by inconsistencies between python2 and python3, 
here I cover some of the changes that may come in handy for data scientists.

- Питон 2 популярен среди ДС
- Однако питон 3 уже тоже весьма популярен, вот например Джейк https://jakevdp.github.io/blog/2013/01/03/will-scientists-ever-move-to-python-3/ писал про это и он теперь уверен, что стоит двигаться в сторону питон 3
- Если вы только начинаете, то точно стоит сразу учить питон3
- Здесь гайд по переходу на третий питон, и какие плюшки это принесет
- Мне не нравится учить людей питону3, потому что требуется сразу объяснять iterables, которые не слишком нужны, но путают



## Better paths handling with `pathlib`

`pathlib` is a default module in python3, that helps you to avoid tons of `os.path.join`s:

```python
from pathlib import Path
dataset_root = Path('/path/to/dataset/') 
train_path = dataset_root / 'train'
test_path = dataset_root / 'test'
for image_path in train_path.iterdir():
    with image_path.open() as f: # note, open is a method of 
        # do something with an image
```

Previously it was always tempting to use string concatenation (which is obviously bad, but more concise), 
with `pathlib` code is safe, concise, and readable.

Also `pathlib.Path` has a bunch of methods, that every python novice had to google (and anyone who is not working with files all the time):

```python
p.exists()
p.is_dir()
p.parts()
p.with_name('sibling.png') # only change the name, but keep the folder
p.with_suffix('.jpg') # only change the extension, but keep the folder and the name
p.chmod(mode)
p.rmdir()
```

`Pathlib` should save you lots of time, 
please see [docs](https://docs.python.org/3/library/pathlib.html) and [reference](https://pymotw.com/3/pathlib/) for more.


## Type hinting is now part of the language

<img src='pycharm-type-hinting.png' />
<span style='font-size: 0.5em;'>type hinting in pycharm</span>

Python is not just a language for small scripts anymore, 
data pipelines these days include numerous steps each involving different frameworks (and sometimes very different logic).

Type hinting was introduced to help with growing complexity of programs, so machines could help with code verification.

For instance, the following code may work with dict, pandas.DataFrame, astropy.Frame, numpy.recarray and a dozen of other containers.
```
def compute_time(data):
    data['time'] = data['distance'] / data['velocity'] 
```


Things are also quite complicated when operating with tensors, which may come from different frameworks.

```
def convert_to_grayscale(images):
    return images.mean(axis=1)
```

Еще нужно return doctypes продемонстрировать, IDE контролирует, когда возвращается что-то не то.

```
def train_on_batch(batch_data: tensor, batch_labels: tensor) -> Tuple[tensor, float]:
  ...
  return loss, accuracy
```

(In most cases) IDE will spot an error if you forgot to convert an accuracy to float.

If you have a significant codebase, tools like [MyPy](http://mypy.readthedocs.io) will likely to become part of your continuous integration pipeline. 

A webinar ["Putting Type Hints to Work"](https://www.youtube.com/watch?v=JqBCFfiE11g) by Daniel Pyrathon is good for a brief introduction.

Sidenote: unfortunately, right now hinting is not yet powerful enough to provide fine-grained typing for ndarrays/tensors, but [maybe we'll have it once](https://github.com/numpy/numpy/issues/7370), and this will be a great feature for DS.

## Type hinting -> type checking in runtime

By default, type hinting does not influence how your code is working, but merely helps you to point code intentions.

However, you can enforce type checking in runtime with tools like [enforce](https://github.com/RussBaz/enforce).

```
@enforce.runtime_validation
def foo(text: str) -> None:
    print(text)

foo('Hi') # ok
foo(5)    # fails   

# enforce also supports callable arguments
@enforce.runtime_validation
def foo(a: typing.Callable[[int, int], str]) -> str:
    return a(5, 6)
    
```


## Matrix multiplication as @

Let's implement one of the simplest ML models - a linear regression with l2 regularization:

```
# L2-regularized linear regression: || AX - b ||^2 + alpha * ||x||^2 -> min

# python2
X = np.linalg.inv(np.dot(A.T, A) + alpha * np.eye(A.shape[1])).dot(A.T.dot(b))
# python3
X = np.linalg.inv(A.T @ A + alpha * np.eye(A.shape[1])) @ (A.T @ b)
```

The code with `@` becomes more readable and more translatable between deep learning frameworks: same code `X @ W + b[None, :]` for a single layer of perceptron works in `numpy`, `cupy`, `pytorch`, `tensorflow` (and other frameworks that operate with tensors).


## Globbing with `**`

Recursive folder globbing is not easy in python2, even custom module [glob2](https://github.com/miracle2k/python-glob2) was written to overcome this. Recurive flag is supported since python3.6:

```python
import glob
# python2

found_images = \
    glob.glob('/path/*.jpg') \
  + glob.glob('/path/*/*.jpg') \
  + glob.glob('/path/*/*/*.jpg') \
  + glob.glob('/path/*/*/*/*.jpg') \
  + glob.glob('/path/*/*/*/*/*.jpg') 

# python3

found_images = glob.glob('/path/**/*.jpg', recursive=True)
```

Better option is to use `pathlib` in python3 (minus one import!):
```python
found_images = pathlib.Path('/path/').glob('**/*.jpg')
```

## Print is a function now

Probably, you've already learnt this, but apart from adding parenthesis, there are some advantages:

- simple syntax for using file descriptor:
    ```python
    print >>sys.stderr, "fatal error"      # python2
    print("fatal error", file=sys.stderr)  # python3
    ```
- printing tab-aligned tables without `str.join`:
    ```python
    print(*array, sep='\t')
    print(batch, epoch, loss, accuracy, time, sep='\t')
    ```
- hacky suppressing / redirection of printing output:
    ```python
    _print = print # store the original print function
    def print(*args, **kargs):
        pass  # or do something useful, e.g. store output to some file
    ```
- `print` can participate in list comprehensions and other language constructs 
    ```python
    result = process(x) if is_valid(x) else print('invalid item: ', x)
    ```

## f-strings for simple and reliable formatting

Default formatting system provides a flexibility that is not required in data experiments. 
Resulting code is either too verbose or too fragile towards any changes.

Quite typically data scientist outputs iteratively some logging information as in a fixed format. 
It is common to have a code like:

```python
# python2
print('{batch:3} {epoch:3} / {total_epochs:3}  accuracy: {acc_mean:0.4f}±{acc_std:0.4f} time: {avg_time:3.2f}'.format(
    batch=batch, epoch=epoch, total_epochs=total_epochs, 
    acc_mean=numpy.mean(accuracies), acc_std=numpy.std(accuracies),
    avg_time=time / len(data_batch)
))

# python2 (too error-prone during fast modifications, please avoid):
print('{:3} {:3} / {:3}  accuracy: {:0.4f}±{:0.4f} time: {:3.2f}'.format(
    batch, epoch, total_epochs, numpy.mean(accuracies), numpy.std(accuracies),
    time / len(data_batch)
))
```

Sample output:
```
120  12 / 300  accuracy: 0.8180±0.4649 time: 56.60
```

**f-strings** aka formatted string literals were introduced in python 3.6:
```python
print(f'{batch:3} {epoch:3} / {total_epochs:3}  accuracy: {numpy.mean(accuracies):0.4f}±{numpy.std(accuracies):0.4f} time: {time / len(data_batch):3.2f}')
```


## Explicit difference between 'true division' and 'integer division'

This may be not a good fit for system programming in python, but for data science this is definitely a positive change

```python
velocity = distance / time

data = pandas.read_csv('timing.csv')
velocity = data['distance'] / data['time']
```

Result in python2 depends on whether 'time' and 'distance' (e.g. measured in meters and seconds) are stored as integers.
In python3, result is correct in both cases, promotion to float happens automatically when needed. 

Another case is integer division, which is now an explicit operation:

```python
n_gifts = money // gift_price
```

Note, that this works for built-in types as well as for custom provided by data packages (e.g. `numpy` or `pandas`).


## Constants in math module

```python
math.inf # 'largest' number
math.nan # not a number

max_quality = -math.inf

for model in trained_models:
    max_quality = max(max_quality, compute_quality(model, data))
```

## Strict ordering 

```python
3 < '3'
2 < None
(3, 4) < (3, None)
(4, 5) < [4, 5]
(4, 5) == [4, 5]
```

- Prevents from occasional sorting of instances of different types
  ```python
  sorted([2, '1', 3])  # invalid for python3, in python2 returns [2, 3, '1']
  ```
- Generally, helps to spot some problems that arise when processing raw data

Sidenote: proper check for None is
```python
if a is not None:
  pass
  
if a: # WRONG check for None
  pass
```

## Iterable unpacking

```python
# handy when amount of additional stored info may vary between experiments, but the same code can be used in all cases
model_paramteres, optimizer_parameters, *other_params = load(checkpoint_name)

# picking two last values from a sequence
*prev, next_to_last, last = values_history

# This also works with any iterables, so if you have a function that yields e.g. qualities,
# below is a simple way to take only last two values from a list 
*prev, next_to_last, last = iter_train(args)
```

## Unicode for NLP

```python
print(len('您好'))
```
Output:
- python2: `6`
- python3: `2`. 

```
x = u'со'
x += 'со'
```
python2 fails, python3 works as expected (because I've used russian letters in strings).

In python3 `str`s are unicode strings, and it is more convenient for NLP processing of non-english texts.

There are less obvious things, for instance:

```python
print(sorted([u'a', 'a']))
print(sorted(['a', u'a']))
```

Python2 outputs:
```
[u'a', 'a']
['a', u'a']
```


## Default pickle engine provides better compression for arrays

```
import cPickle as pickle
import numpy
print len(pickle.dumps(numpy.random.normal(size=[1000, 1000])))
# result: 23691675

python 3
import pickle
import numpy
len(pickle.dumps(numpy.random.normal(size=[1000, 1000])))
# result: 8000162
```

Three times less space.
Actually close compression is achievable with `protocol=2` parameter, but developers usually ignore this option (or simply not aware of it). 


## OrderedDict is faster now 

OrderedDict is probably the most used structure after list. 
It a good old dictionary, which keeps the order in which keys were added. 


## Single integer type

Python2 provides two basic integer types, which are `int` (64-bit signed integer) and `long` (for long arithmetics).
Quite confusing after C++.

Python3 now has only `int`, which also provides long arithmetics.

Checking for integer:

```
isinstance(x, numbers.Integral) # python2, the canonical way
isinstance(x, [long, int])      # python2
isinstance(x, int)              # python3, easiest to remember
```

## Other stuff

- `Enum`s
- yield from 
- async / await


## Main problems for code in data science and how to resolve those

- relative imports from subfloders
  - packaging 
  - sys.path.insert
  - softlinks
  
- support for nested arguments [was dropped](https://www.python.org/dev/peps/pep-3113/)
  ```
  map(lambda x, (y, z): x, z, dict.items())
  ```
  
  However, it is still perfectly working with different comprehensions:
  ```python
  {x:z for x, (y, z) in d.items()}
  ```
  In general, comprehensions are also much better 'translatable' between python2 and python 3.

- `map`, `.values()`, `.items()` do not return lists.
  Problem with iterators are:
  - no trivial slicing
  - can't be used twice
  
  Almost all of the problems resolve it by converting to list.

## Main problems for teaching machine learning and data science with python 

Data science courses struggle with some of the changes (but python is still the most reasonable option).

Course authors should spend time in the beginning to explain what is an iterator, 
why is can't be sliced / concatenated like a string (and how to deal with it).



# Conclusion

Python2 and Python3 co-exist for almost 10 years, but right now it it time that you *should* move to python3.

There are issues with migration, but the advantages worth it.
Your research and production code should benefit significantly from moving to python3-only codebase.

And I can't wait the bright moment when libraries can drop support for python2 (which will happen quite soon) and completely enjoy new language features.

Following migrations will be smoother: ["we will never do this kind of backwards-incompatible change again"](https://snarky.ca/why-python-3-exists/)

### Links

- [Key differences between Python 2.7 and Python 3.x](http://sebastianraschka.com/Articles/2014_python_2_3_key_diff.html) (и смотри внутри)

