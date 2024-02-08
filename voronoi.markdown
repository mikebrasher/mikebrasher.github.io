---
layout: default
title: Voronoi Diagram
permalink: /voronoi/
---
{:refdef: style="text-align: center;"}
![maze](/assets/maze.png)
{:refdef }

This is part 1 and you can read [part 2](/fmm.markdown) here. The source code for this project is on my [github](https://github.com/mikebrasher/robonav).

## Overview

There are several algorithms that perform path planning given some sort of graph representation of the problem space, such as A*, Dijkstra, etc.  However, when determining paths for robot navigation, this graph may not exist, so it is useful to begin by extracting a Voronoi Diagram to define the safest areas furthest away from obstacles, then obtain the actual path using the Fast Marching method, as noted in Garrido et. al. [^garrido]:

“The method combines map-based and sensor-based planning operations to provide a reliable motion plan, while it operates at the sensor frequency. The main characteristics are speed and reliability, since the map dimensions are reduced to an almost unidimensional map and this map represents the safest areas in the environment for moving the robot.”


## Voronoi Diagram

The Voronoi diagram of a set of points in 2D defines planar regions that are closest to each point according to some distance metric, usually Euclidian distance. Several algorithms have been proposed to solve this problem; including Fortune’s advancing front algorithm, and generating the dual of a Delauney triangulation. Beyond Voronoi diagrams of points, generalized Voronoi diagrams define similar regions for line segments, arcs, and even three dimensions. This project generates 2D generalized Voronoi diagrams for sets of point and line segments following the method of HeHu09 [^held].

In summary, staring from an initial diagram, each point site is added sequentially generating intermediate diagrams, then each line site is added sequentially. This method follows the incremental topology-oriented approach of Sugihara and Iri [^sugihara], which makes the algorithm reliable on floating point arithmetic, and the algorithm executes quickly in O(n log n) time.

The first step in this implementation is to extract a closed defining polygon from an input array of cube primitives which define the maze shown above. This is done in an ad-hoc way assuming the blocks are directly adjacent each other and do not overlap. This method which probably would not be robust in general, but works well for the given inputs. This simple closed polygon is then taken as the direct input to the Voronoi diagram.

Each vertex of the polygon shown below is taken as a point site, and each edge of the polygon is taken as a line site. Based on the bounding box of these, four dummy Voronoi noides are placed outside, along with one dummy site in the center.

![closed polygon polygon](/assets/polygon.png)
{:refdef }
**<center>Closed Defining Polygon</center>**

 

From [^held]

“We require the Voronoi diagram under construction to fulfill the following topological conditions:
- Every site has its own Voronoi cell
- Every Voronoi cell is connected
- The Voronoi cell of an open line segment [] is adjacent to the Voronoi cells of its endpoints”

## Point Sites

At any point in time, the current Voronoi diagram will consist of Voronoi nodes which are equidistant to at least three sites, and Voronoi edges which are equidistant to exactly two sites and connect the nodes together. For any point, define a clearance disk as the closed disk centered at p whose radius is the smallest distance to any site.  Each site, s, is then added incrementally via the following method:

1. Find the Voronoi seed node which whose clearance disk is most violated by s, i.e. maximize CD(seed) – dist(seed, s)
2. Recursively descend a depth-first tree to find other nodes whose clearance disks also cover s, and delete the edges connecting them
3. For each edge spanning the new Voronoi cell, calculate the intersection point between the edge and the bisectors of s and the edge’s sites, and make these intersection new nodes
4. Connect these hanging nodes with edges to define the new cell

 

When adding point sites, step 3 doesn’t require using both bisectors of the edge and the site, but it will be helpful when we get to adding line sites. In any case, each bisector should intersect the edge at the same point, but it is only added once. The bisector between any two point sites will just be the line going through the midpoint of the two sites, and the intersection of two lines is simple to calculate. An example of this step is shown below.

![Three Point Sites](/assets/three_points.png)
{:refdef }
**<center>Three Point Sites</center>**

Starting with nodes s0 and s1, their bisector is shown as the dashed black line. An existing edge connects nodes n0 and n1 on this bisector. The clearance disk of n1 is show as the red circle, and the new point site to be inserted, s, lies within this clearance disk. The bisector of s and s0, and the bisector of s and s1 are both shown as the dashed blue lines. Each bisector intersects the previous edge at x. n1 is deleted from the node list, and a new node at x is added. The edge from n0 to n1 is deleted from the edge list, and a new edge from n0 to x is added.

Once all appropriate nodes have been deleted, and the remaining edges have been intersected with the new cell, all that remains is to connect these hanging nodes to form a topologically connect region. This happens as below, by matching the sites that define the edges together and calculating bisectors between these sites and s.

![Connect A New Voronoi Cell](/assets/connect_cell.png)
{:refdef }
**<center>Connect A New Voronoi Cell</center>**

As each new node is created in the previous step, mark them as disconnected.  As shown above, we are currently connecting node n3, which terminates edges e2 and e23.  Edge e2 is defined by sites s and s2, edge e23 is defined by sites s2 and s3, and the node n3 is defined by all three sites.  There is no edge currently in the edge list that is defined by sites s and s3, so we need to create it.  We look through the disconnected node list to find node n4 which is also defined by s3, and generate an edge from n3 to n4 along the dashed blue bisector of s and s3.  We then mark n3 as connected and continue with the next disconnected node, n4.

 

The algorithm is shown inserting all point sites for the example maze below.  Point sites are shown as blue circles, Voronoi nodes as red circles, and Voronoi edges as green lines.

![Inserting Point Sites](/assets/pointsites.gif)
{:refdef }
**<center>Inserting Point Sites</center>**


## Line Sites

Inserting line sites proceeds following the same algorithm, just calculating the bisectors and intersections becomes a bit more involved.  As noted before, the bisector of two point sites is a line.  However, the bisector of a point site and a line site is a parabola, and it’s convenient to define this directly by its vertex an directrix, i.e. the point site and line site.  Two line sites actually have a pair of bisectors, two lines that form an x.  It’s simpler to intersect both bisectors with the current edge, and then throw out any intersections outside the line sites’ cones of influence.  This zone is simply the set of points that lie within the line site when projected onto the underlying line.

 

The intersection of two lines is a single point, and unless they are parallel, will always exist.  The intersection of a line and a parabola can be defined by a quadratic equation, and there can be 0, 1, or 2 solutions.  As long as care is taken in the calculation, these can be taken directly from the exact solution.  Generally two solutions will exist, but only one of these solution will be within the cone of influence of the line site.

 

The intersection of two parabolas is defined by a quartic equation, and even though analytic solutions exist, they are complicated and prone to numerical instability when calculating in floating point.  One could implement a root finding algorithm such as Baristow’s method, but this appears to be unnecessary.  In order to have two parabolic bisectors, we need to be currently looking at two line sites and one point site.  However, we can just use the third bisector between the two line sites and calculate its intersection with either parabola.  This is why earlier we considered both bisectors of the inserted site.  Practically we implement the parabola-parabola intersection as an empty function that returns no intersection and rely on the other bisector to calculate it.

 

The algorithm is shown inserting all line sites for the example maze below.  Additionally we split the parabolic edges at their apex to reduce cycles when descending the tree as in [^held]

![Inserting Line Sites](/assets/linesites.gif)
{:refdef }
**<center>Inserting Line Sites</center>**

 
## References

[^garrido]: Garrido, Santiago & Moreno, Luis & Blanco, Dolores & Jurewicz, Piotr. (2011). Path Planning for Mobile Robot Navigation using Voronoi Diagram and Fast Marching. International Journal of Robotics and Automation. 2. 154-176.

[^held]: Held M., Huber S. Topology-oriented incremental computation of Voronoi diagrams of circular arcs and straight-line segments. Computer Aided Design 2009; 41(5):327-38

[^sugihara]: Sugihara K, Iri M. Construction of the Voronoi diagram for ‘one million’ generators in single-precision arithmetic. Proc IEEE 1992;80(9):147184.

