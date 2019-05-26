---
layout: default
title: Emergent properties in C# Equality
date: 2019-05-25
categories: csharp
---

### Introduction

C# allows developers to define type equality through two mechanisms: equality operator (`==`) overloading, and `Equals(object)` method overriding. Many user defined types leverage both, allowing equality operations through operator notation or method invocation. Often, the `==` operator overload simply calls `Equals(object)`, so that these operations are equivalent and provide flexible syntax. However, the two operations are fundamentally different in C#, even if they appear interchangable.

### Problem

A class definition of a two dimensional point is found below. The equality definitions on this class follow the [MSDN](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/how-to-define-value-equality-for-a-type) guide for C#. The hash code implementation follows guidance from [Programming in Scala, Chapter 28](https://www.artima.com/pins1ed/object-equality.html).

```
class Point : IEquatable<Point>
{
	public int X { get; }
	public int Y { get; }

	public Point(int x, int y)
	{
		this.X = x;
		this.Y = y;
	}

	public override bool Equals(object obj)
	{
		return this.Equals(obj as Point);
	}

	public bool Equals(Point point)
	{
		if (object.ReferenceEquals(point, null))
		{
			return false;
		}

		if (object.ReferenceEquals(this, point))
		{
			return true;
		}

		if (this.GetType() != point.GetType())
		{
			return false;
		}

		return (this.X == point.X) && (this.Y == point.Y);
	}

	public override int GetHashCode()
	{
		return 41 * (41 + this.X) + this.Y;
	}

	public static bool operator ==(Point lhs, Point rhs)
	{
		if (Object.ReferenceEquals(lhs, null))
		{
			return Object.ReferenceEquals(rhs, null);
		}

		return lhs.Equals(rhs);
	}

	public static bool operator !=(Point lhs, Point rhs)
	{
		return !(lhs == rhs);
	}
}
```

Now consider the following code snippet, where the two equality mechanisms are used.

```
// Two point references with same internal values
Point p1 = new Point(1, 3);
Point p2 = new Point(1, 3);
object o1 = p1;
object o2 = p2;

Console.WriteLine(p1.Equals(p2)); // True, uses Equals(Point) on Point
Console.WriteLine(p1 == p2); // True, matches operator overload signature on Point class
Console.WriteLine(o1.Equals(o2)); // True, uses Equals(object) defined on Point class
Console.WriteLine(o1 == o2); // False, uses object's == operator (reference equality)
```

The difference between operator and method resolution becomes obvious on lines 3 and 4.

Overloaded operator calls are static methods that are resolved at compile time using compile time types. Instance method calls support polymorphism and are resolved at runtime. Using `==` on variables with the static type `object` will always call the reference equality implementation on `object`. This behavior is fixed since static methods cannot be overridden, and an operator overload must have at least one argument with the type of the containing class.

Thus `static bool operator ==(object, object)` cannot be defined on the `Point` class. However, a call to `Equals(object)` resolves based on the runtime type of the receiver. `Point` overrides the `Equals(object)` method and this is chosen for invocation at runtime on line 3, while line 4 uses the `object` definition of equality. The `==` operator can be errouneously thought of as syntactic sugar for `Equals(object)`, but the internal mechanics of C# mean these two have a crucial difference. This example used the classes `object` and `Point`, but this behavior would be seen with any polymorphic setup.

### Conclusion

Pragmatically, operator overloading is treated as syntactic sugar for instance method calls, and so the above result can clash with expectations. From the design perspective, it is tempting to consider this result an oversight or poor design; however, locally, the C# designers made reasonable decisions. By defining operators as static methods, truly symmetric operators can be defined, and operator overloading is not simply glorified syntatic sugar. Similarly, polymorphic method resolution is ubiquitous across today's OOP languages. When these local decisions are combined into an integrated whole, new behavior emerges that clashes with often loose formalism.

If you are interested in more discussion about the design decisions behind `==` and `Equals(object)`, Eric Lippert wrote a [small blog post](https://blogs.msdn.microsoft.com/ericlippert/2009/04/09/double-your-dispatch-double-your-fun/) on the topic.
