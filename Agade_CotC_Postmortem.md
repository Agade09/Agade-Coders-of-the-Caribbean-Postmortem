# Agade Coders of the Carribbean Postmortem

I will try to describe my AI which finished second in the CotC contest.

## Local Arena

One of the first things I did was write my own [referee program](https://github.com/Agade09/CG-CotC-Arena) to be able to play games against previous versions of my AI. This is important to test ideas and check that no regressions have snuck into a version.

This contest had a bit of a rock/paper/scissor element to it so I did have a few issues with versions looking good in self play but not working as well against real opponents.

Furthermore, I learned from GitC that the win rate you get in self play is often exaggerated because in the extreme case you'll play a bot against itself, minus a mistake, and win all the time. So a lesson I learned from GitC was to try and focus on big features first and not waste too much time on supposedly 1% coefficient changes that are actually worth 0.1% in the arena.

## Search algorithm

In this game it is necessary to search the space of possible moves in order to navigate your boat while avoiding mines and cannonballs. I believe it isn't possible to do this with heuristics. To search the space I decided to bruteforce each boat individually, this is a good approximation when boats aren't colliding with each other too much and it reduces the complexity of the search. In more detail this is pseudocode for my search:

```
BestStrat=ALL_WAIT_FOREVER;
depth=1;
while(TimeLeft){
    for(ship s in EnemyShips){
        BestStrat[depth][s]=Bruteforce(s,depth,BestStrat);
    }
    for(ship s in MyShips){
        BestStrat[depth][s]=Bruteforce(s,depth,BestStrat);
    }
    ++depth;
}
```

So when I bruteforce the first enemy ship's move at a depth of 1 I assume everyone else waits, then when I bruteforce the enemy's second ship assuming his first ship makes that move and everyone else waits etc...
Then at depth 2 when I bruteforce the first enemy ship's moves I assume every other ship makes the move that was found at depth 1 and then waits during its second turn, etc...

The results of a partially searched depth, because I ran out of time, are discarded. I reached depths of 4 to 6. The win rate was very similar at a fixed depth of 4 and takes little time. This is why some people thought I was running heuristics due to answering in a few milliseconds at the beginning of the competition.

I'm not convinced my search algorithm was so great, it feels hacky to me but it was simple. I suspect for example that when enemy boats are close to mine the algorithm must oscillate between solutions as the depth increases.

## Bruteforce per boat

### Evaluation

The evaluation of a boat's position is:

* `Rum`
* `{-1,1,0.5}[speed]`
* `-0.02*pow(DistToCenter,2)-0.1*AngleDistToCenter`
* `-0.5*DistToNearestBarrel-0.1*AngleToNearestBarrel` if there is a barrel
* `-0.5*DistToEnemyCapitalShip-0.1*AngleToEnemyCapitalShip` if there is no barrel and the enemy capital ship has more rum than mine
* `-10` points if the boat is touching the middle or the back of an enemy ship (blocked)

I value speed 1 more than 2 because although speed 2 might seem better at dodging it is actually less maneuverable: you are more likely to crash into mines, other boats and the edges of the map and you have one less available move (FASTER) to make use of.

I favor being towards the center and facing the center because it means having more space to maneuver. I also figure it keeps the boats closer to the action on average. The opposite of that would be to go and face a corner.

I didn't try to make my ships run away when I had more rum, I suspected it would make my ships back themselves into corners and that the meta would be to "stand and fight".

In the bruteforce the position is evaluated after each move and the final score is `Eval(first position)*pow(0.75,0)+Eval(second position)*pow(0.75,1)+...`. It is important to value immediate, more certain gain than the uncertain future.

### Search

To Bruteforce the moves of a boat I consider 5 moves FIRE, FASTER, SLOWER, PORT and STARBOARD. FASTER and SLOWER are discarded if the speed is too low/high.

FIRE is targeted at a heuristic position in front of the enemy. The heuristic is to intercept the target assuming it moves in a straight line at a speed of maximum 1 without collisions. I don't move the enemy with speed 2 because that usually leads to shooting very far away from the target.

If the shot can be made (dist<=10) and the ship can shoot (cooldown==0) then add some points to the evaluation: `5*exp(-0.1*DistanceOfShot)`. I thought it would be very complicated to give shots value by their hitting of the enemy, as you can never know that you'll hit. Giving shots a more heuristic value seemed like a really good approximation to me and it worked well. Just shoot as much as you can, as close as you can and you'll hit eventually, as well as give less options to your enemy. The more he's dodging, the less he's shooting and picking up barrels.

Keep in mind that it takes `2+round(distance/3.0)` turns to hit a target because the travel time is `1+round(distance/3.0)` and it takes one turn to spawn the cannonball.

If there is an enemy behind the boat which would touch the mine spot after `max(1,speed*2)` moves forward, then FIRE becomes MINE and scores 10 points. So for example if a boat is moving at speed 1 towards my mine spot and is distance 2 away from it, place a mine because next turn he will be right in front of the mine and he'll have to SLOWER, then turn, then FASTER to avoid it, and if he's distance 1 away, even better because he will blow up on the mine unless he anticipates and plays SLOWER, which is still good because then he has to turn and accelerate around the mine.

The points for FIRE and MINE are added to the evaluation of the resultant positon and also multiplied by the pow(0.75,depth) "patience" factor.

## Shooting

After I have bruteforced the enemy's moves along with my moves, if I'm supposed to FIRE this turn, I try to pick a better target than the one found by the heuristic. To do this I disable my cannonballs in the bruteforce so that the enemy doesn't dodge cannonballs I will never fire (I will change the target). I then use my prediction of the enemy's position to intercept the center of his boat as early as possible with two exceptions:
* If I can intercept the position where he was the turn before and still hit him when the cannonball lands then that is a better shot because it is harder to dodge. One example of this is close range shooting in front of a stationary boat because I anticipate its movements, the boat can then dodge that shot by not moving at all, better to shoot at it and still hit it in the tail when it moves.
* If I predict that the enemy will pass by my mine spot within 2 turns place a mine instead of shooting. This rarely happened but it was possible to, for example, mine a boat in the back if it turned into my boat.

The way I saw it, shooting at my enemy's predicted position was a way of denying him his best move, which for example in the early game often involved blowing up his barrels and delaying him.

The enemy's shots are not reassigned, he fires according to the heuristic in my bruteforce. And as I said, in the bruteforce I don't fire at all.

## Details

* Memorise mine cooldowns which aren't given in input so you don't try to place mines when you can't or predict enemy mines when he can't.
* Deduce ship cannon cooldowns by looking at new cannonballs appearing in the inputs.
* Memorise mine positions as they may get out of your line of sight. This allows for slightly better prediction of future navigation of your opponent, because you sometimes aren't in visible range of the mines around him.

## Strategies I didn't use

* Suicide to regroup the rum in one boat didn't seem like it would be the meta game. It's not obvious that 1 boat with 40 rum is better than 2 boats with 20 rum. I might well sink you with my two boats.
* I didn't come up with anything clever when it comes to barrels, I didn't try to be the last one to pick up a barrel or anything like that which I saw other AIs do. Maybe I missed something here.

## Conclusion

I tried to focus on the most simple/effective ideas as I think the 720 lines of code show. I have big doubts about my search algorithm though. I wonder if I missed something ovious in that regard or if the only way to go is minimax.

I'm disappointed to have finished second after being first during so much of the competition, waking up to a bot I never really had a chance to try to beat. But that's how contests are, you stop at some arbitrary time and whoever is first then wins. ReCurse somehow managed to piece together a minimax for this game which is really cool. I didn't think it could be done until I saw it. I guess he had the vision.

And congratulations to pb4 who rivaled me during most of the competition.