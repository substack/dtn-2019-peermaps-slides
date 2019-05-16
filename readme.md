# peermaps

peer to peer cartography

* https://peermaps.org
* https://substack.net
* https://bits.coop

---

* osm-p2p - database for offline mapping
* mapeo - offline desktop/mobile gui for building maps
* peermaps - p2p alternative to web map services (google, mapbox)

---
# personal p2p / gis timeline

```
2009: student job at geographic information network of alaska
      http://www.gina.alaska.edu/
2014: forkdb + other p2p experiments
2015: hyper{log-{index,join},kv} - kappa arch on hyperlogs
      peermaps v1 started
      osm-p2p work started
2016: osm-p2p initial release
      mixmap started
2017: peermaps v1 release
      mixmap release
2018: peermaps v2 started
2019: samgsung next stackzero grant for peermaps
```

---
# osm-p2p / kappa-osm

database for offline mapping

* osm data model (nodes, ways, relations)
* first version used hyperlog
* now uses kappa-core (hypercore-based)
* database used by https://mapeo.world

* https://www.digital-democracy.org/blog/osm-p2p/

---
# mapeo

desktop and mobile map editor
offline p2p sync

---
# peermaps

* view maps on a website (peermaps.org)
* embed maps in a page

---
# peermaps components

* eyros - multi-dimensional spatial database
* swarmhead - cooperative computing cluster
* georender - webgl rendering layer for eyros data

(plus some more...)

---
# peermaps high-level goals

* p2p + offline viewer (and later: editor)
* longevity

---
# kappa architecture

* append-only log
* materialized views built from the log data

---
# kappa architecture

* how to handle sparse data?
* generate views vs sending views around

---
# generating vs sending views

mapeo/osm-p2p;

* always generate from the log

peermaps:

* distribute pre-computed planet.osm view (sparse)
* local edit log builds into a local view  

---
# eyros

multi-dimensional spatial database

previous multi-dimensional spatial databases
(all used by mapeo with real users):

* kbd-tree
* bkd-tree (currently using this one)

---
# eyros

but first... trees!

---
# binary tree



```
    level 0:         _    8   _
                    /          \
    level 1:       4            12
                 /   \        /   \
    level 2:    2     6     10     15
               / \   / \   /  \   /  \
    level 3:  0   3 5   7 9   11 14  17
```

---
# kd-tree

multidimensional binary trees

```
[x] level 0:         _    8   _
                    /          \
[y] level 1:       4           12
                 /   \        /   \
[z] level 2:    2     6     10     15
               / \   / \   /  \   /  \
[x] level 3:  0   3 5   7 9   11 14  17
```

dimension under consideration alternates each level 

---
# b-tree

binary tree with >2 children possible on each node

```
         [ 7,16, , ]
      __/   |  \______ 
     /      |         \
[1,2,5,6 ][9,12, , ][18,21, , ]
```

(branch factor 5)

---
# bkd tree

* combines k-d trees with b-trees
* forest of trees LSM technique

---
# log structured merge trees

bkd approach: forest of trees

```
                                         /\
                               /\       /  \
                      /\      /  \     /    \
             /\      /  \    /    \   /      \
[staging]    B*1      B*2      B*4      B*8

 B*2**0    B*2**0   B*2**1   B*2**2   B*2**3 ...
           tree 0   tree 1   tree 2   tree 3...

* each tree is empty or full

```

---
# log structured merge trees

```
                                         /\
                               /\       /  \
                      /\      /  \     /    \
             /\      /  \    /    \   /      \
[staging]    B*1      B*2      B*4      B*8
  B*1      tree 0   tree 1    tree 2   tree 3
            FULL     FULL     FULL     EMPTY    
```

when staging is full:

* combine with every FULL tree before the first EMPTY
* into the first EMPTY

```
B * (2**0 + 2**0 + 2**1 + 2**2) = B * (2**3)
```

---
# log structured merge trees


```
           _n_
          \    k    n+1
 given:   /__ 2  = 2    - 1
          k=0

  
  2**0 + 2**1 + ... + 2**n = 2**(n+1) - 1 

  2**0 + 2**1 + ... + 2**n = 2**(n+1) - 2**0

  2**0 + 2**0 + 2**1 + ... + 2**n = 2**(n+1)
  
  (previously:)

  B * (2**0 + 2**0 + 2**1 + 2**2) = B * (2**3)

```

https://math.stackexchange.com/questions/1990137/the-idea-behind-the-sum-of-powers-of-2#1990146

---
# log structured merge trees

if batches can be larger than staging,
minimize the number of

---
# interval tree

stores intervals (min,max) intead of points

good for geospatial data: bounding rectangles in 2d

```
    level 0:         _    8   _
                    /          \
    level 1:       4            12
                 /   \        /   \
    level 2:    2     6     10     15
               / \   / \   /  \   /  \
    level 3:  0   3 5   7 9   11 14  17
```

same as a binary tree when an interval is completely
  to the left or right

but! each node points at a set of intersecting intervals

---
# eyros

* heavily based on bkd-tree paper design
* with interval tree capabilities

---
# eyros: branch

```
[length: u32 (bytes)]                 P3
[pivots: T[N]]                      _/ | \_
[data bitfield: u8[(N+BF+7)/8]]   _/   |   \_
[intersecting: u64[N]]           /    I3     \
[buckets: u64[BF]]              P1            P5
                              /  | \        /  | \
                            P0  I1  P2    P4  I5 P6
                          / | \   / | \  / | \  / | \
                         B0 I0  B1 I2  B2 I4  B3 I6 B4
```

---
# eyros: data


```
# block 0
[length: u32 (bytes)]
[point0][value0]
[point1][value1]
[point2][value2]
...
# block 1
[length: u32 (bytes)]
[point0][value0]
[point1][value1]
[point2][value2]

..and so on...
```

---
#

* based heavily on bkd tree paper
* with interval tree capabilities

each branch is a mini-tree:

---
# eyros

* data
* ranges
* tree0, tree1, tree2, ...

---
# eyros

a point contains N coordinates

each coordinate can be a scalar:

123.456

or an interval:

(5.20,9.45)

---
# eyros

* data is written in large blocks
* each block has bounding extents
* rebuild LSM forest out of bounding extents

```
(((-151.29,-151.28),(+60.69,+60.70)),1001)
(((-147.72,-147.71),(+64.83,+64.84)),1002)
(((-88.73,-88.68),(+43.37,+43.43)),1003)
(((-88.73,-88.68),(+43.37,+43.43)),1003)
```

append-only data good for p2p distribution
and good use of cache

---
# eyros

building LSM from much smaller bounding extents fits well with map/reduce










---

# maps

---
# projections

proj4

---
# tiles?

---
# how webgl works

---
# p2p


---
---


---
# trees

---

binary tree

```
  
 /
4
```



---
# eyros

* data
* tree0, tree1, ...

---
# eyros

data block example:

```
(((-151.29,-151.28),(+60.69,+60.70)),1001)
(((-147.72,-147.71),(+64.83,+64.84)),1002)
(((-88.73,-88.68),(+43.37,+43.43)),1003)
(((-88.73,-88.68),(+43.37,+43.43)),1003)
```

---




---
# swarmhead - cooperative computing cluster

swarmhead - cooperative computing cluster

like protein folder or seti@home

---
## swarmhead - design principles

use spare capacity on existing computers!

* more computers = more metal mining, more sweatshop labor
* use a social process for trust and verification,
  not wasting electricity!

---
## swarmhead - doing more with less

* low minimums for disk and cpu
* use computers which are not always online

no node should need enough disk to store the complete archive

---
## cautionary tale: tiles@home

https://lists.openstreetmap.org/pipermail/tilesathome/2012-February/006707.html

> This means that end of February, the T at h server will be shut
> down and go away. Unless a replacement server (&admin) is being
> found, that will also likely imply that the T at H service will
> go away (together with the tiles web server) at that point.

---
## swarmhead - longevity and cooperation

* keep running so long as enough people provide care
* make it easy to provide care autonomously
* no single points of failure

---
## swarmhead - distributing power

* avoid mechanisms that accumulate power
* make the whole swarm forkable
* minimize human stress, reduce burnout

---
## swarmhead - moderation

* mods, admins
* materialized-group-auth
* invite codes to join as a worker node/writer

