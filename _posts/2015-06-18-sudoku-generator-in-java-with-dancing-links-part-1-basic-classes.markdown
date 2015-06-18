---
layout: post
title:  "Sudoku Generator in Java with Dancing Links, Part 1: basic classes"
shortdescription: >
    In this first part of a series of blog posts, I write about my first steps towards implementing Donald Knuth's Algorithm X using Dancing Links in Java. The goal is to generate valid sudoku puzzles to be used in my Vimdoku project.
tags: projects vimdoku dancing-links
---
*In a series of blog posts, I will detail how I implement a sudoku generator for my simple Sudoku game, [Vimdoku][vimdoku-gh]. I am aiming for these blog posts to be a fairly chronological description of the implementation, but I also want them to be a kind of "journal", where I can explain any problems, troubles or questions I ran into during the development process.*

In this initial blog post I will describe what I am aiming to accomplish, the algorithm I chose and why I chose it, as well as the first basic classes to describe the different parts of the matrix.

### Origin
Generating a valid sudoku is basically the big problem I have to solve in order to develop my simple sudoku game, [Vimdoku][vimdoku-gh]. To read more about my Vimdoku project, go read [my post detailing the project]({% post_url 2015-06-17-vimdoku-sudoku-with-vim-bindings %}). The game needs to generate valid sudoku puzzles that the user then will solve.

### The goal
The aim was to write code that could generate a valid sudoku puzzle for the user to solve, i.e. a sudoku grid with some set of starting numbers supplied such that there is only a single possible solution. The code does not have to take into consideration the difficulty of the grid, but I hope to have it generate fairly simple puzzles -- mostly so that it will be easy to test the application later on.

### Dancing Links
While googling around, using search terms such as "sudoku generator" and "sudoku generator java", I stumbled upon a StackOverflow question with [an answer mentioning dancing links][so-links]. From them, I read up on Wikipedia on [Dancing Links][dlx-wiki] (from now on referred to as "DLX") and the algorithm it was designed for, Donald Knuth's [Algorithm X][algo-x-wiki]. After I felt that the Wikipedia articles were not sufficient for me to get my head around what was going on -- especially with the "matrix" of nodes, column headers, and so on -- I followed the references from the articles and decided to give [Knuth's own paper][dlx-paper] on Algorithm X and DLX a try. It turned out to be fairly effective, at least at making me understand how the matrix was structured. I suggest you read it too, in order to follow the rest of this blog post.

### The Node class
As the first step to implementing DLX in Java, I decided to create classes for the different types of nodes in the matrix (as specified by Knuth in his paper).

All types of nodes share the property of being circularly doubly linked both vertically (`up` and `down`) and horizontally (`left` and `right`) to other nodes. These links are mutable, and them being mutable is basically what DLX relies on. Therefore, I decided to create a very simple `Node` class, with the intention to have more specific types of nodes subclass this class. The `Node` class simply has four fields: references to the `Node`s to the up, down, left, and right of this node, as well as intentionally mutable setters and getters for them.

{% highlight java %}
public class Node {

    private Node up, down, left, right;

    public Node(Node up, Node down, Node left, Node right) {
        this.up = up;
        this.down = down;
        this.left = left;
        this.right = right;
    }

    public void setUp(Node up) {
        this.up = up;
    }

    public Node getUp() {
        return this.up;
    }

    public void setDown(Node down) {
        this.down = down;
    }

    public Node getDown() {
        return this.down;
    }

    public void setLeft(Node left) {
        this.left = left;
    }

    public Node getLeft() {
        return this.left;
    }

    public void setRight(Node right) {
        this.right = right;
    }

    public Node getRight() {
        return this.right;
    }

}
{% endhighlight %}

*(I initially made `Node` an abstract class, but later on changed my mind (see "Root node" below). I also initially made the references `protected`, but later on realized that they were not accessed directly by any of the subclasses, so I made them `private` instead.)*

### The ColumnHeader and ConstraintNode classes

Now on to the more specific types of nodes. The nodes in the matrix all have the behaviour of the general `Node`, but they also all have a reference to their column's column header. The column headers all have the behaviour of the general `Node`, but they also have a name and a size. Therefore, I first defined the `ColumnHeader` class as a subclass to `Node`:

{% highlight java %}
public class ColumnHeader extends Node {

    private String name;

    public ColumnHeader(Node up, Node down, Node left, Node right, String name) {
        super(up, down, left, right);

        this.name = name;
    }

    public int getSize() {
        int size = 0;
        Node node = this.getDown();

        while (node != this) {
            size++;
            node = node.getDown();
        }

        return size;
    }

    public String getName() {
        return this.name;
    }

}
{% endhighlight %}

One thing to notice about this class is how I do not store the size for a `ColumnHeader`, instead opting to calculate it each time the `getSize()` method is called. I made this decision since the size is very simple for the `ColumnHeader` itself to calculate on demand, but fairly obnoxious for other classes to manipulate from the outside. KISS -- Keep It Simple, Stupid™.

(I am tempted to also reason that the size calculation is fast (simply comparing references), but [premature optimization is bad][prem-opt], so I will not do that. :P)

Now that I have the `ColumnHeader` class, I can go on to define the class for the nodes in the matrix. I had a hard time deciding on a fitting name for it, and eventually decided on `ConstraintNode`. The reason for this was because these nodes will most likely represent constraints for the sudoku grid, as inspired by an article by Gareth Rees on [*Zendoku puzzle generation*][zendoku] (see section 4 and the example with the latin square).

{% highlight java %}
public class ConstraintNode extends Node {

    private ColumnHeader header;

    public ConstraintNode(Node up, Node down, Node left, Node right, ColumnHeader header) {
        super(up, down, left, right);

        this.header = header;
    }

    public ColumnHeader getHeader() {
        return this.header;
    }

    public void setHeader(ColumnHeader header) {
        this.header = header;
    }

}
{% endhighlight %}

### Root node
I was scratching my head for a while about how to go about representing the root node of the matrix. The root node resides to the left of the leftmost column header, and only has links in the horizontal directions. I eventually decided to change `Node` from an `abstract` class to a concrete class, and simply let the root node be represented as an instance of `Node` where only the `left` and `right` fields (with their accompanying getters and setters) are intended to be used. This felt a little like an ugly hack, but I figured it would (probably) not create any trouble later on. If it does, however, I will probably just create a class `RootNode` which acts like `Node` but only has the `left` and `right` fields.

### Next steps
Since a DLX matrix can very logically be represented by a binary matrix, where 1 means a node exists in that place and 0 means no node exists in that place, the logical next step is to figure out how to parse a binary matrix into a DLX matrix using the above classes, where all the links are in their correct place. I will cover that in a later blog post.

[vimdoku-gh]:   https://github.com/Saser/vimdoku
[so-links]:     http://stackoverflow.com/a/6964044/407890
[dlx-wiki]:     http://en.wikipedia.org/wiki/Dancing_Links
[algo-x-wiki]:  http://en.wikipedia.org/wiki/Dancing_Links
[dlx-paper]:    http://arxiv.org/abs/cs/0011047
[prem-opt]:     http://sahandsaba.com/nine-anti-patterns-every-programmer-should-be-aware-of-with-examples.html#premature-optimization
[zendoku]:      http://garethrees.org/2007/06/10/zendoku-generation/#section-4.4
