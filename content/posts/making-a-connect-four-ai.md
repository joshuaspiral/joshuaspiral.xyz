---
title: "Making a Connect Four AI"
date: 2022-03-30T11:45:01+08:00
description: "A devlog on creating a connect four AI"
author: "Joshua"
tags: ["programming", "ai", "c"]
category: Devlog
draft: false
---
## DISCLAIMER
Things in this article may be incorrect, and I do not know a lot about this topic. If there are any mistakes, please do not hesitate to correct me.

## Introduction

I was tasked with creating a 2D array game for my computer science course, and I went with Tic Tac Toe. Soon after, I wrote an AI for it using the [minimax](https://en.wikipedia.org/wiki/Minimax) algorithm. I wrote it in Python because it was quick to put together. However, I wanted speed, so I wrote a C program, [TicTacC](https://github.com/joshuaspiral/tictacc). After having written the AI, I had to create an AI for either [Connect Four](https://en.wikipedia.org/wiki/Connect_Four), [Othello/Reversi](https://en.wikipedia.org/wiki/Reversi), or the sliding game [15](https://en.wikipedia.org/wiki/15_puzzle). I picked Connect Four because I had the most experience with the game. I wrote a prototype of the basic game logic in Python, [here is the repo](https://github.com/joshuaspiral/connectfourpy) then rewrote it in C. In this devlog, I document my journey in creating a faster and smarter Connect Four AI with several layers of optimisations.


## What is Connect Four?

Connect Four, or otherwise known as Four in a Row, is a two player board game where there is a 6x7 vertical grid, where players take turns dropping pieces down columns. To win, a player must connect four pieces of their color together, either vertically, horizontally, or diagonally. 

![Gameplay of Connect Four - Wikipedia](https://upload.wikimedia.org/wikipedia/commons/thumb/a/ad/Connect_Four.gif/220px-Connect_Four.gif)

## Checking for a win

My original game would store the board as a 2D character array, with each cell either holding an UNCLAIMED, YELLOW or RED enum variant.

```c

enum State {

    YELLOW,

    RED,

    UNCLAIMED

};

enum State board[HEIGHT][WIDTH] = {

    {UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED},

    {UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED},

    {UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED},

    {UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED},

    {UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED},

    {UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED, UNCLAIMED},

};

// I know this is not the best way to do this.

```

I would then loop through the board and check if the `Board[i + offset][j + offset] == Board[i + offset + 1][j + offset + 1]` and so on.  

```c

    for (int i = 0; i < HEIGHT - 3; i++) {

        for (int j = 0; j < WIDTH - 3; j++) {

            /* Primary diagonal check (/) */

            if (board[i][j] == board[i + 1][j + 1] && 
                board [i + 1][j + 1] == board[i + 2][j + 2] && 
                board[i + 2][j + 2] == board[i + 3][j + 3] && 
                board[i][j] != UNCLAIMED) {

                if (board[i][j] == piece) {

                    wins++;

                } else {

                    return LOSE;

                }

            }

        }

            

    }

c4c > ai.c > evaluateBoard()

```

This is not a very elegant solution as has a time complexity of `O(n^2 * 5)` (horizontal, vertical, primary diagonal, secondary diagonal, and checking for draws). However, without much research, you can find out better ways to check for a win that are constant time!

## The minimax algorithm

The [minimax algorithm](https://en.wikipedia.org/wiki/Minimax) is one of the most popular decision rule algorithm for games like Tic Tac Toe, Connect Four, and Chess where there are two players taking alternate moves. I wouldn't be able to explain it any better than a YouTube video, so I would recommend [this one](https://www.youtube.com/watch?v=l-hh51ncgDI) by Sebastian Lague.  

The algorithm prioritises certain permutations, set by your evaluation function and heuristic scoring function. The depth passed into the algorithm dictates how many moves ahead the algorithm should check. The algorithm minimises its losses by picking the moves that would result in an advantage for the AI, if there is a winning scenario, it picks that as its choice. There are many optimisations for the algorithm, such as alpha-beta pruning which prunes trees that are pointless to evaluate, and move ordering, which is an extension to alpha-beta pruning to sort the moves so there is a higher chance of pruning. 

## Bitboards

Another optimisation for the minimax algorithm (very common in chess) is by using [bitboards](https://en.wikipedia.org/wiki/Bitboard). Bitboards are binary representations of a board for a specific piece. For example, a Tic Tac Toe grid could look like this:

```
------------
| X | O | X |
------------
|   | X | X |
-------------
| O | O |   |
------------

```

There would be two bitboards, one for X and one for O.

```

X bitboard:

101_

011_

000

O bitboard:

010_

000_

110

```

These two, of course, would be a little over a byte, so you would use a type like `uint16_t` to store them. This could look like `uint16_t X = 0b0000000_101_011_000` and `uint16_t O = 0b0000000_010_000_110` (starting 7 bits are just to fill the empty space). Instead of a mess of for loops, you can have unreadable bitwise logic to evaluate the board. We can evaluate whether or not a board has 3 in a row horizontally by bitshifting certain offsets due to the nature of the bitboard:

```

  0b000000_010_000_111

  0b000000_010_000_111 >> 1 = 0b000000_001_000_011

& 0b000000_010_000_111 >> 2 = 0b000000_000_100_001
__________________________________________________

  0b0

```

We can write this expression as `bitboard & (bitboard >> 1) & (bitboard >> 2) != 0)` If this evaluates to `true`, then it is a win.

You can do this for other offsets too. For columns, you would shift by 3 and 6, and for the primary diagonal (/), you can shift by 2 and 4.

```

0 1 2

3 4 5

6 7 8

```

This grid can help with figuring out the "magic" numbers to shift by.

## 1D array bitboard

Before we optimise, let's take a look at the current benchmarks:

| Implementation        | Depth | Time (s)  | Visited nodes |
|-----------------------|-------|-----------|---------------|
| 2D Array Board AI     | 7     | 1.250252  | 6634026       |
| 2D Array Board AI     | 8     | 4.637645  | 46028598      |
| 2D Array Board AI     | 9     | 32.516463 | 314060244     |

Wow, that is still pretty fast. However, I think we can do better.

The main bottleneck on speed is the sheer number of loops. We could cut it to O(n) by using bitboards and bitwise operations to evaluate states. However, I was not very confident with using a full bitboard, so I was recommended to use a "pseudo-bitboard" that still heavily relies on bitwise operations. I would keep each column as an 8-bit integer, and I would ignore the first two rows as the height is only 6

I mainly used [this resource](https://github.com/denkspuren/BitboardC4/blob/master/BitboardDesign.md) for the bitboard implementations, I translated my original design into a "pseudo-bitboard" AI in [ai_bitboard.c](https://github.com/joshuaspiral/c4c/blob/main/archives/ai_bitboard.c). I ran benchmarks again and this time, it was very surprising.

| Implementation        | Depth | Time (s)  | Visited nodes |
|-----------------------|-------|-----------|---------------|
| 2D Array Board AI     | 7     | 1.250252  | 6634026       |
| "Pseudo-bitboard" AI  | 7     | 0.171348  | 6602778       |
| 2D Array Board AI     | 8     | 4.637645  | 46028598      |
| "Pseudo-bitboard" AI  | 8     | 1.180703  | 45414144      |
| 2D Array Board AI     | 9     | 32.516463 | 314060244     |
| "Pseudo-bitboard" AI  | 9     | 1.250252  | 6634026       |

(I actually do not know why the visited nodes are slightly different each time. There may be a difference in the heuristics)

## Real bitboard

After conquering the challenge of building a "pseudo-bitboard", I felt ready to write the code for a real bitboard. I spent a while drawing the board out on paper and trying a few bitwise operations to aid the functionality of the dropPiece(), popPiece(), evaluateBoard() functions. I ended up finishing all the code in a little under 1 hour because of my previous experience with the "pseudo-bitboard". The code is in [ai_realbitboard.c](https://github.com/joshuaspiral/c4c/blob/main/archives/ai_realbitboard.c) (I know, not a very good name).

| Implementation        | Depth | Time (s)  | Visited nodes  |
|-----------------------|-------|-----------|----------------|
| 2D Array Board AI     | 7     | 1.250252  | 6634026        |
| "Pseudo-bitboard" AI  | 7     | 0.171348  | 6602778        |
| Bitboard AI           | 7     | 0.051276  | 6601433        |
| 2D Array Board AI     | 8     | 4.637645  | 46028598       |
| "Pseudo-bitboard" AI  | 8     | 1.180703  | 45414144       |
| Bitboard AI           | 8     | 0.384628  | 45386033       |
| 2D Array Board AI     | 9     | 32.516463 | 314060244      |
| "Pseudo-bitboard" AI  | 9     | 8.299119  | 310985082      |
| Bitboard AI           | 9     | 2.373259  | 310528499      |

[Implementation](https://github.com/joshuaspiral/c4c/blob/main/archives/ai_realbitboard.c)

## Closing comments
Since this is quite *overoptimised* for a mere Connect Four AI, maybe this is where I should stop. However, there are some extra things you could try to optimise.
- Transposition table - this caches all board positions, enabling you to skip evaluating already visited permutations (this actually didn't work so well for my implementation as the evaluation function was faster than inserting new grid entries but I could have overcome the problem through preallocating the memory)
- Move ordering - more efficiently ordering the evaluation, pruning more branches in the game tree
- Heuristic optimisations - creating smarter moves for the AI to prioritise

## Very useful information when writing this program
When researching this topic, I found two very useful articles about Connect Four AIs and bitboards. They are listed below:
- [Pascal Pons' 11 part series on creating a Connect Four AI](https://blog.gamesolver.org/solving-connect-four/01-introduction/)
- [A guide on bitboards implemented for Connect Four](https://github.com/denkspuren/BitboardC4/blob/master/BitboardDesign.md)
