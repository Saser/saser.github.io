---
layout: post
title:  "Sudoku Generator in Java with Dancing Links, Part 2: DLXMatrix and parsing"
shortdescription: >
    After defining the basic classes for the nodes in a DLX matrix, I now have to figure out how to represent an entire matrix, as well as parsing a binary matrix into a DLX matrix.
tags: projects vimdoku dancing-links
---
*In a series of blog posts, I will detail how I implement a sudoku generator for my simple Sudoku game, [Vimdoku][vimdoku-gh]. I am aiming for these blog posts to be a fairly chronological description of the implementation, but I also want them to be a kind of "journal", where I can explain any problems, troubles or questions I ran into during the development process.*

In this blog post I will describe how I went about defining a class `DLXMatrix` to represent an entire matrix in the DLX algorithm, as well as how I parse a binary matrix consisting of 0:s and 1:s into a `DLXMatrix`. If you haven't read my [first post in this series][vimdoku-p1], I suggest you do that first.

## Representing the entire DLX matrix
So while we have our basic node classes -- `Node`, `ConstraintNode` and `ColumnHeader` -- we do not have any class to represent the entire DLX matrix as a whole. My thought is that such a class will have the responsibility to perform the actual algorithm steps, such as covering and uncovering columns (as described by [Donald Knuth in his paper][dlx-paper]). It also felt natural to have this class expose functionality to parse a binary matrix into a DLX matrix.

With this in mind, I decided to simply have a class called `DLXMatrix`. This class will not have any public constructors; instead, it will have a public method called `parse()` that takes a binary matrix (that is, an `int[][]` matrix) consisting of 0:s and 1:s, which will create and return a new `DLXMatrix`.

At first I felt unsure about what fields the `DLXMatrix` class should have. I eventually decided on a root node (which is simply a `Node` instance, as explained in the end of the [previous post][vimdoku-p1]), and an array of `ColumnHeader`s. I did not include the actual matrix of `ConstraintNode` instances, since I thought that the `DLXMatrix` class would mostly be operating on column headers (since they are used to cover and uncover columns), and the root node.

Without further ado, here is the skeleton for the `DLXMatrix` class:

{% highlight java %}
public class DLXMatrix {

    private Node root;
    private ColumnHeader[] headers;

    private DLXMatrix(Node root, ColumnHeader headers) {
        this.root = root;
        this.headers = headers;
    }

    public static DLXMatrix parse(int[][] intMatrix) {
        // Parsing here...
        return null;
    }

    public Node getRoot() {
        return this.root;
    }

    public ColumnHeader[] getHeaders() {
        return this.headers;
    }

}
{% endhighlight %}

## Parsing a binary matrix
As you can see above, the `parse()` function does not really do anything yet. We have to fill it with code that correctly parses a binary matrix into the different types of nodes, and returns a `DLXMatrix` instance for the parsed matrix.

I felt that the parsing does not have to be done by the `DLXMatrix` class itself, since that would clutter up the code a bit. Instead, I placed all the parsing code in a package private `abstract` class, with only static methods, called `DLXMatrixParser`, and delegated the parsing in the `DLXMatrix` class to the `DLXMatrixParser` class.

I wanted `DLXMatrixParser` to also have a method called `parse()` that parses a binary matrix, links all the nodes together correctly, and returns the matrix.

{% highlight java %}
abstract class DLXMatrixParser {

    public static Node[][] parse(int[][] intMatrix) {
        // Coming soon...
        return null;
    }

}
{% endhighlight %}

### Creating ConstraintNodes
The logical first step is to create instances of `ConstraintNode`, since otherwise there would be no nodes to link to each other. This is very straight-forward -- create a matrix of `ConstraintNode`s and loop over the binary matrix, placing `null` where there is a 0 and `ConstraintNode` instances without any links where there is a 1. This is done in a private method called `fromIntMatrix` in `DLXMatrixParser`:

{% highlight java %}
private static Node[][] fromIntMatrix(int[][] intMatrix) {
    int rows = intMatrix.length;
    int columns = intMatrix[0].length;
    Node[][] matrix = new Node[rows][columns];

    for (int row = 0; row < rows; row++) {
        for (int column = 0; column < columns; column++) {
            Node n = null;
            if (intMatrix[row][column] == 1) {
                n = new ConstraintNode(null, null, null, null, null);
            }
            matrix[row][column] = n;
        }
    }

    return matrix;
}
{% endhighlight %}

Now we have the first building blocks for our `parse()` method:

{% highlight java %}
public static Node[][] parse(int[][] intMatrix) {
    Node[][] matrix = fromIntMatrix(intMatrix);
    // More stuff to be done here...
    return matrix;
}
{% endhighlight %}

### Linking nodes together
Now we have a matrix of `Node` instances, but all their links are `null`, meaning that they do not link to anything. The next step is therefore to create all the links in the matrix, following the directions given in Knuth's paper: all rows are circularly doubly linked, and all columns are circularly doubly linked. I figured that I should split the linking into two parts: linking all nodes together horizontally in a row, and linking all nodes together vertically in a column. Therefore, I decided to use two new static methods, called `linkRow()` and `linkColumn()` respectively, and use them in the above `parse()` method:

{% highlight java %}
public static Node[][] parse(int[][] intMatrix) {
    Node[][] matrix = fromIntMatrix(intMatrix);
    int rows = matrix.length;
    int columns = matrix[0].length;

    for (int row = 0; row < rows; row++) {
        linkRow(matrix, row);
    }

    for (int column = 0; column < columns; column++) {
        linkColumn(matrix, column);
    }

    return matrix;
}

...

private static void linkRow(Node[][] matrix, int rowNum) {
    // Linking goes here...
}

private static void linkColumn(Node[][] matrix, int columnNum) {
    // Linking goes here...
}
{% endhighlight %}

The linking itself is very straight-forward, and can be seen as sequentially "pairing" two `Node`s together. Say that you have two `Node`s -- `x` and `y` -- and you want to link them together such that `x` is considered the "first" `Node` and `y` is considered the "second" ("first" being the leftmost when linking horizontally, or upper when linking vertically). Then the horizontal pairing can be described like this (in pseudo-code):

    x.right <- y
    y.left <- x

Or, if pairing vertically, like this :

    x.down <- y
    y.up <- x

To keep track of which type of pairing that should be done -- horizontal for rows, vertical for columns -- I decided to create a simple private `enum` in the `DLXMatrixParser` class:

{% highlight java %}
abstract class DLXMatrixParser {

    private enum PairType {
        ROW,
        COLUMN
    }

    ...

}
{% endhighlight %}

So, in order to fully link a row or a column, it is as simple as taking pairs of existing `Node`s (not `null` ones!), starting from the beginning and going all the way to the end, finally "tying the ends together" in order to make the list circular.

First, I added the following code to the `linkRow()` and `linkColumn()` methods, inventing a couple of helper methods as I went along:

{% highlight java %}
private static void linkRow(Node[][] matrix, int rowNum) {
    Node[] nodeArray = matrix[rowNum];
    List<Node> nodeList = nonNullNodes(nodeArray);

    pairAll(nodeList, PairType.ROW);
}

private static void linkColumn(Node[][] matrix, int columnNum) {
    Node[] nodeArray = getColumn(matrix, columnNum);
    List<Node> nodeList = nonNullNodes(nodeArray);

    pairAll(nodeList, PairType.COLUMN);
}
{% endhighlight %}

The names of the helper methods -- `nonNullNodes`, `getColumn` and `pairAll` -- should all be fairly self-explanatory. Nothing fancy goes on in `nonNullNodes` and `getColumn`; however, `pairAll` is a bit more interesting.

The `pairAll` method has the responsibility to pair all nodes in the given `List<Node>` list, as well as "tying the ends" together. It accomplishes this by pairing two nodes at a time. The tying is done by simply considering the last `Node` in the list to be the "first" `Node` in the pair, and the first `Node` in the list to be the second `Node` in the pair.

{% highlight java %}
private static void pairAll(List<Node> list, PairType type) {
    int size = list.size();
    for (int i = 0; i < size; i++) {
        Node first = list.get(i);
        Node second = list.get((i + 1) % size);
        pair(first, second, type);
    }
}
{% endhighlight %}

The modulo calculation is what ties the ends together -- when `i` is at `size - 1`, `(i + 1) % size` is equal to `0`, meaning that `first` becomes the last `Node` in `list`, and `second` becomes the first `Node` in `list`.

The actual pairing is delegated to another method, `pair()`. That method then performs the correct pairing based on the `PairType` it is given:

{% highlight java %}
private static void pair(Node first, Node second, PairType type) {
    if (type == PairType.ROW) {
        pairInRow(first, second);
    } else if (type == PairType.COLUMN) {
        pairInColumn(first, second);
    }
}

private static void pairInRow(Node left, Node right) {
    left.setRight(right);
    right.setLeft(left);
}

private static void pairInColumn(Node up, Node down) {
    up.setDown(down);
    down.setUp(up);
}
{% endhighlight %}

There we have it -- `DLXMatrixParser`s `parse()` method gives us a complete matrix, where all `Node`s are circularly doubly linked. Super simple stuff! Let's add it to the `parse()` method in `DLXMatrix`:

{% highlight java %}
public static DLXMatrix parse(int[][] intMatrix) {
    Node[][] matrix = DLXMatrixParser.parse(intMatrix);
    return null;
}
{% endhighlight %}

Now, let's move on to the next step -- adding the column headers.

### Adding ColumnHeaders
When all the ordinary nodes are in place and correctly linked to each other, we need to "insert" the special column headers that are crucial to the DLX algorithm.

Creating the column headers is very simple. We need one `ColumnHeader` for every column in the matrix created above, so we simply loop through the columns and create one `ColumnHeader` for each, as well as setting the `header` field of each `ConstraintNode` in the column to the newly created `ColumnHeader`. While we are looping through the columns, we take the opportunity to "insert" the `ColumnHeader` at the same time.

Since we want our `DLXMatrix` class to have an array of `ColumnHeader`s, we add all `ColumnHeader`s to an array and return it. All of this is done in a top-level method `insertHeaders()`, which we create the skeleton for like so:

{% highlight java %}
public static ColumnHeader[] insertHeaders(Node[][] matrix) {
    int columns = matrix[0].length;
    ColumnHeader[] headers = new ColumnHeader[columns];
    for (int i = 0; i < columns; i++) {
        // Create ColumnHeader

        // Set header field of constraint nodes in column

        // Insert the ColumnHeader between bottom and top node

        // Add created ColumnHeader to headers array
    }

    // Circularly doubly link the headers together

    return headers;
}
{% endhighlight %}

One thing that always feels a little ugly is using `matrix[0].length` as column count (since `0` is a hardcoded value). However, since we know that the matrix is in fact a matrix we also know that all rows have the same length, so getting the column count from the row length of any given row is valid.

Creating the `ColumnHeader` is very simple: it doesn't have any links at creation, so the `up`, `down`, `left`, and `right` fields are all `null`. I chose using the `i` value as name since it guarantees all names are unique (unless you have more than 2<sup>32</sup> columns, which I doubt).

{% highlight java %}
// Create ColumnHeader
ColumnHeader header = new ColumnHeader(null, null, null, null, "" + i);
{% endhighlight %}

Then we have to set the `header` field of each `ConstraintNode` in the column. Fortunately, the helper methods from earlier -- specifically `getColumn()` and `nonNullNodes()` -- can still be used and make our lives a lot easier.

{% highlight java %}
// Set header field of constraint nodes in column
List<Node> nodeList = nonNullNodes(getColumn(matrix, i));
{% endhighlight %}

But wait -- we have a list of `Node`s, but we need `ConstraintNode`s. The good thing is that we know that `matrix` consists of `ConstraintNode`s only, so we can cast them. The [`Stream`][j8-stream] API:s from Java 8 provides functionality to write some really clean, expressive code for this. Casting to `ConstraintNode` and then setting the `header` field is done in only three lines, like so:

{% highlight java %}
nodeList.stream()
        .map(n -> (ConstraintNode) n)
        .forEach(n -> n.setHeader(header));
{% endhighlight %}

Perhaps it is better to specify `matrix` to have the type `ConstraintNode[][]` instead of `Node[][]`, but for now I will leave it as it is.

Now that we have linked all the `ConstraintNode`s to the newly created `ColumnHeader`, we are going to insert it. By inserting I mean to insert the node into the circular doubly linked list, between the bottom and top nodes. At the moment, the bottom and top nodes are linked to each other:

    ..., bottom, top, ...

For the DLX algorithm to work, the list should look like this:

    ..., bottom, header, top, ...

Inserting a node like above can be generalised to inserting the node `between` between the nodes `first` and `second`. Note that the order matters:

Before insertion of `between`:

    ..., first, second, ...

After insertion of `between`:

    ..., first, between, second, ...

I decided to write a helper method for this, akin to the pairing helper methods used earlier. In fact, the insertion can be simplified to only consisting of two pairings: pairing `first` with `between`, and pairing `between` with `second` (remember that the order within the pairs matters).

{% highlight java %}
public static void insertBetween(Node first, Node second, Node between, PairType type) {
    pair(first, between, type);
    pair(between, second, type);
}
{% endhighlight %}

Now inserting the header into the column is simply a matter of getting references to the top and bottom nodes of a column, and inserting the header between them.

Since we have a list of all `Node`s in the column we simply get the first and last elements of that list:

{% highlight java %}
// Insert the ColumnHeader between bottom and top node
Node top = nodeList.get(0);
Node bottom = nodeList.get(nodeList.size() - 1);
insertBetween(bottom, top, header, PairType.COLUMN);
{% endhighlight %}

The final thing to do while looping over the columns is to add the header to the `headers` array:

{% highlight java %}
// Add created ColumnHeader to headers array
headers[i] = header;
{% endhighlight %}

There is only one thing left -- linking the column headers together so that they also are circularly doubly linked as a row. So, before returning the `headers` array, we link the headers together:

{% highlight java %}
// Circularly doubly link the headers together
List<Node> headerList = nonNullNodes(headers);
pairAll(headerList, PairType.ROW);

return headers;
{% endhighlight %}

Now the column headers are done. Cool! We revisit the `parse()` method in `DLXMatrix` and add our `insertHeaders()` method:

{% highlight java %}
public static DLXMatrix parse(int[][] intMatrix) {
    Node[][] matrix = DLXMatrixParser.parse(intMatrix);
    ColumnHeader[] headers = DLXMatrixParser.insertHeaders(matrix);
    return null;
}
{% endhighlight %}

The final step is to create and add the root node.

### Adding the root node
The root node is an instance of `Node`, and is placed to the left of the leftmost column header. Since the headers are already correctly linked, we need to insert the root node between the first and last headers. Therefore I wrote a method `createRoot()` that takes an array of `ColumnHeader`s, creates and inserts the root node, and returns a reference to the root node.

{% highlight java %}
public static Node createRoot(ColumnHeader[] headers) {
    // Create root Node instance
    Node root = ...;

    // Insert in headers

    // Return reference to root
    return root;
}
{% endhighlight %}

We create the root `Node` by simply creating a `Node` (not `ConstraintNode` or `ColumnHeader`) without any links to anything.

{% highlight java %}
// Create root Node instance
Node root = new Node(null, null, null, null);
{% endhighlight %}

Then we get the first and last headers, and insert `root` so that the order becomes `..., last, root, first, ...`:

{% highlight java %}
// Insert in headers
Node first = headers[0];
Node last = headers[headers.length - 1];
insertBetween(last, first, root, PairType.ROW);
{% endhighlight %}

Now we are ready to return:

{% highlight java %}
// Return reference to root
return root;
{% endhighlight %}

Let's revisit the `parse()` method, now that we have all the pieces we need. This is what it currently looks like:

{% highlight java %}
public static DLXMatrix parse(int[][] intMatrix) {
    Node[][] matrix = DLXMatrixParser.parse(intMatrix);
    ColumnHeader[] headers = DLXMatrixParser.insertHeaders(matrix);
    return null;
}
{% endhighlight %}

We add our `createRoot()` method. After that, we are ready to create a new `DLXMatrix` instance and return it:

{% highlight java %}
public static DLXMatrix parse(int[][] intMatrix) {
    Node[][] matrix = DLXMatrixParser.parse(intMatrix);
    ColumnHeader[] headers = DLXMatrixParser.insertHeaders(matrix);
    Node root = DLXMatrixParser.createRoot(headers);

    return new DLXMatrix(root, headers);
}
{% endhighlight %}

### Comments
You can see there is a kind of beautiful structure here: the outputs of methods are "piped" into each other. `intMatrix` is "piped" into `parse(intMatrix)` and `matrix` is returned. `matrix` is "piped" into `insertHeaders(matrix)` and `headers` is returned. Finally, `headers` is "piped" into `createRoot(headers)` and `root` is returned. It actually resembles Unix one-liners a little, where you pipe the output of a program into the input of another program, e.g.:

{% highlight bash %}
cat a b b | sort | uniq -u > c # c is the set difference a - b
{% endhighlight %}

## Summary
Whew, that was a long post! Let's summarize it:

We split the parsing of a binary matrix into three parts, which all fit together to completely parse and link the binary matrix:

1.  `DLXMatrixParser.parse()`: the binary matrix is converted to unlinked `ConstraintNode`s. These `ConstraintNode`s are then circularly doubly linked together both vertically and horizontally.
2.  `DLXMatrixParser.insertHeaders()`: column headers, in the form of `ColumnHeader` instances, are created. All `ConstraintNode` instances recieve their respective header, and the headers are inserted in the columns.
3.  `DLXMatrixParser.createRoot()`: a root `Node` is created and inserted into the row of column headers.

## Moving forward
After this, we have completely parsed a binary matrix into a valid DLX matrix, with circularly linked nodes, column headers, and a root node. Now that we have our DLX matrix, it is time for the fun stuff -- using it to solve exact cover problems! I will cover that in the next blog post.

[vimdoku-gh]:   https://github.com/Saser/vimdoku
[vimdoku-p1]:   {% post_url 2015-06-18-sudoku-generator-in-java-with-dancing-links-part-1-basic-classes %}
[dlx-paper]:    http://arxiv.org/abs/cs/0011047
[j8-stream]:    http://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html
