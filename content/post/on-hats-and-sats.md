---
title: "The Hat, the Spectre and SAT solvers"
subtitle: "An invitation to SAT solvers"
date: 2024-01-27T09:21:40.557Z
Description: ""
Tags: []
Categories: []
DisableComments: false
---
# Introduction

In this blog post you are going to read about two things:

* A new flashy discovery in mathematics: aperiodic tilings of the plane with a single monotile
* SAT solvers. A family of not so well known algorithms in computer science

Hopefully by the end of the post you will know a fair amount about the hat, the turtle and the spectres and have another powerful tool under your belt, SAT solvers.

Thus, you can see this post either as an exercise in recreational mathematics or as an invitation to SAT solvers.

The post is organized as follows. We first introduce the Hat and state the main result. Then we define what SAT solvers are. In section 3 we describe how we use the solver in wasm and apply it to the problem of solving a Sudoku problem as a warm up exercise in the next section. In section 5 we explain Craig Kaplan's idea of how to use SAT solvers to tile finite regions of the plane and apply it to the Hat. Section 6 introduces the Turtle, a second monotile capable of tiling the plane aperiodically. With all those constructions we build the Spectre, from deformations of Hats and Turtles, a true chiral aperiodic monotile. In the last section we explain how to use the [app](https://www.nhatcher.com/hats/)

# Code

You can find all the code discussed here in [GitHub](https://github.com/nhatcher/hats).

There is an app you can play along with at:

https://www.nhatcher.com/hats/

The sudoku solver mentioned below is deployed at:

https://www.nhatcher.com/hats/sudoku.html

A small app solving general SAT problems in the browser:

https://www.nhatcher.com/hats/sat.html

I used the fantastic [dplr](https://github.com/shnarazk/splr) SAT Solver by [Shuji Narazaki](https://github.com/shnarazk) and adapted it to wasm in:

https://github.com/nhatcher/splr-wasm

# 1. The Hat: a shape perplexing the world

It is quite unusual that a discovery in mathematics is understandable by the general public, but 2023 witnessed precisely that.

In November of 2022 just as ChatGPT was going to take world by a storm, [David Smith](https://en.wikipedia.org/wiki/David_Smith_(amateur_mathematician)) a retired printer technician and amateur mathematician found a shape:

![the hat](/images/hats/hat.png "The Hat")

Thats is able to _tile_ the whole infinite plane in a non periodic way:

![tilling](/images/hats/simple-tilling.png "simple tilling")

The announcement was made in a [paper](https://arxiv.org/abs/2303.10798) but you might have heard of it in any of your favorite channels like [veritasium](https://www.youtube.com/watch?v=48sCx-wBs34), [numberphile](https://www.youtube.com/watch?v=_ZS3Oqg1AX0), 
[quanta magazine](https://www.quantamagazine.org/hobbyist-finds-maths-elusive-einstein-tile-20230404/) or even the 
[New York Times](https://www.nytimes.com/2023/03/28/science/mathematics-tiling-einstein.html).

The story of non periodic tilings of the plane is somewhat old and was popularized by Sir Roger Penrose and Martin Gadner in his famous Scientific American column on recreational mathematics. For example Penrose found [two tiles](https://en.wikipedia.org/wiki/Penrose_tiling) (the dart and the kite) that were able to tile the plane in a non periodic way.

In this blog post I will write a bit about these new tilings of the plane and how to cover finite regions using an algorithm that not many people, even people in the industry, know: a SAT solver.

# 2. The SAT solver, an underappreciated tool.

A [SATisfiability solver](https://en.wikipedia.org/wiki/SAT_solver) is a computer program that solves problems in boolean algebra. Say you have a number of variables `a`, `b`, `c` , `d`, ... that can be either **true** or **false**. And then you have a series of statements regarding those variables, like:

* If `a` is **true** then either `b` is **true** or `c` is **false**.
* If `a` and `b` are both **true** if and only if `d` or `c` is false.

Solvers can deal with problems with a lot of variables. We will run some experiments in your browser with more than 60 thousand variables and millions of statements.

To be able to use the solver we first need to convert all of our logic statements in what is called a **[conjunctive normal form](https://en.wikipedia.org/wiki/Conjunctive_normal_form)** (CNF). We don't really need to explain much in here because as it happens the statements that we will produce are exactly in this form already.

In _conjunctive normal form_ we have a series of statements, called _clauses_, that all need to be true (those are the conjunction or AND blocks). Each clause is composed of an OR array of literals (variables and their negations). For example:

* a or b or not c
* a or d
* not c or d

This is composed of 3 clauses and there are 4 variables (a, b, c , d).

There is a general constructive theorem called the _Tseitin transformation_ that converts any problem in boolean logic to an equivalent in CNF.

As I said, in our case the problem will already be in CNF so we don't need to bother about that. The next step will be to write the problem in a way a Solver can digest it. The standard way is to assign a positive to each variable. For instance

* a -> 1
* b -> 2
* c -> 3
* d -> 4

The each clause is an array:

* a or b or not c <=> [1, 2, -3]

As a you can see a negative value means negation of the variable.

Once you feed the solver an array of arrays it returns an array like [1, -2, 3, -4] that means variable 1 and 3 are true and variables 2 and 4 are false. The solver might not find a solution and will return an error or might be able to prove that there is not a valid a solution and will return "UNSAT".

# 3. A SAT Solver in wasm

There are a million SAT solvers out there but we will use [splr](https://github.com/shnarazk/splr) that we can compile to [WASM](https://github.com/nhatcher/splr-wasm), fits into a 200kB file and runs in the [browser](https://www.nhatcher.com/hats/sat.html).

Since we are communicating with wasm instead of passing an array of arrays to the solver we will pass just one array from which you can rebuild the array of arrays:

```
[(length, a_1, ..., a_l)*]
```

So, for instance the array of arrays:

```
[
    [5, -4, 7],
    [3, -5, 89, 3]
]
```

Will be converted to:
```javascript
const array = [3, 5, -4, 7, 4, 3, -5, 89, 3]
```

Once you have the array you pass it to the splr solver by

```javascript
const result = JSON.parse(solveSat(array));
```

The `result` will either be successful and contain a `data` array or fail and have a `details` message. If it succeeds the `result.data` will be an array with all the numbers of the variables being either positive (meaning they are true) or negative (meaning they are false).

To sum up if you want to solve a problem in boolean logic you need:

* Step 1: Convert the problem to a CNF
* Step 2: Number all variables
* Step 3: Create the array of array of clauses
* Step 4: Convert them to a single array and feed it to `solveSat`
* Step 5: Read the output from `result.data`
* Step 6: Convert each of the indices to variables

# 4. Intermezzo: Using a SAT solver to solve Sudokus

Instead of jumping right at the main issue let's use the algorithm to solve a _different_ problem.

Let's solve a [sudoku](https://en.wikipedia.org/wiki/Sudoku):

![sudoku](/images/hats/sudoku_wikipedia.svg "A sudoku board")

There are 81 squares on a sudoku board and any number from 1 to 9 can be in any of those squares.

Our variables are:

* The is a number `n` in square (row, column)

where n in [1, .., 9] and row, column in [0, .., 8]

There are 9x9x9 = 729 variables in total and only 81 of them can be true.
We need to index each variable, so the previous statement ("There is an n in square (row, column)") is the variable number:

```
index = row*9*9+column*9+n
```

We also need a way to get (row, column) and n from the index:

```javascript
const fromIndexToCell = (index) => {
    const row = Math.floor((index - 1) / 81);
    const column = Math.floor((index - 1 - row * 81) / 9);
    const n = index - row * 81 - column * 9;
    return { row, column, n };
}
```

Now we need to create all the clauses. For instance, One of the following must be true 
 * number 1 is in (row=0, column=0)
 * number 1 is in (row=0, column=1)
 ...
 * number 1 is in (row=0, column=8)
 That's one clause. We have 9 of those, one for each number `n`. But of course we have 9 rows. So we have 81 of those.

 We can do the same in columns, so we have 81 clauses for columns.

 We should do that again for every 3x3 block, another 81 clauses.

 That's 81x3=243 clauses.

 We still need more clauses. One solution of all the previous clauses is make all of them true.

 On each square there can only be a number. For instance:
 * NOT(number 1 is in (row=3, column=4))
 * NOT(number 2 is in (row=3, column=4))

 For each spot there are (9x8)/2=36 clauses (combinations of 9 elements taken 2 by 2, order is not important).
 There are 81 spots so that is a total of 81x36=2916 clauses of this kind for a grand total of:

 2916 + 243 = 3159

 Clauses. That is, of course for an empty sudoku board. We will need to add the known numbers. In the diagram above:

 * There is a 5 in position (row=0, column=0)
 ...
 * There is a 9 in position (row=8, column=8)

 That is 30 more clauses for that particular puzzle. For a general sudoku with a single solution it is known that you need at least 17 hints.

 If you want to see the details look at the [source code](https://github.com/nhatcher/hats/blob/main/sudoku.js) to see how you can generate all those in just a few lines of simple code.

 Now, using a SAT solver to find solutions to sudokus might not be the most performant algorithm. Also the most important bit is how you _generate_ solutions. I enjoyed reading this [piece](https://eli.thegreenplace.net/2022/sudoku-go-and-webassembly/) from the beautiful blog of Eli Bendersky.
 
# 5. Using a SAT solver to find solutions to the tiling problem

In his book "The Art of Computer Programming" in volume 4B Donald Knuth proposes as an (easy) exercise to the reader to find solutions to _tatami tilings_ using a SAT solver.

Besides that, it was Craig Kaplan's paper of the [Heesch Numbers of Unmarked Polyforms](https://arxiv.org/abs/2105.09438) that used SAT solvers extensively to solve tiling problems (see also his [blog post](https://isohedral.ca/heesch-numbers-of-unmarked-polyforms))

Given a shape (like the Hat above) or a plolyomio below:

![Fontaine](/images/hats/Fontaine.svg "Fontaine Polyomio")

The [Heesch number](https://en.wikipedia.org/wiki/Heesch%27s_problem) is the number of times you can surround
 the original shape with copies of it without gaps or overlaps. For instance the [Fontaine polyomino](https://core.ac.uk/reader/82160310)

![Fontaine Heesch](/images/hats/Fontaine_heesch_2.svg "Fountaine Heesch")
(credit wikipedia)

A [polyomino](https://en.wikipedia.org/wiki/Polyomino) is a shape that is made from squares of the same size (like tetris figures or _tetraminoes_). When you are trying to tile the plane with polyominos the vertices can only be on a square grid. This converts the tiling problem of a finite part of the plane into a combinatorial problem and from there Kaplan was able to produce a set of clauses for a SAT solver.

The particular shape that David Smith found, the Hat, is not a polyomino but one of it's vertices must be in the center of an hexagon of a regular hexagonal grid:

![the marked hat](/images/hats/hat-marked.png "The marked Hat")

You can go ahead and play with it at the [app](https://www.nhatcher.com/hats/).

Note also a very important fact of the Hat. In it composed of a series of 8 "kites":

![the hat with kites](/images/hats/hat_with_kites.svg "The Hat with Kites")

Much like a polyomino is a [polyform](https://en.wikipedia.org/wiki/Polyform) of squares, the Hat is a polyform of [kites](https://en.wikipedia.org/wiki/Kite_(geometry)). This fact will be extremely useful for us.

You can rotate the shape and all the vertices of the Hat will still be on the grid. There are in total 6 possible rotations. But you can also have the mirror image of the Hat (the anti-Hat), in the app just press the space bar to get an anti-Hat. In each center of an hexagon you can have any of the 12 possibilities:

* Any of the 6 rotations of the Hat
* Any of the 6 rotations of the anti-Hat

In a grid of `M` columns and N rows we have `12×M×N` variables:

* There is Hat (or an anti-Hat) in vertex (m, n) with rotation `60×i` (with i=0,..,5)

If we solve for all those variables, i.e. if we decide which of those statements are true and which are false we will find a tiling of the `N×M` rectangle with Hats and anti-Hats.

All we need to transform this problem into a something a SAT solver can use is to find a set of clauses.

The clauses are:

* There cannot be two tiles in the same location
* Two tiles cannot collide
* Every kite must be occupied

None of those are difficult to build in terms of our variables but some require a bit of geometry.


For a different take on the same problem see [Hastings Greer's post](https://www.hgreer.com/HatTile/) using Microsoft's [z3 theorem prover](https://github.com/Z3Prover/z3).

# 6. It's turtles all the way down

A little after finding the Hat, David Smith found another shape:

![the turtle](/images/hats/turtle.png "The Turtle")

This is called "the Turtle" and it is very similar to the Hat. It also tiles the whole plane in an aperiodic way. All the technology we applied to the Hat, works here as well.

They soon realized that the Hat and the Turtle are all shapes from the same family.

Take a Hat. There are two kinds of sides, some of length `a`, the short ones and some of length `b=a*√3`. there is one side that is `2xa`, we will count that as two sides of length `a`. Now mentally remove the grid behind it and, while preserving the angles, change the length `a` and `b`. All the tiles that you generate that way will also tile the whole plane. They call those tiles `Tile(a, b)`. In this way `Tile(1, √3)` is the Hat and `Tile(√3, 1)` is the Turtle.

Both Hats and Turtles are polykites, but that is not true for other members in the family. What is even more interesting, you can combine Hats and Turtles to tile the plane.

This last fact would prove fundamental for our next step.

You can play in the [app](https://www.nhatcher.com/hats/) with Hats, anti-Hats, Turtles and anti-Turtles, color them differently, and using the SAT solver to create interesting patterns.


# 7. The spectre

Here is where things get a little bit more involved, the Hat, or the Turtle by themselves do not tile the whole plane. you also need the anti-Hat or the anti-Turtle. Some people felt that that is not really a solution to the [einstein problem](https://en.wikipedia.org/wiki/Einstein_problem).

The authors of the Hat realized in a [subsequent paper](https://arxiv.org/abs/2305.17743) that one member of the family, the `Tile(1, 1)` does tile the whole plane in an aperiodic way without the need of an `anti-Tile(1, 1)`. The tile, and its variations were christened the Spectre (technically they call it Tile(1, 1) and call Spectres some variations of it, for us Tile(1, 1) will be the Spectre.):

![the spectre](/images/hats/spectre.png "The Spectre")

The Spectre, however, does not share the nice properties of its siblings the Hat and the Turtle:

* It does not live in an hexagonal grid
* It is not made out of kites (It is actually not a polyform)

We can't really apply a SAT solver directly to this tile. The great insight of the exceptional team is [Theorem 3.1 in the paper](https://arxiv.org/abs/2305.17743):

**Theorem 3.1.** (David Smith, Joseph Samuel Myers, Craig S. Kaplan and Chaim Goodman-Strauss)

    There is a bijection between combinatorially equivalent tilings by Tile(1, 1) and
    by the set {hat, turtle}, such that a Tile(1, 1) tiling has a translation as a symmetry if and only
    if the corresponding hat-turtle tiling has a corresponding translation, and the Tile(1, 1) tiling
    includes a reflected tile if and only if the hat-turtle tiling does.

If you are not mathematically oriented don't run just yet. This is a fairly easy to understand statement once we distill it into common English. If I start with a Hat and change the length of it's sides until they are all the same I will get the Spectre. The inverse transformation, that is starting with the Tile(1, 1) you can change the length of some of the sides and get either a Hat, an anti-Hat, a Turtle or an anti-Turtle. That's all that theorem is saying. The only tricky bit is to know what sides to change. Note that the tiles have edges that are parallel in pairs. The team called some even and some odd depending on the initial rotation of the spectre, then change the even and odd lengths differently.

![paper](/images/hats/paper.png "Deformations of hats and turtles")
(from "A chiral aperiodic monotile")

There is not better way to see that than through direct manipulation on the [app](https://www.nhatcher.com/hats/)

If you start with a [tessellation](https://en.wikipedia.org/wiki/Tessellation) of Hats and Turtles alone (no anti-Hats and no anti-Turtles) and you change the side of the edges to be the same you will end up with only Spectres (and no anti-Spectres!). That's the Einstein, that is the monotile. They got it!

They went even a little bit further. It turns out that the Spectre and the anti-Spectre also tile the plane but they can do it in a periodic way. Although I won't go into the details here, modifying the shape of the sides in a symmetrical way will get you shapes that tile the plane only in non periodic ways and you can't use the anti-Shape.


# 8. Using the Hats app

You might be using the app either to follow along the blog post and understand little bits amd pieces here and there.

The app is composed of a toolbar with some minimal options, the full canvas and a status bar with some info on what is selected

## Adding tiles manually

You can add as many tiles manually as you want. If you have a keyboard, click 'A' to add a new tile, 'Space' to cycle through the different types of tiles (Hats, anti-Hats, Turtles and anti-Turtles), 'R' to rotate a shape. You can click on 'Help' to get this and further instructions. There are also 'On Screen Controls' if you don't have a keyboard.

Adding tiles manually is a fun way to try to tile the plane by yourself.

## Tiles menu

In the _tiles_ menu you can select what tiles to use and pick different colors for different orientations of each tile.

## Grid menu

Here you can choose the number of columns in the grid, and wether or not to show the subjacent hexagon grid.

## Solver menu

Once you are happy with a pattern that you have build manually you can use the SAT solver to fill the rest of the available area. If you check "Randomize" the order of the clauses will be randomized before sending them to the solver (As far as I now there is not an option for randomized solutions in this particular solver)

Modern laptops or even phones can solve in a few seconds problems with ~35 columns and two different tiles. A good laptop can solve problems with ~75 columns and the four tiles in an hour or so. This amounts to 5.5 million clauses!

You can use the app to test if a particular pattern can appear in the full tilling. For instance can four Hats:

![four hats](/images/hats/fourhats.png "Four Hats")

appear in a full tiling of the plane. The answer is no :)

## Spectres

To draw the spectres and other members in the family `Tile(a , b)`. Go to the "Spectres" menu and click the checkbox "Draw Spectres". Remember that spectres and other elements in `Title(a, b)` family do not belong in a grid so there is no straightforward way to draw them. We use a recursive algorithm:

* Start drawing one tile.
* If at a particular vertex there are other tiles, draw them first
* Continue until you are done.

That will draw a connected component. If you have several disjoint tiles the algorithm will only draw one. Also if the tiles are in a wrong position the algorithm will produce weird results:

![wrong tiles](/images/hats/wrong_tiles.png "Wrong tiles")
![wrong spectres](/images/hats/wrong_spectres.png "Wrong spectres")

But if you do everything alright you should find a nice tiling with tiles of your chosen family.

Note that in general you will have up to 4 different kinds of tiles for each tilling. But when you choose spectres you will have only two (the `Tile(1, 1)` and the `anti-Tile(1, 1)`), This is because both the Hat and the Turtle transform into the spectre. If you start tilling with Hats and Turtles (no ani-Hats and no anti-Turtles):

![select hats and turtles](/images/hats/hats-and-turtles.png "Select Hats and Turtles")

![hats and turtles](/images/hats/hats-and-turtles-tiling.png "Hats and Turtles Tiling")

and you transform them to Spectres you will end up with:

![spectres](/images/hats/hats-and-turtles-spectres.png "Spectres")

Note that in this last image, all tiles are the same but with different orientations, there are no mirror images, and the tiling is aperiodic.


# 9. References
[quanta magazine](https://www.quantamagazine.org/math-that-goes-on-forever-but-never-repeats-20230523/)


Original sources:
[An aperiodic monotile](https://cs.uwaterloo.ca/~csk/hat/)
[A chiral aperiodic monotile](https://arxiv.org/abs/2305.17743)


[borealis tutorial](https://www.borealisai.com/research-blogs/tutorial-9-sat-solvers-i-introduction-and-applications/)


Applications:

Craig Kaplan:
[patches of hats](https://cs.uwaterloo.ca/~csk/hat/app.html)
[H7/H8 substitution rules](https://cs.uwaterloo.ca/~csk/hat/h7h8.html)
[GitHub](https://github.com/isohedral/hatviz)


[Mathigon polypad](https://mathigon.org/polypad/8kVqVH2Mor6JTQ)


SAT Solver online:
[cryptominisat](https://msoos.github.io/cryptominisat_web/)
[GitHub](https://github.com/msoos/cryptominisat)

Craig S. Kaplan uses this solver (Heesch Numbers of Unmarked Polyforms):
[blog post](https://isohedral.ca/heesch-numbers-of-unmarked-polyforms/)
[paper](https://arxiv.org/pdf/2105.09438.pdf)
