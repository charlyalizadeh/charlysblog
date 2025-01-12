+++
title = 'Tictacchess'
date = 2024-01-03T10:45:25+01:00
draft = false
+++

[Github](https://github.com/charlyalizadeh/TicTacChess)

{{ $tictacchess := .Resources.Get "tictacchess.png" }}
<img src="{{ $tictacchess.RelPermalink }}" width="{{ $tictacchess.Width }}" height="{{ $tictacchess.Height }}">

In this project I built an artificial intelligence that can play the game of Tic Tac Chess thanks to the [Minimax algorithm](https://en.wikipedia.org/wiki/Minimax) and [bitboards optimization](https://www.chessprogramming.org/Bitboards).


## Rules of TicTacChess

The TicTacChess is a variation of the TicTacToe which includes chess pieces instead of crosses and circles. It's played by 2 players on a 4x4 board, each player has four pieces including a pawn, a knight, a bishop and a rook. Each player play successively by either moving a piece, eating a enemy piece or placing a dead piece on an empty square of the board. A party is over when one of the player aligned all its pieces in a horizontal, vertical or diagonal line.

## Bitboard

A bitboard is an integer representing a board. In regular chess a board has a dimension of `8 * 8 = 64` which is, when we think about it, kind of a big coincidence with the fact that modern computer are based of 64 bits processors. In a bitboard integer, when a bit is set at the index `n` it means that a piece occupies this square.
In the following board an upper letter represents a white piece and a lower letter a black piece, next to it you can see a bitboard representing its occupancy (meaning that we can only retrieve information about whether a square is occupied or not but not by which piece).
```
 ┏━━━┳━━━┳━━━┳━━━┳━━━┳━━━┳━━━┳━━━┓
8┃   ┃   ┃   ┃   ┃   ┃   ┃   ┃   ┃
 ┣━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━┫
7┃ R ┃   ┃   ┃   ┃   ┃ p ┃ k ┃   ┃
 ┣━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━┫
6┃   ┃   ┃   ┃   ┃   ┃ p ┃   ┃   ┃
 ┣━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━┫
5┃   ┃ p ┃   ┃ r ┃   ┃   ┃   ┃ p ┃
 ┣━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━┫
4┃   ┃   ┃   ┃   ┃   ┃   ┃   ┃   ┃  8 00000000
 ┣━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━┫  7 10000109
3┃ P ┃   ┃   ┃   ┃   ┃ K ┃ P ┃   ┃  6 00000100
 ┣━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━┫  5 01010001
2┃   ┃   ┃   ┃   ┃   ┃ P ┃   ┃ P ┃  4 00000000
 ┣━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━╋━━━┫  3 10000110
1┃   ┃   ┃   ┃   ┃   ┃   ┃   ┃   ┃  2 00000101
 ┗━━━┻━━━┻━━━┻━━━┻━━━┻━━━┻━━━┻━━━┛  1 00000000
   a   b   c   d   e   f   g   h      abcdefgh
```

```
                            00000000<--End of the integer
                            10000110
                            00000100
                            01010001
                            00000000
                            10000110
                            00000101
Begginging of the integer-->00000000
```

To store all the information about a board we actually need multiple bitboards, one for each piece type/color.

In the case of TicTacChess we doesn't need a full 64 integer to represent a 4x4 board. Our board has 16 squares but we cannot use a 16 bits integer, indeed when computing if four pieces are aligned we use bitshifting and AND operator like so:
```
0000   0000   0000   0000   0000
1111   0111   0011   0001   0001
0000 & 1000 & 1100 & 1110 = 0000
0000   0000   0000   0000   0000
```
In this case we don't have any problems, but let's analyse the following board:
```
0001   0000   0000   0000   0000
1110   1111   0111   0011   0010
0000 & 0000 & 1000 & 1100 = 0000
0000   0000   0000   0000   0000
```
We can see in the original board is not a winning board however our algorithm will detect a win. To overcome this issues we had a "ghost" column:

```
00010   00001   00000   00000   00000
11100   01110   10111   01011   00000
00000 & 00000 & 00000 & 10000 = 00000
00000   00000   00000   00000   00000
    ^
    |
Ghost column
```

So in order to use board with a ghost column we need to take the size above 16 bits integer which is 32 bits integer, here an example:

```
 ┏━━━┳━━━┳━━━┳━━━┓
4┃   ┃   ┃   ┃   ┃
 ┣━━━╋━━━╋━━━╋━━━┫
3┃ P ┃   ┃ p ┃   ┃
 ┣━━━╋━━━╋━━━╋━━━┫
2┃   ┃ k ┃ R ┃   ┃  4 00000000000000000
 ┣━━━╋━━━╋━━━╋━━━┫  3 10100
1┃   ┃ B ┃   ┃   ┃  2 01100
 ┗━━━┻━━━┻━━━┻━━━┛  1 01000
   a   b   c   d      abcd
```

## Magic Bitboard

### Introduction

I'll use the following representation of a chess board to illustrate the magic bitboard:
```
rhbqkbhr
pppppppp
........
........
........
........
PPPPPPPP
RHBQKBHR

Description:
    r: black rook
    h: black knight
    b: black bishop
    q: black queen
    k: black king
    p: black pawn

    R: white rook
    H: white knight
    B: white bishop
    Q: white queen
    K: white king
    P: white pawn

    .: empty cell
```

And when I'll talk about bitboard more specifically I'll use the following representation:
```
                         11111111 <-- end of the integer
                         11111111
                         ........
                         ........
                         ........
                         ........
                         11111111
start of the integer --> 11111111

where . corresponds to a 0 in the 64-bits integer
```

### Why

One feature of a chess engine is to know where a piece can move in function of the board state. One category of pieces is called "sliding piece", this category includes the rooks, the bishops and the queen. All those pieces have the particularity that they can slide on board "indefinitely" until another piece or the border of the board blocks it.

This characteristic makes their possible moves hard to generate. But thanks to some passionate peoples several techniques have been created that allow for fast computation of the possible moves of a sliding piece given its type, position and board state. In this tutorial we'll focus mainly on the technique called "magic bitboard".

### How


#### Key space reduction

The idea behind magic bitboard is to create a map for each combination of piece type and position that contains all the possible moves for all the combination of board state. You can see a magic bitboard as a function that takes as input the piece type/position and the board state and returns the legal move for this piece.

```
magic(piece_type, position, board_state) = legal_moves
```

Or more visually:

```
              ........  ........    ........
              ........  ........    1.......
              ........  .....1..    .1...1..
              ........  ........    ..1.1...
magic(bishop, ...1...., ........) = ........
              ........  ........    ..1.1...
              ........  ........    .1...1..
              ........  ........    1.....1.

Note: we don't care about the piece color when using magic bitboard.
Indeed if the other piece on the board is of the same color we only
need to remove the same piece color from the legal_moves (using bit operation).

In this example we would obtain:

........
1.......
.1......
..1.1...
........
..1.1...
.1...1..
1.....1.

```

As you can expect we can not store all the possible combination of piece type, position and board state in the keys of map, there are way to many of them. There are two observations we can make that'll let us reduce the key space drastically:

* We only need to take into account pieces that are on the trajectory (also called attacks) of the piece
* Some combination of piece type, position and board state have the same legal moves (see example bellow)


```
              ........  ........    ........
              ........  ........    1.......
              ........  .....1..    .1...1..
              ........  ........    ..1.1...
magic(bishop, ...1...., ........) = ........
              ........  ........    ..1.1...
              ........  ........    .1...1..
              ........  ........    1.....1.


              ........  ........    ........
              ........  .....1..    1.......
              ........  ....1...    .1...1..
              ........  ........    ..1.1...
magic(bishop, ...1...., ........) = ........
              ........  ........    ..1.1...
              ........  ........    .1...1..
              ........  ........    1.....1.


              ........  .......1    ........
              ........  ........    1.......
              ........  .....1..    .1...1..
              ........  ........    ..1.1...
magic(bishop, ...1...., ........) = ........
              ........  ........    ..1.1...
              ........  ........    .1...1..
              ........  ........    1.....1.
```

In order to use the first observation at our advantage we use [bit masking](https://en.wikipedia.org/wiki/Mask_(computing)) to get only the relevant pieces of board.

For example let's take the following board with a bishop on d4 (`1` denotes a piece, we don't care which one in this example):

```
8 ........
7 ........
6 .....1..
5 ...1....
4 .1.b....
3 ........
2 ...1...1
1 .1....11
  abcdefgh
```

On this board the only relevant pieces in order to get the bishop moves are the ones on its diagonals, i.e:

```
8 .......1
7 1.....1.
6 .1...1..
5 ..1.1...
4 ........
3 ..1.1...
2 .1...1..
1 1.....1.
  abcdefgh
```

To get the board with only the relevant pieces we use the bitwise `and` operator (also denoted `&` in C/C++)


```
 .......1     ........   ........
 1.....1.     ........   ........
 .1...1..     .....1..   .....1..
 ..1.1...     ...1....   ........
 ........ and .1.b.... = ........
 ..1.1...     ........   ........
 .1...1..     ...1...1   ........
 1.....1.     .1....11   ......1.
```

Another observation we can make is that a piece that is on the border of the board doesn't have any impact on the possibles moves.

```
              ........  ........    ........
              ........  ........    1.......
              ........  .....1..    .1...1..
              ........  ........    ..1.1...
magic(bishop, ...1...., ........) = ........
              ........  ........    ..1.1...
              ........  ........    .1...1..
              ........  ........    1.....1.


              ........  ........    ........
              ........  ........    1.......
              ........  .....1..    .1...1..
              ........  ........    ..1.1...
magic(bishop, ...1...., ........) = ........
              ........  ........    ..1.1...
              ........  ........    .1...1..
              ........  ......1.    1.....1.
```

So we only need a mask that stop at one square before the borders.

```
 ........     ........   ........
 1.....1.     ........   ........
 .1...1..     .....1..   .....1..
 ..1.1...     ...1....   ........
 ........ and .1.b.... = ........
 ..1.1...     ........   ........
 .1...1..     ...1...1   ........
 ........     .1....11   ........
```

Note that for the rook this property is only applied to the end of its rays.


```
 ........     1..1....   ........
 1.......     ........   ........
 1.......     .....1..   ........
 1.......     ........   ........
 .111111. and r...1... = ....1...
 1.......     ........   ........
 1.......     1..1...1   1.......
 ........     .1....11   ........
```


#### The magic bitboard structure

We talked a lot about reducing the key space of our map, now we'll talk about the details of how we use a magic bitboard.
One magic bitboard corresponds to one piece at one position. A magic bitboard is composed of:

* **A magic number**: which is a 64-bits integer
* **A database** (array) containing all the possible move for a specific piece at a specific position.
* **The shift** which is an integer inferior to 64

With those three components we can retrieve the moves for a piece thanks to the following formula (using C++ notation):

```
moves = database[((board & mask) * magic) >> (64 - shift)]
```

A lot is going in this formula so let's break it down.

* `(board & mask)`: this part allows to retrieve only the relevant pieces from the board as explained in the previous part
* `(board & mask) * magic`: here is the tricky part, we multiply our board by some magic constant (constant by piece/position) in order to get the index of our array on the "top" of our integer (don't worry if you don't understand for the moment, an example is coming)
* `(board & mask) * magic >> (64 - shift)`: The index in the `database` array of the move is now on the top of `(board & mask) * magic`, so we need to bring it down in order to get its value, the value of shift depends on the quality of the *magic number*
* `database[((board & mask) * magic) >> (64 - shift)]`: now that we have our index we only need to retrieve the corresponding moves in the database array


Let's take our bishop in d4:
```
8 ........
7 ........
6 .....1..
5 ...1....
4 .1.b....
3 ........
2 ...1...1
1 .1....11
  abcdefgh
```

First we use a bit mask to get the relevant pieces:

```
 ........     ........   ........
 ........     1.....1.   ........
 .....1..     .1...1..   .....1..
 ...1....     ..1.1...   ........
 .1.b.... and ........ = ........
 ........     ..1.1...   ........
 ...1...1     .1...1..   ........
 .1....11     ........   ........
```

Now we multiply it by the magic number (here we don't specify it, we'll explain how to find it later) which gives use this imaginary result:

```
........           11.1....
........           ........
.....1..           ........
........           ........
........ * magic = ........
........           ........
........           ........
........           ........
```

Now let's imagine that we have `shift = 4`. The third step gives us the following result:

```
11.1....         ........
........         ........
........         ........
........         ........
........ >> 60 = ........ = 13
........         ........
........         ........
........         ....11.1
```

Then in the `database` array at the index `13` we have:

```
               ........
               1.......
               .1...1..
               ..1.1...
database[13] = ........
               ..1.1...
               .1...1..
               1.....1.
```

Which corresponds to the possible moves for our bishop in d4.

#### Building the magic bitboard

Pseudo code of the algorithm used to find magic bitboard:

{{ $find_magic := .Resources.Get "find_magic.png" }}
<img src="{{ $find_magic.RelPermalink }}" width="{{ $find_magic.Width }}" height="{{ $find_magic.Height }}">

Again there's a lot going on here so let's break it down.
First we find all the possible setup of blockers of the given piece and position. Then we need to know on how many bits the index given after multiplying by the magic and shifting will be coded. It's `log2(number of blockers possition) + 1`, note that `log2` gives a float output, in order to be sure that we have enough place for all the possible moves of blockers we add 1 to this value and then remove the decimal part (ex: `log2(5) = 2.3219` so the `bits = 3` and we're sure to have enough place for all the moves). We then initialize `database` and `magic` to default values. `failed` is a boolean that is set to `true` if the magic currently tested is not valid.

In the loop part we test all the possible block boards and check if the current `magic` is valid. A `magic` number is valid for a given block board if at the iteration we test the block board in the `for` loop the `database` array at the index `board * magic >> (64 - bits)` is either 0 or the corresponding move board.

Then if the current `magic` is not valid we change it to another random value and clear the `database`.

This algorithm doesn't look very smart because we test magic number randomly but finding a valid magic is fast and only done once at the start of the engine.

## Credits

* [rust-sfml](https://github.com/jeremyletang/rust-sfml)
* [Chess Programming Wiki](https://www.chessprogramming.org/Main_Page)
* [Binary converter](https://www.binaryhexconverter.com/binary-to-decimal-converter)
* [YT video on Bitboard](https://www.youtube.com/watch?v=MzfQ8H16n0M&)
* [Some blog about chess programming](https://peterellisjones.com/posts/generating-legal-chess-moves-efficiently/)
* [Paper on Magic Bitboard](http://pradu.us/old/Nov27_2008/Buzz/research/magic/Bitboards.pdf)
* [Article on Magic Bitboard](https://essays.jwatzman.org/essays/chess-move-generation-with-magic-bitboards.html)
* [Chess Programming wiki page on magic bitboard](https://www.chessprogramming.org/Magic_Bitboards)
* [Article on Magic bitboard](http://www.vicki-chess.blogspot.com/2013/04/magics.html)
