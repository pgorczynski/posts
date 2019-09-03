---
title: "10 lessons learnt from the Ruby Refactoring Kata - Tennis Game"
created_at: 2019-09-03 17:55:30 +0200
kind: article
publish: false
author: Andrzej Krzywda
newsletter: :skip
---

Over the last ~2 months, I’ve been scheduling some time to work on a specific Ruby code which is designed to be a good starting point for a refactoring.

Those exercises are called Refactoring Katas. The one I picked up is called Tennis Game.

<!-- more -->

# 23 things I learnt from the Ruby Refactoring Kata - Tennis Game

## Introduction

The responsibility of this code is to return a string with a Tennis game result. The input comes as sequential calls to the `won_point` method. The result is returned via `score` method. Those 2 methods create the public API of the TennisGame1 object.

## Initial code

```ruby
class TennisGame1

  def initialize(player1Name, player2Name)
    @player1Name = player1Name
    @player2Name = player2Name
    @p1points = 0
    @p2points = 0
  end
        
  def won_point(playerName)
    if playerName == "player1"
      @p1points += 1
    else
      @p2points += 1
    end
  end
  
  def score
    result = ""
    tempScore=0
    if (@p1points==@p2points)
      result = {
          0 => "Love-All",
          1 => "Fifteen-All",
          2 => "Thirty-All",
      }.fetch(@p1points, "Deuce")
    elsif (@p1points>=4 or @p2points>=4)
      minusResult = @p1points-@p2points
      if (minusResult==1)
        result ="Advantage player1"
      elsif (minusResult ==-1)
        result ="Advantage player2"
      elsif (minusResult>=2)
        result = "Win for player1"
      else
        result ="Win for player2"
      end
    else
      (1...3).each do |i|
        if (i==1)
          tempScore = @p1points
        else
          result+="-"
          tempScore = @p2points
        end
        result += {
            0 => "Love",
            1 => "Fifteen",
            2 => "Thirty",
            3 => "Forty",
        }[tempScore]
      end
    end
    result
  end
end

```

## Initial tests
At the first glance the tests look quite cool - the code is declarative, you should be able to see the expectations. However, while working on this code for longer I came to the conclusion that those tests are not really that perfect.

```ruby
TEST_CASES = [
   [0, 0, "Love-All", 'player1', 'player2'],
   [1, 1, "Fifteen-All", 'player1', 'player2'],
   [2, 2, "Thirty-All", 'player1', 'player2'],
   [3, 3, "Deuce", 'player1', 'player2'],
   [4, 4, "Deuce", 'player1', 'player2'],
   
   [1, 0, "Fifteen-Love", 'player1', 'player2'],
   [0, 1, "Love-Fifteen", 'player1', 'player2'],
   [2, 0, "Thirty-Love", 'player1', 'player2'],
   [0, 2, "Love-Thirty", 'player1', 'player2'],
   [3, 0, "Forty-Love", 'player1', 'player2'],
   [0, 3, "Love-Forty", 'player1', 'player2'],
   [4, 0, "Win for player1", 'player1', 'player2'],
   [0, 4, "Win for player2", 'player1', 'player2'],
   
   [2, 1, "Thirty-Fifteen", 'player1', 'player2'],
   [1, 2, "Fifteen-Thirty", 'player1', 'player2'],
   [3, 1, "Forty-Fifteen", 'player1', 'player2'],
   [1, 3, "Fifteen-Forty", 'player1', 'player2'],
   [4, 1, "Win for player1", 'player1', 'player2'],
   [1, 4, "Win for player2", 'player1', 'player2'],
   
   [3, 2, "Forty-Thirty", 'player1', 'player2'],
   [2, 3, "Thirty-Forty", 'player1', 'player2'],
   [4, 2, "Win for player1", 'player1', 'player2'],
   [2, 4, "Win for player2", 'player1', 'player2'],
   
   [4, 3, "Advantage player1", 'player1', 'player2'],
   [3, 4, "Advantage player2", 'player1', 'player2'],
   [5, 4, "Advantage player1", 'player1', 'player2'],
   [4, 5, "Advantage player2", 'player1', 'player2'],
   [15, 14, "Advantage player1", 'player1', 'player2'],
   [14, 15, "Advantage player2", 'player1', 'player2'],
   
   [6, 4, 'Win for player1', 'player1', 'player2'], 
   [4, 6, 'Win for player2', 'player1', 'player2'], 
   [16, 14, 'Win for player1', 'player1', 'player2'], 
   [14, 16, 'Win for player2', 'player1', 'player2'], 

   [6, 4, 'Win for player1', 'player1', 'player2'],
   [4, 6, 'Win for player2', 'player1', 'player2'], 
   [6, 5, 'Advantage player1', 'player1', 'player2'],
   [5, 6, 'Advantage player2', 'player1', 'player2'] 
]

class TestTennis < Test::Unit::TestCase
  def play_game(tennisGameClass, p1Points, p2Points, p1Name, p2Name)
    game = tennisGameClass.new(p1Name, p2Name)
    (0..[p1Points, p2Points].max).each do |i|
      if i < p1Points
        game.won_point(p1Name)
      end
      if i < p2Points
        game.won_point(p2Name)
      end
    end
    game
  end

  def test_Score_Game1
    TEST_CASES.each do |testcase|
      (p1Points, p2Points, score, p1Name, p2Name) = testcase
      game = play_game(TennisGame1, p1Points, p2Points, p1Name, p2Name)
      assert_equal(score, game.score())
    end
  end
end
```

## Lessons learnt

1. Don’t trust the tests

At first, I thought that the tests five me enough coverage, that I can do the initial refactoring safely. However, it’s only over time that I learnt what are the drawbacks of the current tests design. The main problem is this code:

```ruby
  def play_game(tennisGameClass, p1Points, p2Points, p1Name, p2Name)
    game = tennisGameClass.new(p1Name, p2Name)
    (0..[p1Points, p2Points].max).each do |i|
      if i < p1Points
        game.won_point(p1Name)
      end
      if i < p2Points
        game.won_point(p2Name)
      end
    end
    game
  end
```

You see, this code plays through the game always in the same manner — first add all possible points to player1 and only later add player2 points. 
For certain implementations this might be a correct suite of tests, but if we switch to more stateful implementations, we’re lacking the coverage.

I consider refactoring a process of learning. It’s learning of the domain, of the code and of the tests. When you look at it this way, maybe it was alright - I started refactoring and through this process I learnt about the problems with tests. However, this is only valid, if I don’t push my changes before I learn the lessons. If I do, I risk introducing breaking changes.

2. Learn at least some basics of the domain

I’m not a big fan of tennis, but I thought I knew enough about it work on this code.

In practice, this was hard. I kept forgetting what’s the meaning of `Love`, I had to constantly look up the possible results.

I think this led me to overgeneralising the code sometimes. The names I used for method names, for object names - they were not really names that would appear in a conversation among the real fans of the game.

That’s something what I’m trying to be more professional in my commercial projects. When I worked on accounting project, I took an online class on accounting. When I worked on a publishing project, I have studied the publishing industry, including the possible business models, what publishers struggle with, how publishers cooperate with authors. I talked to certain publishers.

In this kata, I clearly failed at it. I wasted some time, because I couldn’t visualise the domain well enough.

My domain vocabulary was very poor here - I kept using the words: `game`, `score`, `result` without learning some more.

3. Merciless refactoring can be a nice learning technique

Let me quote the extreme programming definition of what I mean here:

```
Refactor mercilessly to keep the design simple as you go and to avoid needless clutter and complexity. Keep your code clean and concise so it is easier to understand, modify, and extend. Make sure everything is expressed once and only once. In the end it takes less time to produce a system that is well groomed.
￼
There is a certain amount of Zen to refactoring. It is hard at first because you must be able to let go of that perfect design you have envisioned and accept the design that was serendipitously discovered for you by refactoring. You must realize that the design you envisioned was a good guide post, but is now obsolete.
```

The above definition makes most sense when applied to a situation when you think you know what is the perfect design and then you try to apply it. Usually, the code would tell you why your vision may not be so perfect. The solution is to follow the code.

In this Refactoring Kata, you can see my initial attempts to actually understand what the code does, by changing the code.

<<VIDEO 1>>

Even though, I don’t understand the domain yet - I’m following the typical code smells to restructure the code. It’s purely technical at this stage. I have no idea what the code really does (I’m trying to guess) but I know that certain technical transformations will keep the behaviour the same, while allow me to look at the code from a different angle.

## TODO

2. Extreme Refactoring requires extreme test coverage

3. It was easy for me to experiment with different designs/models for the code

4. It was hard for me to change the design of the tests

The existing tests are a bit “creative”. For some reason I didn’t touch the tests design. Maybe I just wasn’t confident enough without some mutation testing (tests for tests)?
Still, the original shape of the tests remained the same, till the end (if there’s any end).

5. 