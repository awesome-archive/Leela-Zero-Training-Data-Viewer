# LZ Training Data Viewer
Reversing LZ training data into commented sgf files.

This is my personal repo to share some analysis I have done on Leela Zero training data. As a way to learn to use Github.

Fresh beginner in Python and in object oriented programing, I chose as a toy problem to write a python script that reverses Leela Zero training data into self-play games sgf files, embedding some statistics on the search policy:

![image](https://user-images.githubusercontent.com/37498331/46257726-b7b93a00-c4be-11e8-9a2f-978c3c632e3c.png)

## Methodology

Empirically, I have found that LZ training data files consist in a series of training samples corresponding to successive positions in a series of self-play games. Currently, each training data chunk wraps 32 such games.

By taking advantage of this property (successiveness of positions) and comparing the input planes of a given position with the input planes of the next position, it is possible to deduce the move that was actually picked by temperature (T=1) in that given position. And to check what were the policy value (p_picked) and the rank in the search (picked_rank) for that move. In turn, this allows to reconstruct the whole game and to generate a sfg file with comments on the search policy distribution.

This script also outputs an additional .csv table in which each line corresponds to a training position, with meta data and the basic statistics:

```
    # Position meta-data:
    chunk_id:          chunk number within training file
    game_id:           game number within the chunk (start at 1)
    game_len_raw:      game length, raw
    game_length:       game length, floored by 20
    move_count_raw:    move count, raw
    move_count:        move count, floored by 20
    side_to_play:      color to play (Black/White)
    played_by:         move belongs to winner/loser of the game
    picked_move        board coordinate of move picked by temperature
    
    # Policy statistics:
    p_picked_raw       policy of picked move, raw
    p_picked           policy of picked move, floored by 0.02
    picked_rank        the rank of the move picked by temperature in the search policy (best move rank = 1)
    p_pass_raw:        policy probability of pass move
    p_pass:            policy probability of pass move, floored by 0.02
    pass_rank:         the rank of the pass move in the search policy 
    p_max_raw:         policy max (=policy of best move), raw
    p_max:             policy max (=policy of best move), floored by 0.02
    p_best5_raw:       cumulated policy probability of top  5 moves, raw
    p_best5 :          cumulated policy probability of top  5 moves, floored by 0.02
    p_best10_raw:      cumulated policy probability of top 10 moves, raw
    p_best10:          cumulated policy probability of top 10 moves, floored by 0.02
    p_pass_raw:        policy probability of pass move
    p_pass:            policy probability of pass move, floored by 0.02
    norm_cost_raw      normalized computing cost of genrating the next move in current position = tree reuse ratio
    norm_cost          normalized computing cost of genrating the next move in current position, floored by 0.02
    
    # Commented sgf:
    chunk_XXXX_game_YYYY.sgf : a text file in FF4 format of the game ID YYYY of chunk ID XXXX
```

**Sample files: chunk n°1000**
[train_dec5b9d8__1000.zip](https://github.com/Ishinoshita/Leela-Zero-Training-Data-Viewer/files/2435705/train_dec5b9d8__1000.zip)

![image](https://user-images.githubusercontent.com/37498331/46316483-947eaf80-c5d0-11e8-9de1-9e48dc87530a.png)


# Preliminery observations on the training data
*02-Oct2018*

## Data used

LZ177-generated training data uploaded on 18-Sep-2018 (network hash dec5b9d8, promoted on 15-Sep-2018).

Since I did analysis with Excel (1M lines limit), I took arbitrary slices of 100 consecutive chunks, each corresponding to ~3200 games or ~550k to 600k positions. I analysed chunks 0 to 99, then chunks 1000 to 1099 to check that the conclusions were standing and independent from chunks sampling.

## Found weird data !

I quicly spotted weird policy values in both slices of chunks, not looking like multiples of 1/600th., like here:

![image](https://user-images.githubusercontent.com/37498331/46261875-fb7d6500-c4f9-11e8-8de8-0245f2b93eeb.png)

Suspecting some flaw in my code (although a priori it does not do any arithmetic processing on the values p_picked_raw, p_max_raw and p_pass_raw, just a conversion to float, a sorting step before conversion back to string), I checked the raw training data and found the weird data were already there:

![image](https://user-images.githubusercontent.com/37498331/46261993-7004d380-c4fb-11e8-8b62-de236773cdc7.png)

    0.737846 = 368923/500000, 0.5M not really a reasonable number of visits ...

Note that when the policy contains such a weird value, all non-zero policy values are weird in the training sample. As a shortcut, I here after inaccurately designate such values as *non-fractional*, as they look more like real numbers than like fractions of any reasonable number of visits.

I did an in depth sanity check of the chunks 1000-1099 and found a surprising high proportion of positions containing such non-fractional values: 40592 / 609087, ie 6.7% (not a pee in the ocean ... ).

Then I spotted another kind of weirdness, namely positions where p_picked_raw = p_max_raw = 1 during very long sequence in the game, particularly early in the game. Such games exhibit a clearly different distribution in terms of p_picked_raw or p_best_10_raw:

* To start with, two very standard 300 moves games (left: a no-resign game that probably hit the resign threshold ~100 moves; right a resign game):
![image](https://user-images.githubusercontent.com/37498331/46262164-a8a5ac80-c4fd-11e8-9910-9d9c626efd26.png)
and their sgf:  [chunk_1002_game_7.zip](https://github.com/Ishinoshita/LZ-Training-Data-Viewer/files/2432011/chunk_1002_game_7.zip), [chunk_1004_game_29.zip](https://github.com/Ishinoshita/LZ-Training-Data-Viewer/files/2432012/chunk_1004_game_29.zip)

* Then, two 320 moves games (right: yet another normal no-resign game; left: what I call an out-of-distribution game): 
![image](https://user-images.githubusercontent.com/37498331/46262178-c541e480-c4fd-11e8-8782-589ba62aaa76.png)
and their sgf: [chunk_1047_game_30.zip](https://github.com/Ishinoshita/Leela-Zero-Training-Data-Viewer/files/2435778/chunk_1047_game_30.zip); 
[chunk_1042_game_7.zip](https://github.com/Ishinoshita/Leela-Zero-Training-Data-Viewer/files/2435779/chunk_1042_game_7.zip)

Although p_best_10_raw = 1.000000 is of course not impossible, it turns out that in most cases we lso have p_max_raw = p_picked_raw = 1, which may not be strictly impossible, but a series of such values seems almost impossible.

I initially suspected fake data. But when I replayed a few sfg with Sabaki (LZ177, 1600v, no randomness) I found that LZ177 almost always agreed with the game trajectory.

This second issue is affecting less training samples (1731 / 609087, or 0.28%).

I then discovered that these two issues affect the very same games and are probably one and a single issue. In such games, either p_picked_raw = p_max_raw = 1 and the rest of the policy is de facto 0, or all values are not multiples of 1/1600th and these values display unsually sharp policy (p_best_5 close to 1 if not equal, p_max quite high) 

**Overall, the proportion of training samples affected by this issue is 6.9%.**


Since these data doesn't look synthetic (not indentical games; they seems to follow LZ177 inclination), there must be some rational explanation. Highly concentrated policy and non-fractional values made me think of genuine self-play games but played with temperature parameter  much lower than 1. However, I found that not all games with non-factrional policy exhibit obvious out of distribution policy and long series of p_max = 1 ... Herebelow, for example, all games have all their positions exhibiting non-fractional policy values (but for the 1 and 0 when p_max=1), although they all look normal, but for game 1003_19 !
![image](https://user-images.githubusercontent.com/37498331/46262441-8c0b7380-c501-11e8-923e-3514e61049ee.png).
See sgf: [chunk_1003_game_21.zip](https://github.com/Ishinoshita/Leela-Zero-Training-Data-Viewer/files/2435854/chunk_1003_game_21.zip); [chunk_1003_game_19.zip](https://github.com/Ishinoshita/Leela-Zero-Training-Data-Viewer/files/2435858/chunk_1003_game_19.zip);[chunk_1003_game_18.zip](https://github.com/Ishinoshita/Leela-Zero-Training-Data-Viewer/files/2435855/chunk_1003_game_18.zip). 

Thus, it would take to assume some client not only tinkering with his leela code and but toying with different temperatures.
Hopefully, there is a much less fancy explanation than this variable temperature theory. Eager to know it! 

## Update - 21 Oct 2018

There is indeed a simple explanation for policy values non-multiple of 1/1600, as explained by @Ttl [here](https://github.com/gcp/leela-zero/issues/1904#issuecomment-426263786): "next branch uses no-pruning time management during the self-play game that stops the search if the best move can't change anymore (Pull Request #1497)."

The other two issues described here above, 1) p_picked=0,  2) policiy very concentrated of a few moves,if not a single move, plus another one found in LZ181 traing date and described [here](https://github.com/gcp/leela-zero/issues/1939) are still unexplained.

## Update - 10 Nov 2018

p_picked=0 issue now explained by a bug in `train2sgf` python script. Due to a pecularity in coding of input planes respect to 361st intersection, move of board coordinate T19 was wrongly interpreted as Pass move. No more p_picked = 0 position found after bug correction.

 `train2sgf` python script updated.

