---
published: true
title: syslog-ng tic-tac-toe
layout: post
tags: [syslog-ng, conf-lang, hacking, ]
---

## some context

This is all done with syslog-ng; this is an application to handle logs (collect, sort, and so on). Normally those function would be a focus, but not now. I like to explore different sides, which includes some close-to-magic like configurations. At least at first sight.

A year ago I have shared a configuration that calculates the value of PI. This post: [Calculate PI with syslog-ng](https://www.syslog-ng.com/community/b/blog/posts/calculate-pi-syslog-ng) explains in details how does that configuration works. While it was fun to create it lacked one important thing: user interaction!

I was in a need for a simple game (and mechanics are not always that simple in the background), both in visual (there is only console support for now) and game logic.
Therefore I have decided to re-create a [tic-tac-toe](https://en.wikipedia.org/wiki/Tic-tac-toe).

Almost forget there are some rules I set upon:
* syslog-ng code cannot be modified to achieve any of this (the code should be upstream and releasable)
* no language binding, as it would be easy task to write those things in other language and just connect the proper binding


## tic-tac-toe

Without much explanation, you could grab the latest syslog-ng (you probably need *3.22.1* released before) and the following configuration file: [https://github.com/Kokan/syslog-ng-conf-lang/blob/tic-tac-toe-blog-post/tic-tac-toe.conf](https://github.com/Kokan/syslog-ng-conf-lang/blob/tic-tac-toe-blog-post/tic-tac-toe.conf).
Be sure to start *syslog-ng* in foreground mode without any debug/trace level enabled: *syslog-ng -F -f <path-to-tic-tac-toe.conf>*

```
> syslog-ng -F -f /tmp/tic-tac-toe.conf


  X |   |
 -----------

    |   |
 -----------

    |   |
 -----------
 Tic-Tac-Toe
You are with O, please provide a step (eg.: b3):
b2

  X | X |
 -----------

    | O |
 -----------

    |   |
 -----------
 Please give your next move:

```

and so on.



The only real key difference between this and the one calculating the value of pi; that it can wait and react to user inputs. The easy part is to listen to user inputs via console; as there is already a *stdin* source.

## react

There is a few way to approach this as well, the *junction* and *channel* is a fairly old feature, which could create conditional path. But since that there is also *if-else* (which is a synthetic sugar for the previous). As of now there is no *switch* kinda way to react to messages.

But there is also a cooler way since [syslog-ng#2716](https://github.com/balabit/syslog-ng/pull/2716) there is a way to map values into a different value, as follows:

```
template "state1" "state2";
template "state2" "state3";
template "state3" "final";


$(template state1) # => "state2"
$(template ${current_state}) # => it depends on value of ${current_state}
```

With this it is possible to create a kinda switch case for values; of course this could be always written as an *if-else*, but it would take much more config and complexity quickly raises. Checking out the current tic-tac-toe implementation it may seems as an problem.
It feels more natural to write a state machine via this way.

The following describes a set of possible steps:
```
# Moves

template "X00000000+12" "XO0X00000";
        template "XO0X00000+13" "XOOX00X00"; #X win
        template "XO0X00000+22" "XO0XO0X00"; #X win
        template "XO0X00000+23" "XO0X0OX00"; #x win
        template "XO0X00000+32" "XO0X00XO0"; #x win
        template "XO0X00000+33" "XO0X00X0O"; #x win
        template "XO0X00000+31" "XO0XX0O00";
                template "XO0XX0O00+13" "XOOXXXO00"; #x win
                template "XO0XX0O00+23" "XO0XXOO0X"; #x win
                template "XO0XX0O00+32" "XOOXXXOOO"; #x win
                template "XO0XX0O00+33" "XO0XXXO0O"; #x win
```

The set of *X*, *O* and *0* describes the state of the game, followed by the step provided by user, mapped into the next step. Imagine just this subset of states with *if-else*.


# wait

This was the actual puzzle for me. Actually what was needed to somehow store a *state*, and make it accessible in the message provided by the user (remember we have messages).
The idea was to have one message from the user, and one message to store the *state*. Initially I was thinking to create two logpath for each of those thing, but in the end even with two logpaths merging the messages is needed.

There are is actually a parser called *grouping-by*, that does exactly what is needed here. It is possible to provide a *key* which groups messages together, and the any number of context of the messages can be merged.

After this the logic is fairly easy:
1. generate an initial message with initial state
2. receive user input or receive input from the previous loop with a state (similar to the pi config)
3. wait for user input, and a state message
5. act upon like: display, game logic
5. forward the new state to the next loop and go back to step 2

the same as a configuration:
```
# MAIN

log {
        source { tic-tac-toe-initiate-game-state(); };
        source { tic-tac-toe-input(); };
        source { tic-tac-toe-state(); };

        parser {
          grouping-by(
             key("")
             aggregate(
               value(".pass-throu" "1")
             )

             trigger( match("1", value(".tictactoe.input")) )
             timeout(9999999) #hopefully never
          );
        };

        parser { tic-tac-toe-move(); };

        filter { match("1" value(".pass-throu")); };
        rewrite { set("0" value(".pass-throu")); };
        rewrite { set("0" value(".tictactoe.input")); };
        rewrite { set("0" value("MESSAGE")); };

        destination { tic-tac-toe-tui(template("${.tictactoe.state}")); };
        destination { unix-stream("/tmp/tic-tac-toe.sock" template("$(format-ewmm)")); };
};
```


## next

There are a few ideas to act upon. But coming up and solving these kind of issues requires a state of mind. But I solving more complex issues with wringing code is tiresome, maybe I'll act upon that :)


