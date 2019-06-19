# peermaps

peer to peer cartography

* https://peermaps.org
* https://substack.net
* https://bits.coop

---
# mapping projects

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

* desktop and mobile map editor
* offline p2p sync

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

each tree is empty or full

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
  FULL      FULL     FULL     FULL     EMPTY
```

when staging is full:

* combine with every FULL tree before the first EMPTY
* into the first EMPTY

```
B * (2**0 + 2**0 + 2**1 + 2**2) = B * (2**3)
```

---
# log structured merge trees

if batches can be larger than staging,
minimize the number of merge operations with a planning algorithm

``` rust
let p = plan(
  // staging deconstructed into bitfield:
  &bits![0,1,1,0,0,0,1,0,1,1,1,0,1,0,0],
  // tree size increasing by powers of 2: empty (0) or full (1)
  &bits![1,0,1,1,1,0,1,0,1,0,0,1,1,1,0]
);
// merge operations: (into_tree, staging_inputs, tree_inputs)
assert_eq!(p, vec![
  (1,vec![1],vec![]),
  (5,vec![2],vec![2,3,4]),
  (7,vec![6],vec![6]),
  (14,vec![8,9,10,12],vec![8,11,13])
]);
```

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

other goals:

* updates shouldn't replace too much old cache
* fast batch write speeds

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
# eyros: demo

* demo: grow
* demo: merge

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

---
## swarmhead - demo

[demo time]

---
# graphics layer

* assemble attribute buffers for stuffing into the gpu
* minimal number of draw calls
* allow custom proj4 projections

---
# graphics layer: tiles?

tiles... maybe not needed?

instead: turn database queries into webgl buffers

maybe we will also have tiles eventually too

---
# graphics: designed around rendering modes

* points - `GL_POINTS` (or `GL_TRIANGLES` on a 2-triangle quad)
* lines - packs into `GL_TRIANGLE_STRIP`
* triangles - `GL_TRIANGLES` (fill) and `GL_TRIANGLE_STRIP` (border)

Q: triangulate at runtime or pack into file format?

https://github.com/peermaps/docs/blob/master/bufferschema.md

---
# mixmap: demo

* https://substack.net/wip/shadytile0.html
* dat://81e8ab9b6944e5263ff517be5e9c002446a8a881eff74c1df9ad3fbd6d875da2/
* https://mixmap-ne2swr-cities-demo-substack.hashbase.io/

---

__END__

