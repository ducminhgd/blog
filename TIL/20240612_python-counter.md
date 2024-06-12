# Python 3: Counter object

```python
>>> from collections import Counter

>>> Counter()
Counter()

>>> Counter('Hello world')
Counter({'l': 3, 'o': 2, 'H': 1, 'e': 1, ' ': 1, 'w': 1, 'r': 1, 'd': 1})

>>> Counter({'a': 1, 'b': 2})
Counter({'b': 2, 'a': 1})

>>> Counter({1,2,3,4,1})
Counter({1: 1, 2: 1, 3: 1, 4: 1})

>>> Counter([1,2,3,4,1])
Counter({1: 2, 2: 1, 3: 1, 4: 1})

>>> c = Counter([1,2,3,4,5,6,7,1,2,3,4])
>>> for e in c.elements():
...     print(e)
...
1
1
2
2
3
3
4
4
5
6
7
```

## Read more

1. https://www.geeksforgeeks.org/python-counter-objects-elements/
2. https://docs.python.org/3/library/collections.html#collections.Counter