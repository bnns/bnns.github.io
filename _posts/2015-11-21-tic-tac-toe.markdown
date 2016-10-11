--- layout: post title:  "Tic Tac Toe in Elixir" date:   2015-11-21 20:55:37
-0600 categories: elixir ---

The following is a tutorial on creating a tic-tac-toe game in
[Elixir](http://wwww.elixir-lang.org), a functional language widely regarded as
the lovechild of [Ruby](http://www.ruby-lang.org) and
[Erlang](http://www.erlang.org/). Our goal is to improve our ability to think
about crafting software by making design decisions explicit - and incidentally
this involves grasping the features of the language. In accordance with this,
we will be making extensive stops along the way, highlighting the
**alternatives** to our decisions and discussing their relative merits.

{% comment %} # Uncomment this if we actually get to these features...  In the
end, the game will have the following features:

* A game server capable of spawning multiple instances of tic tac toe Option of
* Player vs. Player or Player vs. AI React-based front-end for the game board
* Websockets with Phoenix channels to play in real-time Test coverage for both
* client and server A page to view games currently in progress
{% endcomment %}

The first step is to break down our game into its logical constituents. In the
object-oriented world, this entails defining objects, and in the functional
world, it entails - you guessed it - thinking about functions. In both cases,
functions describe the transformation of state, but with FP, our design goal
should be to make our functions as
[composable](https://en.wikipedia.org/wiki/Function_composition) as possible.

Here's a list of game actions we need to think about:

1. Starting the game
2. Stopping the game
3. Displaying the board with the result of moves
4. Making a move
5. Evaluating a winning arrangement
6. Switching turns
7. Displaying an error or invalid move
8. Getting input

Object-oriented thinking naturally starts off thinking of possible states,
rather than actions, since that is what objects are (collections of states).
However, given a non-trivial state of a system, we as developers have a hard
time reasoning about its history. A functional approach is one that outlines
every possible *change* of state ahead of time. If we've planned our game
properly, there should be no state of the game that is not a result of
a sequence of our actions. This is desirable regardless of paradigm, but it's
just much easier when you think and implement your system in terms of [pure
functions](https://en.wikipedia.org/wiki/Pure_function).

That being said, objects have not disappeared - they are now to be thought of
as a sum of actions instead of states - in Erlang/Elixir parlance, they are
*actors*. Here is a list of possible objects, with a possible mapping of their
actions:

* Game [2, 5, 6, 7, 8] Board [3] Player [1, 4] AI [4, 5]

This is not the only way we could've arranged it - for example, we could allow
the evaluation of winning arrangements to be handled by the board, which will
come in handy in cases where there are multiple boards per game.

Before we create a complicated hierarchy of modules for each of our actors,
let's see how far we can get within a single one. We can start by specifying
the initial state of every game. Part of that is allowing the player to choose
**x**'s or **o**'s. Here is some sample code:

~~~elixir defmodule TicTacToe do @players [:x, :o]

    def start_game do user = "Do you want to play x or o? " |> IO.gets |>
    String.first |> String.to_atom game_state = [ ["1", "2", "3"], ["4", "5",
    "6"], ["7", "8", "9"] ] move_history = %{:x => [], :o => [], :moves =>
    Enum.into(1..9, [])} # new_turn user, game_state, move_history end end ~~~
    This code does the following:

1. It establishes that there are only two kinds of players (x and o)
2. It starts a game by asking which side the user would like to play
3. It sets up a board state and move history for each side

Those familiar with Ruby might recognize some of the language features.
Briefly:

* `@players` is a module constant `[:x, :o]` is a list of atoms `defmodule` and
* `def` define modules and functions `|>` is the Elixir pipe operator which
* passes output to input of the next function `move_history` is a variable set
* to a map `%{:key => value}` (a.k.a a hash) `1..9` is short hand for a range
* of integers `IO`, `String` and `Enum` are native modules in Elixir

Finally, we have a comment line for the `new_turn` function, which will kick
off the game.

The `new_turn` function takes a user (either `:x` or `:o`) which allows us to
re-use the same function for both players. In the OO universe, we would store
the user as a member variable and then reference it in our methods. Similarly,
in OO it is optional to pass the game state and move history since they would
generally be mutable entities. Given our statelessness, however, the natural
way to construct our function is via recursion. Here it is:

~~~elixir def new_turn(user, game_state, move_history) do
print_board(game_state) user_input = IO.gets("What is your move? ") |>
String.first |> String.to_integer valid_moves = move_history[:moves] opponent
= List.first(@players -- [user])

  if Enum.member?(valid_moves, user_input) do IO.puts "You selected " <>
  to_string(user_input)

    new_state = update_state( game_state, user, user_input)

    case not_over?(move_history, user, user_input) do {:tie, new_history} ->
    new_turn(:tie, new_state, new_history) {:over, new_history} ->
    new_turn(user, new_state, new_history) {:not_over, new_history} ->
    opponent_move = evaluate_next( game_state, valid_moves -- [user_input])
    IO.puts "Opponent selects " <> to_string(opponent_move) newer_state
    = update_state( new_state, opponent, opponent_move) 

        case not_over?(new_history, opponent, opponent_move) do {_,
        new_history} -> new_turn(user, newer_state, new_history) end end else
        IO.puts "Invalid move" new_turn(user, game_state, move_history) end end
        ~~~

This code does the following:

1. Print the game state
2. Get player move
3. Check for validity of the move and give feedback accordingly
4. For valid moves, determine if it ends the game

If the game ends we give the following feedback:

~~~elixir def new_turn(user, game_state, %{:moves => []}) do IO.write """

  Game over!

  """ IO.puts "The winner is " <> to_string user IO.puts "The final board is"
  print_board game_state end ~~~

Notice that we can use the same function names but with different signatures.
This is elixir's pattern matching at work, which for functions enables
polymorphism and destructuring. Next time we'll see how `not_over?`,
`update_state`, and `evaluate_next` are implemented. 
