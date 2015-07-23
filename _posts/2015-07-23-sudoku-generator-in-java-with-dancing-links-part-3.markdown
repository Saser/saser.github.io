---
layout: post
title:  "Sudoku Generator in Java with Dancing Links, Part 3: The Algorithm"
shortdescription: >
    It is time to implement the actual Dancing Links algorithm, using our DLXMatrix class and different node classes.
tags: projects vimdoku dancing-links
---
_In a series of blog posts, I will detail how I implement a sudoku generator for my simple Sudoku game, [Vimdoku][vimdoku-gh]. I am aiming for these blog posts to be a fairly chronological description of the implementation, but I also want them to be a kind of "journal", where I can explain any problems, troubles or questions I ran into during the development process._

In this blog post I will describe how I used the classes created so far to implement the actual Dancing Links algorithm. There still are some problems to solve though, as my implementation is naïve and does not actually return any results yet.

If you haven't read my two previous blog posts yet -- [part 1][vimdoku-p1] and [part 2][vimdoku-p2] -- I suggest you do that first.

## Using the Node and Matrix Classes
We put a lot of effort into creating our classes so far -- `Node`, `ConstraintNode`, `ColumnHeader` and `DLXMatrix` -- and now we will finally get to see the fruit of our work. The classes are specifically designed to be used for this particular algorithm, so hopefully the implementation of the algorithm will be fairly straight-forward.

It felt natural to put the actual algorithm in the `DLXMatrix` class, since it is supposed to be the publicly facing interface for solving an exact-cover problem using the DLX algorithm. The other classes are declared `public` as well, but you should only have to interact with the `DLXMatrx` class from the outside.

## Limitations
My idea was to have a single public method -- `search` -- belonging to the `DLXMatrix` be the only method that should be called from the outside to get all solutions (if any). However, after reading how Knuth described the algorithm, it dawned on me that I did not know what kind of result `search` should return. Knuth simply "prints" the solution when one is found -- it is never returned anywhere. I feel like figuring out the return value of `search` is something worth thinking about for a while to get it right. For the time being, `search` is simply a `void` that prints any solutions it finds.

## A Word of Advice
Before we go any further, make sure you have read and have a fairly good understanding of the algorithm: both the more abstract Algorithm X, and the Dancing Links/DLX implementation of it. If you need a reminder, [Knuth's paper][dlx-paper] is the definitive place to read up on them.

## The Search Method
Knuth uses a variable _k_ and some kind of list _O_ in his description of Dancing Links. Since the algorithm is recursive it felt natural to initially create two methods -- one `public void search()` and one `private void searchAux(List<Node> list, int k)`. The thought is that `searchAux` is the recursive method, and `search` is simply responsible for starting the recursion by calling `searchAux(list, 0)`.

Let's have a look at the two skeleton methods so far:

{% highlight java %}
public void search() {
    List<Node> list = new ArrayList<>();
    this.searchAux(list, 0);
}

private void searchAux(List<Node> list, int k) {
    // TODO: implement the recursive algorithm here
}
{% endhighlight %}

All of the algorithm will go into the `searchAux` method. Let's write down some pseudo-but-not-actually-pseudo-code by trying to directly translate Knuth's description of Dancing Links into Java, making up methods as we go. A good cue of where we need to make up a method is when Knuth writes "(see below)":

{% highlight java %}
private void searchAux(List<Node> list, int k) {
    if (this.root.getRight() == this.root) {
        printSolution(list);
        return;
    }

    ColumnHeader c = this.selectHeader();
    cover(c);

    for (Node r = c.getDown(); r != c; r = r.getDown()) {
        if (list.size() <= k) {
            list.add(r);
        } else {
            list.set(k, r);
        }

        for (Node j = r.getRight(); j != r; j = j.getRight()) {
            cover(j);
        }

        this.searchAux(list, k + 1);

        r = list.get(k);
        c = ((ConstraintNode) r).getColumnHeader();

        for (Node j = r.getLeft(); j != r; j = j.getLeft()) {
            uncover(j);
        }
    }
    uncover(c);
}
{% endhighlight %}

Whew, that is a lot of code. Let's walk through it little by little, so that we get an understanding of how it corresponds to the actual algorithm.

We begin with the edge condition. Knuth phrases it as such:

> If R[h] = h, print the current solution (see below) and return.

The important part here is "If R[h] = h [...]". We need to check if the node to the right of the root is the root itself, which is very simple:

{% highlight java %}
if (this.root.getRight() == this.root) {
    ...
}
{% endhighlight %}

Notice how we are deliberately comparing references. It makes sense in this context -- we want to know if the node to the right of `this.root` is _actually_ `this.root` itself.

We need some way to print the solution, but it seems fitting to break out the printing into a method itself, so we made up a _static_ method `printSolution` and call it with `list` as the only argument.

{% highlight java %}
if (this.root.getRight() == this.root) {
    printSolution(list);
    return;
}
{% endhighlight %}

Next, we look at the next two lines of Knuth's description:

> Otherwise choose a column object c (see below).  
Cover column c (see below).

We have two "(see below)"s here, which hints us to create two new methods to implement the behaviour that he describes:

{% highlight java %}
ColumnHeader c = this.selectHeader();
cover(c);
{% endhighlight %}

This is very straight forward, but one thing to notice is that I made `selectHeader` an _instance_ method, but `cover` is a _static_ method. The reason for that is that I read ahead a little and saw that the selection of a column header is done by traversing all "active" column headers, and how do we do that? That is right -- by starting at `this.root` and using `getRight` or `getLeft` to traverse the headers. Therefore, it needs to be an instance method. However, covering a column is only dependent on the column itself, and therefore can be done in a static context. (How the actual covering is done is discussed later -- let's just pretend we have a static method for it for the time being.)

Here comes the hairy part. We are entering a loop over a couple of nodes, and doing some other stuff like covering columns inside that loop. However, let's start with the loop itself, which Knuth describes as such:

> For each r ← D[c], D[D[c]], ... , while r ≠ c:

At first I thought about using a `while` loop, primarily because the condition said "while r ≠ c":

{% highlight java %}
Node r = c.getDown();
while (r != c) {
    ...
    r = r.getDown();
}
{% endhighlight %}

But then I remembered reading some blog post about specifically Dancing Links in sudoku, and the author had done some clever use of the standard `for` loop:

{% highlight java %}
for (Node r = c.getDown(); r != c; r = r.getDown()) {
    ...
}
{% endhighlight %}

The advantages of this approach over the `while` approach are primarily:

1.  We do not need to introduce a `Node` variable in the outer scope
2.  We do not need to remember putting a `r = r.getDown()` call at the end of the while loop

I had never thought about using a standard `for` loop this way. Previously I had either used a standard `for` loop with an `int` increasing or decreasing a value (`for (int i = 0; i < limit; i++)`), or used a "foreach" loop (`for (Thing something : someCollection)`). I will definitely keep this trick up my sleeve as I am sure I will find similar uses for it in the future.

Moving on, let's look at the actual contents of the loop. Knuth writes:

> set O<sub>k</sub> ← r;

Here, _O_ is our `list`, and _k_ is our `k`, so we might be tempted to follow the paper wording as close as possible and naïvely do:

{% highlight java %}
list.set(k, r);
{% endhighlight %}

This will lead to out-of-bounds exceptions though, so let's prevent that:

{% highlight java %}
if (list.size() <= k) {
    list.add(r);
} else {
    list.set(k, r);
}
{% endhighlight %}

Next up, we are going to loop over some constraint nodes and cover columns. Knuth writes:

> for each j ← R[r], R[R[r]], ... , while j ≠ r:  
cover column j (see below);

We can do this with another clever `for` loop:

{% highlight java %}
for (Node j = r.getRight(); j != r; j = j.getRight()) {
    cover(j);
}
{% endhighlight %}

There's really nothing fancy going on here -- covering a column will be discussed later. One thing to notice, though, is that the algorithm instructs us to "cover column _j_"; however, _j_ is not a column header, but a constraint node. We will have to deal with that in our `cover` method (probably by checking the dynamic type of the argument to that method).

Next up is the recursive call to the `search` method again. Knuth writes:

> search(k + 1);

Which is very simple to write in code:

{% highlight java %}
this.searchAux(list, k + 1);
{% endhighlight %}

After the recursive call, it is time to begin "working upwards" from the recursion, and restore the DLX matrix to a previous state. This is the actual backtracking performed. The first part is to restore a node from the _O_ list, and restore the column header based on that node. In the paper:

> set r ← O<sub>k</sub> and c ← C[r];

This is a little more complicated than it might seem at first, because of the different node types. We have to get the `Node` from `list`, and cast it to a `ConstraintNode` because otherwise we are not able to call the `getHeader` method to acquire the column header. It looks like this in the code:

{% highlight java %}
r = list.get(k);
c = ((ConstraintNode) r).getHeader();
{% endhighlight %}

Perhaps this casting between types several times is suboptimal. I am of the opinion that typecasting should be done as little as possible, instead focusing on catching as many type errors at possible at compile time. Perhaps my entire structure of classes and implementation needs to be redone to avoid typecasting, but that is a later problem. For now, let's roll with it.

Now it's time for another clever for loop, but this time we are _uncovering_ the columns instead of _covering_ them. The paper says:

> for each j ← L[r], L[L[r]], ... , while j ≠ r,  
uncover column j (see below).

The implementation of the loop is basically the same as previously, but we use our made-up `uncover` method instead:

{% highlight java %}
for (Node j = r.getLeft(); j != r; j = j.getLeft()) {
    uncover(j);
}
{% endhighlight %}

Finally, we return from the outmost loop and uncover the column header we began with. Knuth writes:

> Uncover column c (see below) and return.

Which is as simple as:

{% highlight java %}
uncover(c);
{% endhighlight %}

Whew, that was a lot of code and words. Let's just have a final look at the entire `searchAux` method before we move on to implement our made-up methods.

{% highlight java %}
private void searchAux(List<Node> list, int k) {
    if (this.root.getRight() == this.root) {
        printSolution(list);
        return;
    }

    ColumnHeader c = this.selectHeader();
    cover(c);

    for (Node r = c.getDown(); r != c; r = r.getDown()) {
        if (list.size() <= k) {
            list.add(r);
        } else {
            list.set(k, r);
        }

        for (Node j = r.getRight(); j != r; j = j.getRight()) {
            cover(j);
        }

        this.searchAux(list, k + 1);

        r = list.get(k);
        c = ((ConstraintNode) r).getColumnHeader();

        for (Node j = r.getLeft(); j != r; j = j.getLeft()) {
            uncover(j);
        }
    }
    uncover(c);
}
{% endhighlight %}

## Printing the Solution
The printing of the solution is very straight-forward, so we warm up by implementing the `printSolution` method. Knuth describes it as:

> The operation of printing the current solution is easy: We successively print the rows containing _O_<sub>0</sub>, _O_<sub>1</sub>, ... , _O_<sub>k -- 1</sub>, where the row containing data object _O_ is printed by printing N[C[O]], N[C[R[O]]], N[C[R[R[O]]]], etc.

First, let's think about how the printing is actually performed. It "loops over" the elements of _O_, meaning that we "loop over" nodes (and more specifically, `ConstraintNode`s). Then, for each such node, we "loop over" all nodes in that row, and for each node in _that_ loop, we print the name of the column header. Does that make sense? If not, sit down with pen and paper and try to follow the printing by hand. Don't worry about having a DLX matrix with a valid solution, just assume that some subset of rows make up a solution and try to print it.

Okay, now let's have a go at the code to implementing this printing method. We start with a skeleton, like so:

{% highlight java %}
private static void printSolution(List<Node> list) {
    // Printing goes here...
}
{% endhighlight %}

We look at the first loop in the description of the printing: we loop over the elements of _O_, which in the code is represented by `list`. This time we can use a "for each" style loop, since we do not need to keep track of the index of each element:

{% highlight java %}
for (Node n : list) {
    // Printing goes here...
}
{% endhighlight %}

Now we are supposed to loop over the nodes to the right of this node, as hinted in the paper. However, the paper doesn't explicitly state how far we are going to loop, so when do we stop? The most probable answer is when we have printed the whole row, which means going right all the way until we end up at the start -- the lists are circularly linked, remember? We can do this with one of the fancy for loops again, with the small exception that we have to print the start node first, then loop over the rest:

{% highlight java %}
ColumnHeader c = ((ConstraintNode) n).getHeader();
System.out.print(c.getName());
for (Node next = n.getRight(); next != n; next = next.getRight()) {
    c = ((ConstraintNode) next).getHeader();
    System.out.printf(", %s", c.getName());
}
System.out.println();
{% endhighlight %}

However, we don't want to repeat code, so we break out the retrieval of the column header name into its own function:

{% highlight java %}
private static String getColumnHeaderName(Node n) {
    ColumnHeader c = ((ConstraintNode) n).getHeader();
    return c.getName();
}
{% endhighlight %}

Now our final `printSolution` function looks like this:

{% highlight java %}
private static void printSolution(List<Node> list) {
    for (Node n : list) {
        System.out.print(getColumnHeaderName(n));
        for (Node next = n.getRight(); next != n; next = next.getRight()) {
            System.out.printf(", %s", getColumnHeaderName(next));
        }
        System.out.println();
    }
}
{% endhighlight %}

## Selecting a Column Header
We stated earlier that we needed a method to decide which column to cover, so we created `selectHeader`. But how do we decide this? Fortunately, Knuth proposes a very simple way:

> To choose a column object c, we could simple set c ← R[h]; this is the leftmost uncovered column. Or if we want to minimize the branching factor, we could set s ← ∞ and then  
> for each j ← R[h], R[R[h]], ..., while j ≠ h,  
>     if S[j] < s, set c ← j and s ← S[j]

What this means is simply to choose the column with the smallest "size", i.e. the column with the lowest number of nodes in it. Let's implement that:

{% highlight java %}
private ColumnHeader selectHeader() {
    Node header = null;
    int smallestSize = Integer.MAX_VALUE;

    for (Node j = this.root.getRight(); j != this.root; j = j.getRight()) {
        int currentSize = ((ColumnHeader) j).getSize();
        if (currentSize < smallestSize) {
            header = j;
            smallestSize = currentSize;
        }
    }

    return (ColumnHeader) header;
}
{% endhighlight %}

This implementation is a little awkward -- we are constantly casting from `Node` to `ColumnHeader`. Also, we cannot set the initial `smallestSize` to infinity (obviously) so `Integer.MAX_VALUE` will have to suffice. Apart from that, the implementation is fairly straight-forward.

## Covering and Uncovering
The final piece to the puzzle is to implement the `cover` and `uncover` methods. In order to do that, we once again take to Knuth's paper.

In the very first paragraph of his paper, Knuth describes the operations required to remove from and reinsert into a double linked list. (The reinserting is very similar to the linking (as discussed in [part 2][vimdoku-p2] of this series), however it is a little simpler.)

Removing an element _x_ from a double linked list is performed like such (as taken from Knuths paper):

> L[R[x]] ← L[x],    R[L[x]] ← R[x]

Or, described a little clearer:

    x.right.left ← x.left,
    x.left.right ← x.right

Reinserting an element _x_ into a doubly linked list is more or less the inverse:

> L[R[x]] ← x,    R[L[x]] ← x

Or, described a little clearer:

    x.right.left ← x,
    x.left.right ← x

Note that this description explicitly uses "right" and "left". In our case, where we have both vertical and horizontal doubly linked lists, we simply have to substitute "right" and "left" for "down" and "up" in appropriate places when working with the vertical lists.

It makes sense to implement methods for the removal and reinsertion of elements in a doubly linked list, like we did with the linking, so let's do that. We have to make a distinction between removing in the horizontal list and removing in the vertical list, so we create an enum at the top of the class:

{% highlight java %}
public class DLXMatrix {

    private enum DirectionType {
        ROW,
        COLUMN
    }

    ...
}
{% endhighlight %}

Further down, let's create our methods for removal and reinsertion. They're very simple, and very similar to the linking methods described in the previous blog post, so I will not cover them in any detail:

{% highlight java %}
private static void remove(Node n, DirectionType type) {
    if (type == DirectionType.ROW) {
        removeInRow(n);
    } else if (type == DirectionType.COLUMN) {
        removeInColumn(n);
    }
}

private static void removeInRow(Node n) {
    Node left = n.getLeft();
    Node right = n.getRight();
    left.setRight(right);
    right.setLeft(left);
}

private static void removeInColumn(Node n) {
    Node up = n.getUp();
    Node down = n.getDown();
    up.setDown(down);
    down.setUp(up);
}

private static void reinsert(Node n, DirectionType type) {
    if (type == DirectionType.ROW) {
        reinsertInRow(n);
    } else if (type == DirectionType.COLUMN) {
        reinsertInColumn(n);
    }
}

private static void reinsertInRow(Node n) {
    Node left = n.getLeft();
    Node right = n.getRight();
    left.setRight(node);
    right.setLeft(node);
}

private static void reinsertInColumn(Node n) {
    Node up = n.getUp();
    Node down = n.getDown();
    up.setDown(node);
    down.setUp(node);
}
{% endhighlight %}

Now when we have that fresh in mind, let's take a look at the covering process for a column header _c_. It looks like the following in Knuth's paper:

    Set L[R[c]] ← L[c] and R[L[c]] ← R[c].
    For each i ← D[c], D[D[c]], ..., while i ≠ c,
        for each j ← R[i], R[R[i]], ..., while j ≠ i,
            and set S[C[j]] ← S[C[j]] - 1.

_(Sorry for the code block formatting -- indentation doesn't work very well with quote blocks in Markdown.)_

Knuth's description also updates the size of the column header for each node, but since we compute the size of the column on demand -- instead of updating it externally -- we simply skip that step in our implementation.

Knuth also has a simpler overall explanation of the process, in words:

> The operation of covering column c is more interesting: It removes c from the header list and removes all rows in c’s own list from the other column lists they are in.

Okay, so we first "remove" the column header horizontally. Then we loop through all nodes of the column vertically. We create a new method `cover`:

{% highlight java %}
private static void cover(Node n) {
    // ...
}
{% endhighlight %}

Wait, look at the method signature -- `cover` takes a single argument, `Node n`. But we want to cover columns! The reason for this is because of our `searchAux` method, where we called our `cover` method with `Node`s as arguments, since that is how Knuth described it. So, first of all we have to make sure we are working on a `ColumnHeader`. We know that we are either covering a `ColumnHeader` directly, or a `ConstraintNode`, so we need to handle those cases:

{% highlight java %}
private static void cover(Node n) {
    ColumnHeader c;
    if (n instanceof ColumnHeader) {
        c = (ColumnHeader) n;
    } else {
        c = ((ConstraintNode) n).getHeader();
    }

    // ...
}
{% endhighlight %}

Now, let's remove the column header horizontally and begin looping through its nodes (essentially looping through rows):

{% highlight java %}
remove(c, DirectionType.ROW);
for (Node i = c.getDown(); i != c; i = i.getDown()) {
    // ...
}
{% endhighlight %}

Now, for each row, loop through all its nodes and remove them vertically:

{% highlight java %}
for (Node i = c.getDown(); i != c; i = i.getDown()) {
    for (Node j = i.getRight(); j != i; i = i.getRight()) {
        remove(j, DirectionType.COLUMN);
    }
}
{% endhighlight %}

So, the entire `cover` method looks like the following:

{% highlight java %}
private static void cover(Node n) {
    ColumnHeader c;
    if (n instanceof ColumnHeader) {
        c = (ColumnHeader) n;
    } else {
        c = ((ConstraintNode) n).getHeader();
    }

    for (Node i = c.getDown(); i != c; i = i.getDown()) {
        for (Node j = i.getRight(); j != i; i = i.getRight()) {
            remove(j, DirectionType.COLUMN);
        }
    }
}
{% endhighlight %}

Now let's have a look at the uncovering process for a column _c_. It is more or less precisely the reverse of the covering process -- all steps are reversed:

    For each i ← U[c], U[U[c]], ..., while i ≠ c,
        for each j ← L[i], L[L[i]], ..., while j ≠ i,
            and set U[D[j]] ← j, D[U[j]] ← j.
    Set L[R[c]] ← c and R[L[c]] ← c.

Again, we can skip the size updating since we calculate it on demand. It is very straight-forward to implement the `uncover` method:

{% highlight java %}
private static void uncover(Node n) {
    ColumnHeader c;
    if (n instanceof ColumnHeader) {
        c = (ColumnHeader) n;
    } else {
        c = ((ConstraintNode) n).getHeader();
    }

    for (Node i = c.getUp(); i != c; i = i.getUp()) {
        for (Node j = i.getLeft(); j != i; i = i.getLeft()) {
            reinsert(j, DirectionType.COLUMN);
        }
    }

    reinsert(c, DirectionType.ROW);
}
{% endhighlight %}

Notice that we have the retrieval of the `ColumnHeader` in two places in our code, so let's refactor it out to it's own method:

{% highlight java %}
private static ColumnHeader retrieveHeader(Node n) {
    ColumnHeader c;
    if (n instanceof ColumnHeader) {
        c = (ColumnHeader) n;
    } else {
        c = ((ConstraintNode) n).getHeader();
    }

    return c;
}
{% endhighlight %}

We update our `cover` and `uncover` methods:

{% highlight java %}
private static void cover(Node n) {
    ColumnHeader c = retrieveHeader(n);

    ...
}

private static void uncover(Node n) {
    ColumnHeader c = retrieveHeader(n);

    ...
}
{% endhighlight %}

## Summary
Woah, that was a long post. But it was worth it -- we have a (hopefully!) working implementation of the DLX algorithm! Let's give ourselves a pat on the back! \*patpat\*

There is two major problems left to solve, though:

1.  Our `search` method does not yet return anything, so any solutions that we find are simply printed and then lost.
2.  We need to find a way to use our DLX algorithm to solve/generate sudoku puzzles, meaning that we need to create a fitting DLX matrix to model a sudoku puzzle.

That is for another blog post, though.

[vimdoku-gh]:   https://github.com/Saser/vimdoku
[vimdoku-p1]:   {% post_url 2015-06-18-sudoku-generator-in-java-with-dancing-links-part-1-basic-classes %}
[vimdoku-p2]:   {% post_url 2015-06-22-sudoku-generator-in-java-with-dancing-links-part-2 %}
[dlx-paper]:    http://arxiv.org/abs/cs/0011047
