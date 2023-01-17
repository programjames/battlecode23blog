### Introduction
Hey, it's me, \[Double J\] programjames. My first year of Battlecode was 8th grade, and that is when I decided MIT students were cool. Now it's my first year being one of those cool MIT hackers ;).

### Overview
The game this year is ocean themed. The map contains currents that push around robots, clouds that reduce visibility & speed, resource wells, and islands, and the goal is to capture as many islands as possible in 2000 turns.

A *very* brief overview of the game mechanics:
- Each team has 1–4 headquarters (HQs) that can spawn robots and craft anchors.
- Robots:
	- Carriers (squares)—bring resources to HQs and carries anchors to capture islands.
	- Launchers (triangles)—basic attack bot.
	- Amplifiers (circles)—allows nearby robots to communicate back to HQ.
	- Boosters (plus sign)—speeds up nearby units.
	- Destabilizers (times sign)—slows down nearby units and deals area of effect damage.
- Resources:
	- Adamantium—used to build carriers & amplifiers.
	- Mana—used to build launchers & amplifiers (costs both adamantium and mana to create amplifiers).
	- Elixir—used to build boosters and destabilizers, but hard to obtain.
	- Resource wells can be upgraded by extracting and throwing back in enough of the same resource. Alternatively, throwing adamantium into mana wells and vice versa transforms them into elixir wells.
- Anchors & Islands:
	- Anchors cost a lot of adamantium/mana/elixir.
	- Anchors on an island lose health if enemy units step on the island, until they break and the island is no long captured.
- Movement:
	- Currents push you in the direction their arrow shows.
	- Clouds slow down units.
	- Each unit has a different speed. Launchers can only move once every other turn, while carriers start out moving twice a turn, but lose speed as they gain weight from an anchor/gathering resources.
- Communication:
	- 1024 bits of communication space.
	- Any robot can read, but can only write if near an HQ or amplifier.

-----

### First Bot Uploaded Sunday, January 15, 9:54 AM EST
*"Victorious warriors win first and then go to war, while defeated warriors go to war first and then seek to win." -Sun Tzu*

We spent most of the first week building up our infrastructure: pathing, communication, and resource gathering. It wasn't until 36 hours before submissions were due that we even created any units other than carriers!

![[launcher_commit_msg.png]]

That's because pathing and resource gathering are *really hard* this year. Last year every spot was more-or-less passable, there was just rubble that slowed units down. Also, mined resources automatically got added to your team's bank. This year pathing has to take into account impassable locations and currents, and resources have to be brought back to base in order to use them. The largest change this makes is *you cannot path backwards from your destination anymore*. You have to monitor your robot's cooldown during the pathing algorithm, and if it lands on a current with too high a cooldown, it will move along that current. Resource gathering was also a lot trickier. Carriers can move in, grab a couple resources, and move out, so simply sitting at a well is wasting time for other carriers. Plus, bringing resources to headquarters, upgrading wells, and trying to concoct an elixir well all add complexity to resource gathering.

Our pathing and resource gathering is certainly not complete. By Sunday it was good enough to begin work on other things, but we didn't have upgrading wells/making elixir until a few hours before the tournament deadline, and updated our pathing algorithm several times in the twenty four hours leading up to the tournament. I'll explain the difficulties pathing presented, but the details on resource gathering and communications will have to stay secret.

-----

### Pathing is a Bytecode-sum Game

Each robot only gets 10k bytecode per turn so we need to be incredibly efficient. There are 69 locations in most units vision radii, and a for loop takes ~10 bytecode per iteration just in overhead costs, so you can easily blow 5% of your compute just looping through the list of locations.

One common trick is to rearrange for loops, as decrementing from the top takes two less bytecode.
```
public static void main(String[] args) throws Exception {
	for (int i = 0; i < 10; i++); // 9 bytecode per iteration
	for (int i=10; --i>=0;); // 7 bytecode per iteration
}


Compiled Bytecode:

public static void main(java.lang.String[]) throws java.lang.Exception;
    Code:
       0: iconst_0
       1: istore_1
       2: iload_1
       3: bipush        10
       5: if_icmpge     14
       8: iinc          1, 1
      11: goto          2
      14: bipush        10
      16: istore_1
      17: iinc          1, -1
      20: iload_1
      21: iflt          27
      24: goto          17
      27: return
```

However, you can do much better, with a giant switch statement! Instead of checking each time if the loop is ended, why not copy and paste your code ten times? Putting it in a switch statement lets you `goto` any initial value for `i` instead of just ten, with a measly two bytecode per "loop" iteration.
```
int i;
switch(maxValue) {
default:
case 10: i=9;
case 9: i=8;
case 8: i=7;
...
case 1: i=0;
case 0:
}

Compiled Bytecode:

29: tableswitch   { // 0 to 10
 0: 112
 1: 110
 2: 108
 3: 106
 ...
10: 88
default: 88
}
88: bipush        9
90: istore_1
91: bipush        8
93: istore_1
94: bipush        7
96: istore_1
97: bipush        6
99: istore_1
100: iconst_5
101: istore_1
102: iconst_4
103: istore_1
104: iconst_3
105: istore_1
106: iconst_2
107: istore_1
108: iconst_1
109: istore_1
110: iconst_0
111: istore_1
```

We make use of this heavily in pathing. The algorithm we chose was [Dijkstra's algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm), but there's a chance we'll switch over to A* in the future. (A* is more expensive per loop, but on average takes less iterations to find a path.) Both of these require some kind of priority queue.

Java's built in `PriorityQueue` is far too inefficient. I don't know how many bytecode it takes per operation, but it's probably in the hundreds. So we implemented our own binary heap, taking advantage of the fact that our queue has exactly 69 slots to unroll sifting up/down. Overall we got ~15 bytecode per `add` and ~10 bytecode per `pop`. Combined with the switch statement trick, we should be able to add and pop all locations in ~2k bytecode.

Of course, we also needed to read in all the map info. This unfortunately takes *a lot* of bytecode, and there's nothing that can be done about it. It costs $200$ to sense all the information in vision range, but then we have to call functions
```
info.getCooldownMultiplier() // 5 bytecode
info.getCurrentDirection() // 5 bytecode
info.getMapLocation() // 1 bytecode
info.isPassable() // 5 bytecode
```
for each location as well as format it correctly. For example, you need to multiply the cool down multiplier by your actual cooldown, transform the map location into an encoding for the queue, etc. We got this down to ~35 bytecode per map location, but that still costs ~2.5k bytecode. We also do a few other things to prepare the Dijkstra algorithm, amounting to 3k bytecode total. We've already used up half our bytecode, and we haven't implemented any pathing logic yet.

I won't go into the pathing logic, as that'll stay secret until the competition is over, but it ended up eating all the remaining bytecode, bringing it up to just over 10k. Before all of the bytecode saving measures, it was closer to 35k, so this was a massive improvement, but still not good enough. In the future we'll explore using A* or shortening the logic, but for now we did a dirty trick of ignoring locations behind the robot. Why waste compute on locations you likely won't ever go to? Overall, our pathing took about 6k bytecode.

Then on Saturday night/Sunday morning we tested our bot with \[BruteForcer\] SiestaGuru's map "zzCornerTrouble" and our bot got completely stuck. If it can't see around the obstacle it can't path around it, so we had to implement a bug navigator in case Dijkstra fails. Luckily this is pretty cheap, so no worries there.

-----
### 24hr Hack~~job~~athon

After finishing up most of our infrastructure, we had <1 day to actually fight back. At first we tried a unit cloud, having our units spread out until they saw an enemy, then all converging together on the enemy. Big mistake. Launchers can fire twice as fast as move, so by the time the other units converged the initial launchers were dead. Teams that grouped their launchers together easily picked ours off one by one.

So, what if we make like electrons, and tell our launchers to move towards each other and away from other units? Still no bueno. Each HQ starts with 200 mana to spend, so most teams (like us) immediately build three launchers at each HQ. Well, [Lanchester's square law](https://en.wikipedia.org/wiki/Lanchester%27s_laws#Lanchester's_square_law) says 6 > 3 + 3, so on maps with multiple HQs our soldiers would make their own isolated groups of three and die from a combined mass from the enemy.

In the end we just told them to all move to the center of the map, to guarantee one big clumb. When they're actually forced to fight they don't micro super well, but hopefully we'll win through superior numbers.

Then we had a few other things to check off:
- [x] Decent build order.
- [x] Make boosters move.
- [x] Run away from fights we can't win.
- [ ] Destabilizers? Not sure if they actually attack...

In the hours leading up to the deadline, we were exhausted, copying code, and several times uploaded bots to the scrimmage servers that resign at round 300. Oops, a quick elo gain for our seeding became a massive drop. Finally, at 6:14PM, 45 minutes before the submission deadline, we uploaded our finalized version (with no `rc.resign()` this time).

Twenty minutes later we discovered a bug in Teh Dev's engine. I guess we were the first team to craft anchors with elixir and actually use them? They (hopefully) fixed it internally, but that gives us hope our bot might not be so bad.

-----
### Example Match
![[games_presprint.gif]]