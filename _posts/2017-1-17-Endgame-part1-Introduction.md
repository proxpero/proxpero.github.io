---
layout: post
title: The Beginning of the End
comments: false
mathjax: false
ganalytics: true
---

### Endgame

I've been working on a chess framework in Swift called [Endgame]. Among other things, it validates moves, stores move history, encodes games into and out of [pgn]. Basically all the things you would expect a chess program to do, except for AI, which I'm going to leave off to one side for now.

[Endgame]: https://github.com/proxpero/Winchester/tree/master/Endgame
[pgn]: http://www.saremba.de/chessgml/standards/pgn/pgn-complete.htm

### Raison d'Etre

I hear you say, "Who the hell needs another chess framework? There are many good ones already (e.g. [Python], [Javascript]), even ones written in [Swift]. There's certainly no shortage of [websites] or [apps]. Isn't it a colossal waste to devote time and energy in such a vain pursuit? I mean, you might as well go write a [Lisp interpreter]() while you're at it."

Fair enough. [Ben Laurie] once described the game sudoku as a "denial of service attack on human intellect" ([source]). I hope this was for me more intellectually edifying than playing [sudoku]. And even if it hadn't been, I like to solve puzzles and making things. And that's what I like so much about programming.

[Python]: https://github.com/niklasf/python-chess
[Javascript]: https://github.com/jhlywa/chess.js
[Swift]: https://github.com/nvzqz/Sage
[websites]: https://en.lichess.org
[apps]: https://itunes.apple.com/us/app/chess-play-learn/id329218549?mt=8&ign-mpt=uo%3D4
[Ben Laurie]: http://en.wikipedia.org/wiki/Ben_Laurie
[sudoku]: https://github.com/proxpero/Pseudoku
[source]: http://www.norvig.com/sudoku.html

### Prospectus

In this series, I will go into detail about how I wrote it, partly as an explanation to anyone who might be curious, but also to help me clarify for myself the good and the bad. I have a habit of coding madly away halfcocked ideas because I can't always comprehend their true value if they're not set down in type. I therefore hope that forcing myself to explain in dirt simple posts the justification behind the components of the framework will help me understand what code to gut. And who doesn't [love to gut code](http://inessential.com/2006/03/03/negative_5750)?

### Overview

Before I start in to the nitty-gritties, let me give a brief architectural overview. Think of this framework as having four layers:

  1. A `Bitboard` wraps a `UInt64`, mapping each square on the chessboard to a simple bit.
  2. A `Board` is a collection of `Bitboard`s, where each bitboard models a particular piece: one bitboard for the white bishops, one for black pawns, etc. In this any setup of pieces can be represented.
  3. A `Position` stores a `Board` as well as turn information, castling rights, the en passant square, and other things. A `Position` can serialize itself into an [FEN]() and back.
  4. It's not totally correct to say that a `Game` is a collection of `Position`s, but it's not far off either, one position for each move in the game. A `Game` also stores player data, the date, the outcome, etc. The `Game` provides the public interface for all the logic in the framework.

There are several other objects, but these form the backbone. One of the design principles is that computations happen at the lowest level possible. For example, the `board` can say whether a king is being attacked, because it has all the necessary information, whereas a `bitboard` doesn't have enough information. But a `board` can't say whether a move is valid or not because it doesn't have turn information, so that responsibility goes to the `position`.

Next time, let's take a closer look at the `Bitboard` struct.
