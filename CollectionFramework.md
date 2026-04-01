# Collection Framework Library



## Collection Framework Library
- Collection Framework Library is a Library of pre-defined interfaces and implementation classes.
- The Collection Framework provides predefined implementation of the standard data structure.
- It provide ready made algorithms to operate on the data.
- Collection is objects of different class type.
- No restriction(limit) in size.
- Implemented using standard data structure.

## What is Collection?
- Collection is a group of objects of same type or different type, which is used to maintain, manage, modify and update the Objects.
- Collection doesn't have any limitation in size, it is a dynamic in nature and it can grow at runtime.

# Types of Collection:
    There are 4 types of Collection,
1. List
2. Queue
3. Set
4. Map


# Pre-defined interfaces of Collection Framework:
1. List
2. Queue
3. Set
4. Map
5. Iterable
6. Iterator

# Pre-defined implementation classes of Collection Framework
1. ArrayList
2. Vector
3. LinkedList
4. Stack
5. PriorityQueue
6. HashSet
7. LinkedHashSet
8. TreeSet
9. HashMap
10. HashTable
11. LinkedHashMap
12. TreeMap

# Utility class of Collection Framework:
1. Collections

# Arrays:
- Collection of homogenous types.
- Object[] arr; // support heterogenous type.

# Data Structure:
- Storing data in a structured manner.
- Provide set of algorithms for operations.
- Algorithms are set of instructions written to perform operations on Data Structure.

# Standard Data Structure & Algorithms:
1. Arrays
2. LinkedList
3. Stack
4. Queue
5. Map & Set
6. Tree
7. Graph
8. Dictionary


## List
- Storing the elements with index is known as List.
- All the elements will have the index.
- Each element is accessed based on its index.
- it is called ordered collection or indexed collection.
- It allows the duplicate elements to be stored.
- It allows, null values.
- It maintains the order of insertion untill it is disturbed.
- It provide index based access.
	

# List Interface : 
- It defines the standard operation that can be performed on a List.
1. public boolean add(int index, Object element);
2. public Object remove(int index)
3. public Object set(int index, Object element)
4. public Object get(int index)
5. public ListIterator listIterator()
6. public List subList(int fromIndex, int toIndex)
7. public int indexOf(Object element)
8. public int lastIndexOf(Object element)

1. public boolean add(int index, Object element);
			This method is used to insert the given element at a given index. If the index is not present the method throws Exception.

2. public Object remove(int index);
			This method removes the element present in the given index, it returns the removed element reference.

3. public Object set(int index, Object element);
			This method is used to replace an element in the given index by another element, it returns the removed element.

4. public Object get(int index);
			This method is used to get reference present in the given index.

5. public ListIterator listIterator();
			This method returns ListIterator type Object used to iterate the elements of the List.

6. public List subList(int fromIndex, int toIndex);
			This method returns the partial List from the specified index till the specified end index excluding the element at the end index.

7. public int indexOf(Object element);
			This method returns the index of first occurance of the given element in the List.

8. public int lastIndexOf(Object element);
			This method returns the index of last occurance of the given element.


# ArrayList: 
- ArrayList is a implementation class of List interface. 
- It provide the implementation to all the abstract methods of List & Collection interface.
- ArrayList class implements 3 marker interface,
1. Serializable
2. Clonable
3. RandomAccess
- ArrayList class contains overloaded constructor.
- The ArrayList uses resizable array data structure to store the elements.
- Whenever we create an object of ArrayList class, it internally create object class type array with default capacity of 10.
- We can create a ArrayList object by specifying the essential capacity, this can be done by using int parameter constructor.
- ArrayList l1 = new ArrayList();
        " ArrayList Has-A array of object class type."
        With default array capacity of 10.
- ArrayList l2 = new ArrayList(100);
        Creating ArrayList with initial capacity of 100.
- In ArrayList class, toString() is overriden to return the elements with comma(,) seperation.
- ArrayList uses the resizable data structure when it grows with the formula,
        newCapacity = (oldCapacity * 1.5) + 1

    # Growing mechanism of ArrayList.
- Steps:
1. Create new ArrayList object with new capacity,
        newCapacity = (oldCapacity * 1.5) + 1
2. Copy elements from old ArrayList to new ArrayList index wise.
3. Old reference changed to point to new ArrayList object.
4. Add new elements.
		





# Vector:
- It is a implementation class of List interface & it is a legacy class.
- Vector & ArrayList functionalities are same.
- Vector is a thread safe class because vector class methods are synchronized.
- The vector grows by doubling capacity of vector.

# LinkedList: 
- LinkedList is a implementation class of List interface.
- It implements 2 marker interfaces,
1. Serializable
2. Clonable.
- The LinkedList class uses doubly LinkedList Data Structure to store the elements.
- The LinkedList class also implements Queue interface hence LinkedList can be used as Queue also.
- ArrayList & LinkedList functionality wise they are similar but implementation wise they are different.
	
	
# Difference between ArrayList & LinkedList
## ArrayList:
        Store:  Sequential List
        Access: faster O(1)
        insert: slower O(n)
        delete: slower O(n)
        ArrayList can't be used as Queue.

## LinkedList:
        Store:  Non-Sequential List
        Access: slower O(n)
        insert: faster O(1)
        delete: faster O(1)
        LinkedList can be used as Queue because it implements Queue interface.
		




## MAP
- It is a collection of the key value pairs.
- The value is associated to key & stored in the map.
- Without key we can not access the value.
- Key must be unique and values can be duplicate.

# Map interface: 
    The Map interface defines the set of standard methods that can be operated on any types of Map.

1. public boolean put(Object key, Object value);
2. public Object get(Object key);
3. public Object remove(Object key);

1. public boolean put(Object key, Object value);
        This method associated the given value with the given key & store in the map.
        It makes a entry of key value pair in the Map.
        If the key is duplicate, it removes the old value & associate the new value.

2. public Object get(Object key);
        This method returns the reference to value associated with the key.

3. public Object remove(Object key);
        This method removes the entire key value pair associated with specified key.
        This method will reduce size of the Map.

# HashMap:
- HashMap is implementation class of Map interface.
- It implements 2 marker interface,
1. Serializable
2. Clonable
- The HashMap uses HashTable Data Structure.
- The default capacity is 16, it grows based on the 'load factor' of the HashMap.
- The HashMap has overloaded constructor.
- HashMap h1 = new HashMap();
        create empty HashTable with default capacity of 16.
        default load factor of 0.75(based on fill ratio).

        Rehashing : creating new HashTable with new capacity.(unknown)
        Rehashing Time: Capacity * load factor
        16*0.75  ==> 12 (This happen only when map is filled 75% ie for 16 size when it becomes 12 it start to Rehashing).
- HashMap h2 = new HashMap(100);
        create an HashTable with given initial capacity of 100 and default load factor of 0.75.
- HashMap h3 = new HashMap(100, 0.9f);
        create an HashTable with given initial capacity of 100 and load factor of 0.90.
# LinkedHashMap:
- LinkedHashMap is a child class of HashMap.
- LinkedHashMap has same properties of HashMap with the only difference HashMap is not synchronized, LinkedHashMap is a synchronized hence LinkedHashMap is Thread safe.
- LinkedHashMap uses 2 Data Structures to store the pair of elements
1. LinkedList
2. HashTable
- LinkedHashMap maintains the order of insertion.

# HashTable:
- HashTable is a implementation class of Map interface.
- It implements 2 marker interface,
1. Serializable
2. Clonable
- HashTable & HashMap uses same Data structure, the only difference is, in HashMap null is allowed as key or value but in HashTable null is not allowed.
- HashTable is also synchronized.
	


## Set
- Stores unique elements only.
- It's an unordered collection.
- It not indexed based collection.
- Doesn't allow duplicate elements.
- we can store null element only once.
- HashSet is a type of Set which strictly doesn't maintain order of insertion.
- LinkedHashSet is a type of Set which doesn't support duplicate elements but maintain the order of insertion.
- TreeSet is a type of Set which doesn't allow duplicates elements but elements are stored in naturally sorted order(Ascending order).

# HashSet:
- HashSet is a implementation class of Set interface.
- It implements 2 marker interfaces
1. Serializable
2. Clonable
- The HashSet internally uses HashMap object.
- Each HashSet object contains an instance of HashMap.
- When an element is added into the HashSet, the hashCode value of element becomes key and element becomes value of the HashMap.
- Since HashSet uses HashMap duplicate values are not allowed.
- It expands its capacity when size > capacity * loadFactor
- where loadfactor is 0.75
	
	Q: How HashSet Ensures the Uniqueness?
	
- hashCode() is used find the bucket index.
- equals() is used to compare elements inside the bucket,
- So,
			- two object with different hashcodes - definitely different.
			- two object with same hashcode - compared using equals().
- Internal Structure Diagram:
			Bucket  Array(table)
			0	-> 	null
			1	->	Node('x')
			2	->	null
			3	->	Node('A') -> Node('B') -> Node('C')
			4	->	null
- Each bucket has either,
			- null
			- a Node
			- or TreeNode(Red-Black Tree)
- Treeification(Java 8+)
- If elements in a single bucket become too many:
## Threshold:
			- Bucket Size >= 8 : Convert Linked List -> Red-Black Tree
			- Bucket Size <= 6 : Convert back to List.
## Reason:
			- Linked List: O(n)
			- Red-Black Tree: O(log n)
- HashSet Time Complexities
		Operation	Time
		add			O(1) average
		remove		O(1) average
		contains	O(1) average
		Worst case	O(log n) after treeification
	
    
# LinkedHashSet:
- It is child class of HashSet.
- It uses hybrid Data Structure, which is a combination of LinkedList and HashMap.
- The capacity of this is 16, maintain the order of insertion.
- It uses,
		- HashMap for hashing.
		- Doubly Linked List to remember the order.
- We can use LinkedHashSet if we need,
		- Unique + Order
		- Not sorted
		- Slightly slower than HashSet.

# TreeSet: 
- It used TreeMap internally.
- It stores elementes in sorted order.
- It used a Red-Black Tree(self-balancing BST).
- It cannot store null(in many cases).
- Uses comparison(Comparable/Comparator).
- Slower than HashSet.
- Only comparable type objects are allowed.
- Internally add() method is casting elements like this,
        Comparable comp = (Comparable)e1;
- Any object added to TreeSet is casted to Comparable type in order to sort element.
- TreeSet is a implementation class of SortedSet interface.
- TreeSet implements two marker interface,
1. Serializable
2. Clonable
- TreeSet follows Binary Tree Data Structure.
- Since TreeSet is a SortedSet, all the elements are arranged in a sorted order, the default sortin order is ascending order.
- We can store only Comparable type object, if we try to store non-Comparable type object we get ClassCastException.
- Whenever an element is added into TreeSet, the element is compared with all the elements already present in the TreeSet and get arranged in ascending order.
	
- Time Complexity of TreeSet
		Operation	Time
		add			O(log n)
		remove		O(log n)
		contains	O(log n)
		traversal	O(n) (sorted order)


## Queue
- Queue is a type of Collection, where elements are processed in FIFO order.
- Generally a Queue is not a indexed.
- Queue allows duplicate but doesn't allow null.
- Generally in a Queue, the intermediate elements can not accessed untill we start removing the elements from head position.

# Queue interface:
- The Queue interface defines standard methods that can be operated on any type of Queue.
- The Queue interface methods are,
        // Used to insert element into Queue at tail end
1. public boolean offer(Object element);
2. public boolean add(Object element);

        // Used to retrieve the head element of the Queue.
3. public Object peek();
4. public Object element();
		
        // Used to remove the head element of the Queue.
5. public Object poll();
6. public Object remove();

# PriorityQueue:
- PriorityQueue is a implementation class of Queue interface.
- It implements two marker interfaces,
1. Serializable
2. Clonable
- The PriorityQueue follows/uses Priority Heap Data Structure.
- Internally uses Binary Heap(Minimum Heap by default).
- The PriorityQueue allows to store same type of elements.
- The least/smallest element becomes head of the Queue.
- Default capacity of PriorityQueue is 11.
- The PriorityQueue stores the elements in unordered fashion.
- It process(remove) the element in a Priority order.
- By default, the least element gets highest Priority.
- On calling poll() on empty Queue return null whereas calling remove() on empty Queue throws NoSuchElementException.
- On calling peek() on empty Queue return null whereas calling element() on empty Queue throws NoSuchElementException
- PriorityQueue Complexities
		Operation				Time
		add						O(log n)
		poll					O(log n)
		peek					O(1)
		remove specific element	O(n)
		contains				O(n)
		
# ArrayDeque:
- It is most recommended for Stack & Queue.
- It is faster than LinkedList, Stack(Legacy), ArrayList(as Stack).
- ArrayDeque Complexities,
		Operation


	## LINKEDLIST INTERVIEW QUESTIONS

1. What is LinkedList in Java?
- LinkedList is a doubly linked list implementation of List and Deque interfaces.
- Elements are stored as nodes where each node has data + reference to prev + next.

2. How does LinkedList store elements internally?
- Internally uses a chain of nodes (Node<E>) with:
- E item
- Node<E> next
- Node<E> prev

3. What is the time complexity of LinkedList operations?
- addFirst() → O(1)
- addLast() → O(1)
- removeFirst() → O(1)
- removeLast() → O(1)
- add(index) → O(n)
- get(index) → O(n)
- remove(index) → O(n)

4. Why is LinkedList slow for random access?
- It must traverse from head or tail to reach the index.
- No direct index-based access like ArrayList.

5. Why is LinkedList faster for insertion in the middle?
- Once the position is found, insertion is O(1).
- Pointer changes only. Traversal still costs O(n).

6. Where is LinkedList preferred over ArrayList?
- When many insertions/deletions happen in beginning or middle.
- When shifting elements in ArrayList is expensive.

7. Why is LinkedList not cache-friendly?
- Nodes are scattered in memory → poor locality of reference.
- CPU caching works poorly → slower operations.

8. How does LinkedList implement Queue and Deque?
- As Queue → addLast(), removeFirst()
- As Deque → addFirst(), addLast(), removeFirst(), removeLast()

9. What is memory overhead in LinkedList?
- Each node stores data + next pointer + prev pointer → high memory use.

10. Does LinkedList support fail-fast iterator?
- Yes. It throws ConcurrentModificationException if structurally modified during iteration.

11. Difference between ArrayList and LinkedList?
- ArrayList → array-based, fast random access.
- LinkedList → node-based, fast insertion/deletion after traversal.
- ArrayList is usually faster in practice.

12. Why is LinkedList slower in real-world usage?
- Poor cache locality
- Higher memory overhead
- More frequent object creation
- More GC pressure

13. How is Node class defined in LinkedList?
- Private static inner class with item, next, prev fields.
- Forms a doubly linked chain.

14. How does LinkedList remove elements internally?
- Adjusts next.prev and prev.next references.
- Clears removed node references to help GC.

15. Is LinkedList thread-safe?
- No. Use Collections.synchronizedList() or ConcurrentLinkedDeque if thread safety is required.
		
		

	## ARRAYLIST INTERVIEW QUESTIONS

1. What is the default capacity of an ArrayList?
	   Default capacity is 10 in Java 8+. Actual array is created only when first element is added.

2. What is the time complexity of major ArrayList operations?
- get() – O(1)
- set() – O(1)
- add(element) – Amortized O(1)
- add(index) – O(n)
- remove(index) – O(n)
- contains() – O(n)
- iteration – O(n)

3. What is the difference between size and capacity?
- size = number of elements in the list.
- capacity = length of internal array.

4. Why is ArrayList not thread-safe?
- Methods are not synchronized. Multiple threads can modify simultaneously leading to inconsistent state.

5. How to make ArrayList thread-safe?
- Use Collections.synchronizedList() or CopyOnWriteArrayList.

6. Why is CopyOnWriteArrayList used?
- Very fast reads, thread-safe, but slow writes because it copies array on every write.

7. Can ArrayList store null values?
- Yes, it can store multiple nulls.

8. How does remove() work internally?
- Removes the element and shifts all remaining elements left by one index. Time complexity: O(n).

9. Difference between Array and ArrayList?
- Array is fixed size; ArrayList is dynamic.
- Array can store primitives; ArrayList cannot (uses wrappers).
- Array has no built-in utility methods; ArrayList has many.

10. Can we increase capacity manually?
- Yes, using ensureCapacity().

11. What happens if you access an invalid index?
- Throws IndexOutOfBoundsException.

12. Difference between remove(int) and remove(Object)?
- remove(int index) removes by index.
- remove(Object o) removes by object value.
- For Integer lists, list.remove(5) removes index 5, not value 5.

13. How does clone() work for ArrayList?
- Creates a shallow copy. Elements are not cloned, only references are copied.

14. What is subList() in ArrayList?
- Returns a view of the list, not a new list. Modifying subList affects original list.

15. Why is ArrayList implemented using Object[] instead of a generic array?
- Java does not allow creation of generic arrays due to type erasure.

16. How does ArrayList reduce capacity?
- Using trimToSize().

17. Does ArrayList implement RandomAccess? Why?
- Yes, to indicate that it supports fast (O(1)) random access.

18. Can we make ArrayList immutable?
- Yes, using List.of() or Collections.unmodifiableList().

19. What happens during serialization of ArrayList?
- Only elements are serialized. Internal array capacity is not serialized.

20. Is ArrayList fail-fast or fail-safe?
- It is fail-fast.
		
21. How does ArrayList grow internally?
- Uses a dynamic array underneath.
- When full, capacity automatically increases.
- Growth formula in Java 8+: newCapacity = old + old/2 (1.5x growth).
- Creates a new larger array.
- Copies all elements from old array to new array.
- Old array is garbage collected.
- Resizing is expensive (O(n)) and happens occasionally.

22. Why is ArrayList faster than LinkedList?
- ArrayList stores elements in contiguous memory.
- LinkedList stores elements in separate nodes with next and previous pointers.
- ArrayList has better CPU cache locality, resulting in faster reads.
- LinkedList has pointer chasing, which makes CPU access slower.
- ArrayList provides O(1) random access.
- LinkedList provides O(n) access because it must traverse nodes.
- LinkedList creates more objects, causing more garbage collection work.

23. Difference between ensureCapacity() and trimToSize()
## ensureCapacity(minCapacity):

- Increases capacity in advance.
- Prevents repeated resizing operations.
- Improves performance for bulk insertions.
- Useful before adding many elements.

## trimToSize():

- Reduces capacity to exact current size.
- Frees unused memory.
- If elements are added later, resizing will happen again.
- Useful when the list is final and will not grow.

24. Why is add(0, x) slow in ArrayList?
- ArrayList is backed by a contiguous array.
- Inserting at index 0 requires shifting all elements to the right.
- The larger the list, the more elements must be shifted.
- Time complexity becomes O(n).
- Any insertion in the middle also suffers from this problem.

25. How does a fail-fast iterator work?
- ArrayList maintains a modCount variable for structural changes.
- Iterator stores expectedModCount when it is created.
- Each call to next() or remove() checks if modCount equals expectedModCount.
- If they differ, a ConcurrentModificationException is thrown.
- Happens when the list is modified outside the iterator.
- Fail-fast behavior is best effort, not guaranteed.
- Helps detect accidental concurrent modifications early.
		
		
	## SET INTERVIEW QUESTIONS

1. How does HashSet work internally?

- HashSet is backed by a HashMap.
- Each element is stored as a key in the HashMap.
- A dummy object is stored as the value.
- Uses hashCode() to find the bucket index.
- Uses equals() to resolve collisions.
- No duplicates because keys in HashMap are unique.
- Order of elements is not guaranteed.

2. Difference between HashSet and LinkedHashSet?

- HashSet does not maintain insertion order.
- LinkedHashSet maintains insertion order using a doubly linked list.
- LinkedHashSet uses more memory.
- LinkedHashSet is slightly slower due to ordering.
- HashSet is preferred when order doesn't matter.

3. What is treeification in HashMap/HashSet?

- When too many elements collide in the same bucket, linked list converts to Red-Black Tree.
- Trigger happens when bucket size reaches 8.
- De-treeify happens when bucket size drops to 6.
- Improves performance from O(n) to O(log n).
- HashSet inherits this behavior from HashMap.

4. Why HashSet uses hashCode() + equals()?

- hashCode() helps find bucket location.
- equals() ensures actual equality among elements.
- Both together prevent false duplicates.
- Needed to maintain correctness and uniqueness.
- Without equals(), hash collisions break duplicate logic.

5. How TreeSet stores elements?

- TreeSet is backed by a TreeMap.
- Internally uses Red-Black Tree structure.
- Stores elements in sorted order.
- Uses Comparable or Comparator for ordering.
- No duplicate elements allowed.

6. Difference between HashSet and TreeSet?

- HashSet is unordered; TreeSet is sorted.
- HashSet uses hashing; TreeSet uses Red-Black Tree.
- HashSet operations → O(1) average.
- TreeSet operations → O(log n).
- HashSet allows null; TreeSet does not in Java 8+.
- TreeSet is slower but preserves sorted order.

7. Time complexities of each Set implementation

- HashSet: O(1) average, O(n) worst
- LinkedHashSet: O(1) average
- TreeSet: O(log n)

8. Why is HashSet not synchronized?

- Designed for performance, not thread-safety.
- Use synchronizedSet() or concurrent Set wrappers for thread-safe operations.

9. Can HashSet store null?

- Yes, allows one null value.
- LinkedHashSet also allows null.
- TreeSet does not allow null (Java 8+).

10. How does HashSet handle collisions?

- Uses chaining with linked lists or trees.
- equals() checks final equality inside the bucket.

11. Why TreeSet is slower than HashSet?

- Needs tree rotations and comparisons.
- Hashing is faster than tree traversal.

12. When to use TreeSet over HashSet?

- When sorted order is required.
- When range operations like headSet(), tailSet(), subSet() are needed.

13. How TreeSet ensures sorting?

- Uses compareTo() for natural ordering.
- If a Comparator is provided, that is used instead.

14. Difference between Comparator and Comparable in TreeSet?

- Comparable defines natural ordering inside the class.
- Comparator defines custom ordering externally.
- TreeSet prefers Comparator; if absent, uses Comparable.

15. Why is HashSet iteration order unpredictable?

- Depends on hash distribution and resizing behavior.
- Any size change may change iteration order.

16. How LinkedHashSet maintains insertion order?

- Uses a doubly linked list connecting all entries.
- Linked list preserves the insertion sequence.

17. Can HashSet be converted to TreeSet?

- Yes, by passing HashSet to TreeSet constructor.
- Resulting TreeSet will automatically sort the elements.


