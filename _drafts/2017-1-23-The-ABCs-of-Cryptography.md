---
title: The ABCs of Cryptography
summary: I write a program implementing a Caesar cipher, and then another to break it.
tags: [Swift, Cryptography]
---

A *code* is a set of rules for transforming information from one form into another. [Morse code], for example, converts letters into a series of dots and dashes. [Braille] converts letters into blocks of raised dots. The motivation here isn't to hide the information, but rather to make it more accessible. When you're limited to tones or silences, then encoding letters into long and short beeps is a practical way to communicate a message across a wire.

A *cipher* is a kind of code, but the motivation is different. Here you're turning the information from an readable form into an intentionally unreadable form. You want to keep the information secret. *Crypto-* comes from a Greek word [κρυπτός], which means hidden. Cryptography is "hidden writing".

[Morse code]: https://en.wikipedia.org/wiki/Morse_code
[Braille]: https://en.wikipedia.org/wiki/Braille
[κρυπτός]: http://www.perseus.tufts.edu/hopper/text?doc=Perseus%3Atext%3A1999.04.0057%3Aentry%3Dkrupto%2Fs

## The Caesar Cipher

The [Caesar cipher](https://en.wikipedia.org/wiki/Caesar_cipher) is simple substitution cipher where each letter the message is replaced by another letter some fixed distance down the alphabet. For example, if the distance is 13, then the 0th letter of the alphabet "A" would be encoded by the 13th letter of the alphabet "N".

<table style="width: 100px">
<tr><td>A</td><td>N</td></tr>
<tr><td>B</td><td>O</td></tr>
<tr><td>C</td><td>P</td></tr>
<tr><td>D</td><td>Q</td></tr>
<tr><td>E</td><td>R</td></tr>
<tr><td>F</td><td>S</td></tr>
<tr><td>G</td><td>T</td></tr>
<tr><td>H</td><td>U</td></tr>
<tr><td>I</td><td>V</td></tr>
<tr><td>J</td><td>W</td></tr>
<tr><td>K</td><td>X</td></tr>
<tr><td>L</td><td>Y</td></tr>
<tr><td>M</td><td>Z</td></tr>
<tr><td>N</td><td>A</td></tr>
<tr><td>O</td><td>B</td></tr>
<tr><td>P</td><td>C</td></tr>
<tr><td>Q</td><td>D</td></tr>
<tr><td>R</td><td>E</td></tr>
<tr><td>S</td><td>F</td></tr>
<tr><td>T</td><td>G</td></tr>
<tr><td>U</td><td>H</td></tr>
<tr><td>V</td><td>I</td></tr>
<tr><td>W</td><td>J</td></tr>
<tr><td>X</td><td>K</td></tr>
<tr><td>Y</td><td>L</td></tr>
<tr><td>Z</td><td>M</td></tr>
</table>
