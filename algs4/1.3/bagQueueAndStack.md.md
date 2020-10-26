``` python
class Node:
    def __init__(self, item=None, next=None):
        super().__init__()
        self.item = item
        self.next = next

class Bag:
    '''
    * `void add(Item item)`
	* `boolean isEmpty()`
	* `int size()`
    '''
    _N = 0
    _first = None

    def add(self, item):
        if item == None:
            print('item should not be None')
            return
        n = Node()
        n.item = item
        n.next = self._first
        self._first = n
        self._N += 1
    def isEmpty(self):
        return self._N == 0
    def size(self):
        return self._N

class Stack:
    '''
    * `void push(Item item)`'
	* `Item pop()`
	* `boolean isEmpty()`
	* `int size()`
    '''
    _N = 0
    _first = None

    def push(self, item):
        if item == None:
            print('item should not be None')
            return
        n = Node()
        n.item = item
        n.next = self._first
        self._first = n
        self._N += 1
    def pop(self):
        if self.isEmpty():
            print('Stack is empty')
            return None
        n = self._first
        self._first = self._first.next
        self._N -= 1
        return n.item
    def isEmpty(self):
        return self._first == None
    def size(self):
        return self._N
        

class Queue:
    '''
    * `void enqueue(Item item)`
	* `Item dequeue()`
	* `boolean isEmpty()`
	* `int size()`
    '''
    _N = 0
    _first = None
    _last = None
    def enqueue(self, item):
        if item == None:
            print('item should not be None')
            return
        n = self._last
        self._last = Node()
        self._last.item = item
        if self.isEmpty():
            self._first = self._last
        else:
            n.next = self._last
        self._N += 1
    
    def dequeue(self):
        if self.isEmpty():
            print('Queue is empty')
            return None
        n = self._first
        self._first = self._first.next
        if self.isEmpty():
            self._last = None
        self._N -= 1
        return n.item
        
    def isEmpty(self):
        return self._first == None
    def size(self):
        return self._N

b = Bag()
s = ['to', 'be', 'or', 'not', 'to', '-', 'be', '-', '-', 'that', '-','-', '-', 'is']
'''
stack = Stack()
for obj in s:
    if obj != '-':
        stack.push(obj)
    elif not stack.isEmpty():
        print(f'{stack.pop()} ')
print(f'({stack.size()} left on stack)')
'''

q = Queue()
for obj in s:
    if obj != '-':
        q.enqueue(obj)
    elif not q.isEmpty():
        print(f'{q.dequeue()} ')
print(f'({q.size()} left on queue)')
```