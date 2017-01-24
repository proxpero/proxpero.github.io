---
layout: post
title: The Board
comments: false
mathjax: false
ganalytics: true
---

### Board

Whereas `Bitboard` stored a single, simple `UInt64`, a `Board` struct stores an array of 12 `Bitboard`s, one for each piece in chess (6 types from pawn to king and two colors, black and white).
