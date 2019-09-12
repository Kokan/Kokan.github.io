---
published: true
title: Play Tic-Tac-Toe with syslog-ng
layout: post
tags: [syslog-ng, conf-lang, hacking, ]
---

## Background information

The fun experiment below all done using syslog-ng an application for handling (that is, collecting, sorting, and so on) logs. Normally the functions above would be in focus, but not now. I would like to explore different aspects of the application, which including some configurations that may seem nothing short of magical (at first sight, at least).

About a year ago I shared a configuration that calculates the value of PI. The post ([Calculate PI with syslog-ng](https://www.syslog-ng.com/community/b/blog/posts/calculate-pi-syslog-ng)) explains in detail how that configuration works. While it was fun to create, it lacked one important thing: user interaction.

I was in a need of a simple game (and to be honest, mechanics are not always that simple in the background), both when it comes to visuals (as only console support is available) and game logic.
As a result, I have decided to re-create a classic: [Tic-Tac-Toe](https://en.wikipedia.org/wiki/Tic-tac-toe).

There were some rules I set up for myself:
* The original syslog-ng code was not to be modified to achieve any of this (namely, the code was to remain upstream and releasable)
* No language bindings were to be allowed, as it would have been an easy task to write those things in different programming language and just connect the proper bindings afterwards.


## Tic-Tac-Toe

Without further ado or lengthy explanation, just grab the latest syslog-ng (at least *3.22.1* version is required) and the following configuration file: [https://github.com/Kokan/syslog-ng-conf-lang/blob/tic-tac-toe-blog-post/tic-tac-toe.conf](https://github.com/Kokan/syslog-ng-conf-lang/blob/tic-tac-toe-blog-post/tic-tac-toe.conf).
Make sure you start *syslog-ng* in foreground mode without any debug/trace level enabled: *syslog-ng -F -f <path-to-tic-tac-toe.conf>*

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



The only significant difference between this configuration and the one calculating the value of PI is that it can wait for, and react to user input. The easy part is to listen to user inputs via console, as there is already a *stdin* source.

## Reacting to user input

There are a few ways to approach this as well. *Junction* and *channel* are fairly old features that, which could create a conditional path. However, in the meantime, *if-else* (essentially a syntactic sugar for the previous) has also become available. As of now, there is no *switch-like* method to react to messages.

Nevertheless, there is also a cooler way to do it; since this patchset [syslog-ng#2716](https://github.com/balabit/syslog-ng/pull/2716) there is a way to map values into a different value, as follows:

```
template "state1" "state2";
template "state2" "state3";
template "state3" "final";


$(template state1) # => "state2"
$(template ${current_state}) # => it depends on value of ${current_state}
```

Using this method, it is possible to create a switch-like case for values. Naturally, this could be always written as an *if-else*, but it would take much more configuration and thus complexity would quickly raise. Logging at the current Tic-Tac-Toe implementation, it may seem as a problem.
In fact, it feels more natural to write a state machine via this way.

The following example describes a set of possible steps:
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

The set of *X*, *O* and *0* describes the state of the game, followed by the step provided by the user, mapped into the next step. Imagine just this subset of states with *if-else*.


# Waiting for user input

This part was the actual puzzle for me. Actually, what I needed was a way to somehow store a *state*, and make it accessible in the message provided by the user (don't forget that we have messages to deal with).
The idea was to have one message from the user, and another one to store the *state*. Initially I was thinking about creating two logpaths for each of those, but in the end I found that even when using two logpaths, merging the messages is needed.

Luckily, a parser called *grouping-by* already exists, and, that does exactly what is needed here. For this parser, is possible to provide a *key* which groups messages together, and then any number of the messages' contexts can be merged.

After this the logic is fairly easy:
1. Generate an initial message with an initial state.
2. Receive user input or receive input from the previous loop with a state (similar to the pi configuration).
3. Wait for user input, and a state message.
5. Act upon them: display, game logic.
5. Forward the new state to the next loop and go back to step 2.

Now let us see the above within the configuration:
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


## What's next?

There are a few ideas to act upon, but coming up with, and then solving these kinds of issues requires a certain state of mind. All the same, solving more complex issues with wringing code is tiresome, so maybe I'll act upon that project next... :)


