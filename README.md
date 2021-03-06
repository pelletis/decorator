# Decorator

A library that emulates in Java the Scala's Stackable Trait Pattern by implementing the decorator pattern at runtime through class composition instead of only through object composition. The goal is to allow the creation of very small partial components (i.e. partial implementation of interfaces or abstract classes) that can be dynamically assembled into full components at runtime.

The library can also be used to mix and chain aop style decorators with partial components, and also as a more elegant way to chain fully implemented or existing decorators.

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/io.github.pellse/decorator/badge.svg)](https://maven-badges.herokuapp.com/maven-central/io.github.pellse/decorator)

## Usage Examples

This example shows how to create a component by extending an interface and only implement the necessary methods, the framework will  automatically generate the unimplemented pass through delegate methods. The framework will use the provided delegate method signature as an injection point for the delegate class instance to which we should forward methods to:
```java
import java.util.Collection;
import java.util.List;

import com.esotericsoftware.kryo.Kryo;

public interface SafeList<E> extends List<E> {
	
	static Kryo kryo = new Kryo();
	
	// The framework will automatically implement this method.
	// The return type is inspected so the actual method name doesn't matter.
	List<E> getDelegate();

	@Override
	default boolean add(E e) {
		return getDelegate().add(clone(e));
	}
	
	@Override
	default void add(int index, E e) {
		getDelegate().add(index, clone(e));
	}

	@Override
	default boolean addAll(Collection<? extends E> c) {
		return getDelegate().addAll(c.stream().map(SafeList::clone).collect(toList()));
	}

	@Override
	default boolean addAll(int index, Collection<? extends E> c) {
		return getDelegate().addAll(index, c.stream().map(SafeList::clone).collect(toList()));
	}

	@Override
	default E set(int index, E e) {
		return getDelegate().set(index, clone(e));
	}

	static <E> E clone(E obj) {
		return kryo.copy(obj);
	}
}

SafeList<String> decoratorList = Decorator.of(new ArrayList<>(), List.class)
	.with(SafeList.class)
	.make();
```

It is also possible to create a partial component by providing an abstract class that implement only the methods that need to be overriden (some methods were removed for brevity), the framework will use the provided constructor to inject the appropriate delegate:
```java
public abstract class DirtyList<E> implements List<E> {

	private List<E> delegate;

	private boolean isDirty = false;

	public DirtyList(List<E> delegate) {
		this.delegate = delegate;
	}

	public boolean isDirty() {
		return isDirty;
	}

	@Override
	public boolean add(E e) {
		isDirty = true;
		return delegate.add(e);
	}

	@Override
	public boolean remove(Object o) {
		isDirty = true;
		return delegate.remove(o);
	}

	@Override
	public boolean addAll(Collection<? extends E> c) {
		isDirty = true;
		return delegate.addAll(c);
	}
}

DirtyList<String> dirtyList = Decorator.of(new ArrayList<>(), List.class)
	.with(SafeList.class)
	.with(DirtyList.class)
	.make();
```

The `@Inject` annotation is also supported:
```java
public abstract class DirtyList<E> implements List<E> {

	@Inject
	private List<E> delegate;

	private boolean isDirty = false;

	public boolean isDirty() {
		return isDirty;
	}

	@Override
	public boolean add(E e) {
		isDirty = true;
		return delegate.add(e);
	}

	...
}
```

Partial components can also have non zero argument constructors:
```java
public abstract class BoundedList<E> implements List<E> {

	private final int maxNbItems;

	protected abstract List<E> getDelegateList();

	public BoundedList(int maxNbItems) {
		this.maxNbItems = maxNbItems;
	}

	@Override
	public boolean add(E e) {
		checkSize(1);
		return getDelegateList().add(e);
	}
	
	// addAll(), remove(), etc. were not implemented here for brevity

	protected void checkSize(int addCount) {
		if (getDelegateList().size() + addCount >= maxNbItems)
			throw new IllegalStateException("Size of list greater than maxNbItems (" + maxNbItems + ")");
	}
}

DirtyList<String> dirtyList = DecoratorBuilder.of(new ArrayList<>(), List.class)
	.with(SafeList.class)
	.with(BoundedList.class)
		.params(50)
		.paramTypes(int.class)
	.with(DirtyList.class)
	.make();
```

We can also chain partial components with existing decorators that fully implement the specified interface:
```java
DirtyList<String> dirtyList = Decorator.of(new ArrayList<>(), List.class)
	.with(delegate -> Collections.synchronizedList(delegate))
	.with(SafeList.class)
	.with(DirtyList.class)
	.make();
```

This is another example using the fluent api as syntactic sugar to chain already fully implemented decorators instead of embedding layers of constructor parameter calls:
```java
DataInputStream din = DecoratorBuilder.of(new FileInputStream("data.txt"), InputStream.class)
	.with(delegate -> new BufferedInputStream(delegate, 50))
	.with(delegate -> new DataInputStream(delegate))
	.make();
```
Which is the equivalent of:
```java
DataInputStream din = new DataInputStream(new BufferedInputStream(new FileInputStream("data.txt"), 50));
```

And we can mix dynamic proxy components:
```java
DirtyList<String> dirtyList = DecoratorBuilder.of(new ArrayList<>(), List.class)
	.with(SafeList.class)
	.with(BoundedList.class)
		.params(50)
		.paramTypes(int.class)
	.with((delegate, method, args) -> method.invoke(delegate, args))
	.with(DirtyList.class)
	.make();
```

```java
public interface IDirtyList<E> extends List<E> {
	boolean isDirty();
}

public class DirtyListInvocationHandler<T> implements DelegateInvocationHandler<T> {

	private boolean isDirty;

	@Override
	public Object invoke(T delegate, Method method, Object[] args) throws Throwable {
		if (method.getName().equals("isDirty"))
			return isDirty;
			
		if (Stream.of("add", "remove", "set", "clear", "retain").anyMatch(s -> method.getName().startsWith(s)))
			isDirty = true;

		return method.invoke(delegate, args);
	}
}

IDirtyList<EmptyClass> dirtyList = DecoratorBuilder.of(new ArrayList<>(), List.class)
	.with(SafeList.class)
	.with(new DirtyListInvocationHandler<>())
		.as(IDirtyList.class)
	.make();
```

## License

Copyright 2017 Sebastien Pelletier

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
