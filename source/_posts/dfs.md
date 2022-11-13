---
Depth First Search - DFS
---

A DFS is an algorithm to travers graph-based structures (also for trees included in graphs)
in a depth order fashion. 

```
    DFS is helpful to traverse nodes of a graph only once and you want to go deep in a child from parent node as much as it is possible.
```

### Usecases:

Before jumping into discussion what DFS is, let's understand the usecase of DFS.

Imagine that you are asked to write a basic algorithm for a Chess game. The opponent which can 
be an AI computer wants to defeat the player. It should decide all available movements for a piece.

To continue, let's imagine computer starts with a Pawn. It is obvious the first move 
only go one or two upfront (just as the beginning of time). If you want to devise a  
very professional algorithm that stimulate the player, it should predict the other movements after.
 
How about the next moves? Computer can decide randomly but have a good play matters to win and challenge the game!

So for this algorithm we can use start the move in the memory and then see other available movements
until no movement is possible and pick the one to prevent defeat fast!

A pseudo code for our simple algorithm is as follows:

~
pick a piece (pawn here)
1. for any available movements in neighbor cells
    check if movement is possible
    insert it into a Stack if it was possible
2. for any movement in Stack
    peek the move
    check if it will be defeated?
    repeat 1.      
~
 

Now you can see our algorithm is getting toward intelligent movements and can simulate a real opponent.


#### Definition      

Before going to implement this chess algo let's DFS.
For a DFS we have a Graph (G for instance),and a starting node that we want to start traversing the graph 
The pseudo code for DFS is like below:

##### G: Graph, v: starting node (can be root node) , vertex: node, vertices: nodes
 
```java
dfs(G, v):
    label v as discovered
    for all vertices connected to v
        check vertex is not visited before?
            call dfs(G,vertex)
```

Note that recursive calling acts like a Stack for us.If we want to have iterative approach we will need a stack.
Below is the non-recursive implementation or iterative approach for the above pseudo code.

 
```java
dfs(G, v):
    define a stack 
    put v into stack
    while stack is not empty
        poll top vertex from stack
        label v as discovered
        for all vertices connected to v
            push vertex into stack
```

Imagine we have tree (sort of a simple graph), then you can see the below action when you run the algo against this tree:  

![dfs](source/images/dfs.gif)
      
## Implementation of Chess algo     

Now Let's see how to we can implement the chess algo in Java using DFS.

There are two ways to implement the DFS:

* Recursive approach
* Iterative approach 
  
I will implement the recursive approach.

```java

```


I will implement the iterative approach.

```java
import java.util.Stack;

public class DFS {

    static class Vertex {
        private final String cell;

        Vertex(String cell) {
            this.cell = cell;
        }
    }

    static class Tree {
        public final Vertex vertex;
        public final Tree left;
        public final Tree right;
        public boolean visited;

        private Tree(Builder builder) {
            this.left = builder.getLeft();
            this.right = builder.getRight();
            this.vertex = builder.getValue();
        }

        public static class Builder {
            private Tree left, right;
            private final Vertex value;

            public Builder(Vertex vertex) {
                value = vertex;
            }

            public static Builder builder(Vertex vertex) {
                return new Builder(vertex);
            }

            public Builder setLeft(Tree left) {
                this.left = left;
                return this;
            }

            public Builder setRight(Tree right) {
                this.right = right;
                return this;
            }

            public Tree getLeft() {
                return left;
            }

            public Tree getRight() {
                return right;
            }

            public Vertex getValue() {
                return value;
            }

            Tree build() {
                return new Tree(this);
            }
        }

        public Vertex getVertex() {
            return vertex;
        }

        public Tree getRight() {
            return right;
        }

        public Tree getLeft() {
            return left;
        }
    }

    public DFS() {
        Tree tree = Tree.Builder.builder(new Vertex("2"))
                .setLeft(Tree.Builder.builder(new Vertex("3"))
                        .setLeft(Tree.Builder.builder(new Vertex("4")).build())
                        .setRight(Tree.Builder.builder(new Vertex("1")).build())
                        .build()
                )
                .setRight(Tree.Builder.builder(new Vertex("10")).build())
                .build();

        Tree tree2 = Tree.Builder.builder(new Vertex("1"))
                .setLeft(Tree.Builder.builder(new Vertex("2"))
                        .setLeft(Tree.Builder.builder(new Vertex("3"))
                                .setLeft(Tree.Builder.builder(new Vertex("4")).build())
                                .setRight(Tree.Builder.builder(new Vertex("5")).build())
                                .build())
                        .build()).build();
 
        Stack<Tree> vertices = new Stack<>();
        vertices.push(tree);
        vertices.push(tree2);
        while (!vertices.isEmpty()) {
            Tree topNode = vertices.pop();
            if (!topNode.visited) {
                System.out.println(topNode.vertex.cell);
            }
            topNode.visited = true;
            Tree leftChild = topNode.left;
            Tree rightChild = topNode.right;
            if (rightChild != null && !rightChild.visited) {
                vertices.push(rightChild);
            }
            if (leftChild != null && !leftChild.visited) {
                vertices.push(leftChild);
            }
        }
    }

    public static void main(String[] args) {
        new DFS();
    }
}

```


Another example using Graph

```java

public class DepthFirstSearch {

    public static void main(String[] args) {
        Vertex v1 = Vertex.build("1");
        Vertex v2 = Vertex.build("2");
        Vertex v3 = Vertex.build("3");
        Vertex v4 = Vertex.build("4");
        Vertex v5 = Vertex.build("5");

        v1.setConnection(v2.setConnection(v3, v4), v5);
        dfs(v1);
    }

    static void dfs(Vertex root) {
        List<Vertex> visited = new ArrayList<>();
        Stack<Vertex> stack = new Stack<>();
        stack.push(root);
        while (!stack.empty()) {
            Vertex top = stack.pop();
            if (!visited.contains(top)) {
                visited.add(top);
            }
            if (top.vertices != null) {
                for (Vertex vertex : top.vertices) {
                    stack.push(vertex);
                }
            }
        }
        visited.forEach(vertex -> System.out.println(vertex.cell));
    }

    private static class Graph {
        private final Vertex root;

        Graph(Vertex root) {
            this.root = root;
        }
    }

    private static class Vertex {
        private final String cell;
        private Vertex[] vertices;

        private Vertex(String cell) {
            this.cell = cell;
        }

        static Vertex build(String cell) {
            return new Vertex(cell);
        }

        public Vertex setConnection(Vertex... vertices) {
            this.vertices = vertices;
            return this;
        }
    }
}
```
