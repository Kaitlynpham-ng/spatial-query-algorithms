# spatial-query-algorithms

import math
import sys
import time

points = []   # Stores every parking location as a dictionary
n = 0

with open("parking.txt", 'r') as dataset:
    for data in dataset.readlines():
        data = data.split()
        
        points.append({
            'id': int(data[0]),      # Unique parking spot identifier
            'x': float(data[1]),     # X coordinate
            'y': float(data[2])      # Y coordinate
        })
        n = n + 1

queries = []   # Stores every query location as a dictionary

with open("query_points.txt", 'r') as dataset:
    for data in dataset.readlines():
        data = data.split()
        
        queries.append({
            'id': int(data[0]),     # Unique query point identifier
            'x': float(data[1]),    # X coordinate
            'y': float(data[2])     # Y coordinate
        })

print("Total parking points loaded:", n)
print("Total queries loaded:", len(queries))
print("\n")

# R-TREE FUNCTION

B = 4

class Node(object):
    def __init__(self):
        self.id = 0
        self.child_nodes = []    # Holds child nodes when this is an internal node
        self.data_points = []    # Holds data points when this is a leaf node
        self.parent = None       # Points to the parent node; None if this is the root
        
        
        self.MBR = {
            'x1': float('inf'),   # Left boundary (min X)
            'y1': float('inf'),   # Bottom boundary (min Y)
            'x2': float('-inf'),  # Right boundary (max X)
            'y2': float('-inf')   # Top boundary (max Y)
        }

    def perimeter(self):
        return (self.MBR['x2'] - self.MBR['x1']) + (self.MBR['y2'] - self.MBR['y1'])

    def is_overflow(self):
        if self.is_leaf():
            if len(self.data_points) > B:   # Checking overflows of data points, B is the upper bound
                return True                 # Node needs to be split
            else:
                return False                # Still under capacity
        else:
            if len(self.child_nodes) > B:   # Checking overflows of child nodes, B is the upper bound
                return True                 # Overflow — exceeds B
            else:
                return False                # Still within limit

    def is_root(self):
        # The root is the only node with no parent
        if self.parent is None:
            return True                    # This node sits at the top of the tree
        else:
            return False                   # This node has a parent above it

    def is_leaf(self):
        if len(self.child_nodes) == 0:
            return True                    # No child -> this is the leaf node
        else:
            return False                   # Has children -> this is an internal node

# --- RTree class ---

class RTree(object):
    def __init__(self):
        self.root = Node()                # Tree begins with a single empty root (also a leaf)

    def insert(self, node, point):        # Insert data point p into the subtree rooted at node
        # Descend to a leaf, then add the point and handle any overflow
        if node.is_leaf():
            self.add_data_point(node, point)     # Place the point here and expand the MBR
            if node.is_overflow():
                self.handle_overflow(node)       # Split this node if it is now over capacity
        else:
            # Choose the child whose MBR grows the least, then go deeper
            sub_node = self.choose_subtree(node, point)
            self.insert(sub_node, point)         # Keep going down until we hit a leaf
            self.update_mbr(sub_node)            # Fix the MBR on the way back up

    def add_data_point(self, node, data_point):  # Append a point to a leaf and stretch its MBR
        node.data_points.append(data_point)

        if data_point['x'] < node.MBR['x1']:
            node.MBR['x1'] = data_point['x']     # Extend left boundary
        if data_point['x'] > node.MBR['x2']:
            node.MBR['x2'] = data_point['x']     # Extend right boundary
        if data_point['y'] < node.MBR['y1']:
            node.MBR['y1'] = data_point['y']     # Extend bottom boundary
        if data_point['y'] > node.MBR['y2']:
            node.MBR['y2'] = data_point['y']     # Extend top boundary

    def add_child(self, node, child):            # Attach a child node and grow the parent MBR if needed
        node.child_nodes.append(child)
        child.parent = node                      # Wire up the back-pointer to the new parent
        
        # Stretch the parent's MBR only when the child's boundary exceeds it
        if child.MBR['x1'] < node.MBR['x1']:
            node.MBR['x1'] = child.MBR['x1']     # Extend left boundary
        if child.MBR['x2'] > node.MBR['x2']:
            node.MBR['x2'] = child.MBR['x2']     # Extend right boundary
        if child.MBR['y1'] < node.MBR['y1']:
            node.MBR['y1'] = child.MBR['y1']     # Extend bottom boundary
        if child.MBR['y2'] > node.MBR['y2']:
            node.MBR['y2'] = child.MBR['y2']     # Extend top boundary

    def handle_overflow(self, node):
        node1, node2 = self.split(node)          # Split the overflowing node into two
       
        if node.is_root():
            new_root = Node()
            self.add_child(new_root, node1)      # Attach first half as left child
            self.add_child(new_root, node2)      # Attach second half as right child
            self.root = new_root                 # Promote new_root to be the tree's root
            self.update_mbr(new_root)            # Recalculate the root's MBR
        
        else:
            parent = node.parent
            parent.child_nodes.remove(node)      # Detach the old overflowing node
            self.add_child(parent, node1)        # Plug in the first replacement
            self.add_child(parent, node2)        # Plug in the second replacement
            if parent.is_overflow():             # Propagate upward if the parent also overflows
                self.handle_overflow(parent)

    def choose_subtree(self, node, point):
        if node.is_leaf():
            return node
        else:
            min_increase = sys.maxsize           # Start with a very large sentinel value
            best_child = None                    # Will hold the winning child node
            for child in node.child_nodes:
                if min_increase > self.peri_increase(child, point):
                    min_increase = self.peri_increase(child, point)     # Keep track of the smallest growth seen
                    best_child = child                                  # This child is currently the best fit
            return best_child                    # Return the child with the smallest perimeter increase

    def peri_increase(self, node, point):        # Calculate the perimeter increase after inserting the new point
        # new perimeter - original perimeter = perimeter increase
        origin_mbr = node.MBR
        x1, x2, y1, y2 = origin_mbr['x1'], origin_mbr['x2'], origin_mbr['y1'], origin_mbr['y2']
        increase = (max([x1, x2, point['x']]) - min([x1, x2, point['x']]) +
                    max([y1, y2, point['y']]) - min([y1, y2, point['y']])) - node.perimeter()
        return increase                          # Return the perimeter growth value

    def split(self, node):                       # Split a node into two nodes
        best_s1 = Node()                         # Placeholder for the first new node
        best_s2 = Node()                         # Placeholder for the second new node
        best_perimeter = sys.maxsize             # Track the smallest combined perimeter found

        # u is a leaf node
        if node.is_leaf():
            m = len(node.data_points)
            # Sort data points by X and Y to check both axis-aligned splits
            divides = [sorted(node.data_points, key=lambda data_point: data_point['x']),
                       sorted(node.data_points, key=lambda data_point: data_point['y'])]
            for divide in divides:
                # Only check split positions that keep each side at least 40% full
                for i in range(math.ceil(0.4 * B), m - math.ceil(0.4 * B) + 1):
                    s1 = Node()
                    s1.data_points = divide[0: i]              # Assign first group of points
                    self.update_mbr(s1)                        # Compute bounding box
                    s2 = Node()
                    s2.data_points = divide[i: len(divide)]    # Assign remaining points
                    self.update_mbr(s2)                        # Compute bounding box
                    # Choose the split that gives the smallest total perimeter
                    if best_perimeter > s1.perimeter() + s2.perimeter():
                        best_perimeter = s1.perimeter() + s2.perimeter()
                        best_s1 = s1
                        best_s2 = s2
       
        # u is an internal node
        else:
            m = len(node.child_nodes)
            # Sort by all four MBR edges to find the best split axis
            divides = [sorted(node.child_nodes, key=lambda child_node: child_node.MBR['x1']),
                       sorted(node.child_nodes, key=lambda child_node: child_node.MBR['x2']),
                       sorted(node.child_nodes, key=lambda child_node: child_node.MBR['y1']),
                       sorted(node.child_nodes, key=lambda child_node: child_node.MBR['y2'])]
            for divide in divides:
                for i in range(math.ceil(0.4 * B), m - math.ceil(0.4 * B) + 1):
                    s1 = Node()
                    s1.child_nodes = divide[0: i]              # Assign first group of children
                    self.update_mbr(s1)                        # Compute bounding box
                    s2 = Node()
                    s2.child_nodes = divide[i: len(divide)]    # Assign remaining children
                    self.update_mbr(s2)                        # Compute bounding box
                    # Choose the split with the smallest combined perimeter
                    if best_perimeter > s1.perimeter() + s2.perimeter():
                        best_perimeter = s1.perimeter() + s2.perimeter()
                        best_s1 = s1
                        best_s2 = s2
        # Update parent pointers for all children in the two new nodes
        for child in best_s1.child_nodes:
            child.parent = best_s1
        for child in best_s2.child_nodes:
            child.parent = best_s2

        return best_s1, best_s2                 # Return the two new nodes
        
    def update_mbr(self, node):                 # Recompute the MBR from scratch using the node's current contents
        x_list = []                             # Will collect all relevant x values
        y_list = []                             # Will collect all relevant y values
        
        if node.is_leaf():
            x_list = [point['x'] for point in node.data_points]    # x positions of stored points
            y_list = [point['y'] for point in node.data_points]    # y positions of stored points
        else:
            # Gather both edges of each child's MBR so the parent fully covers them
            x_list = [child.MBR['x1'] for child in node.child_nodes] + [child.MBR['x2'] for child in node.child_nodes]
            y_list = [child.MBR['y1'] for child in node.child_nodes] + [child.MBR['y2'] for child in node.child_nodes]
        new_mbr = {
            'x1': min(x_list),     # Leftmost x
            'x2': max(x_list),     # Rightmost x
            'y1': min(y_list),     # Bottommost y
            'y2': max(y_list)      # Topmost y
        }
        node.MBR = new_mbr         # Replace the old MBR with the freshly computed one

# BRANCH-AND-BOUND ALGORITHM

def min_dist_to_mbr(qx, qy, mbr):

    cx = max(mbr['x1'], min(qx, mbr['x2']))      # Closest x on the MBR to the query
    cy = max(mbr['y1'], min(qy, mbr['y2']))      # Closest y on the MBR to the query
    return math.sqrt((qx - cx) ** 2 + (qy - cy) ** 2)

def bab_nn(rtree, query):

    qx, qy = query['x'], query['y']  # unpack query coordinates

    # Search a subtree and return (best_point, best_dist) found there
    def search(node, best_point, best_dist):

        # If this node's MBR is already farther than our best, skip the whole subtree
        if min_dist_to_mbr(qx, qy, node.MBR) > best_dist:
            return best_point, best_dist

        # If it's a leaf, compare the query to every point stored here
        if node.is_leaf():
            for point in node.data_points:  # Loop through points in leaf
                dist = math.sqrt((qx - point['x']) ** 2 + (qy - point['y']) ** 2)  # Euclidean distance
                if dist < best_dist:        # Check if found a closer point
                    best_dist = dist        # Update best distance
                    best_point = point      # Save best point found

        # Otherwise it's an internal node: consider its nearest children first
        else:
            # For each child's MBR, calculate the distance from it to the query
            child_bounds = [(min_dist_to_mbr(qx, qy, c.MBR), c) for c in node.child_nodes]
            # Explore the nearest children first
            child_bounds.sort(key=lambda t: t[0])

            for child_dist, child in child_bounds:
                if child_dist > best_dist:  # prune/skip if this child's MBR is farther than best distance
                    continue
                # Otherwise, go down into this child and keep searching
                best_point, best_dist = search(child, best_point, best_dist)

        return best_point, best_dist  # After checking all, return the best result found

    # Start the search from the root; no best point yet, distance is infinity
    return search(rtree.root, None, float('inf'))

# ALGORITHM 1: Sequential Scan
 
print("-------------------")
print("ALGORITHM 1: Sequential Scan Based Method")
print("-------------------\n")
 
start_time = time.time()  # Record start time
 
with open("task_1_1_output.txt", 'w') as output_file:
    output_file.write("ALGORITHM 1: Sequential Scan Based Method\n")
    output_file.write("=" * 60 + "\n")
 
    for query in queries:
        best_point = None
        best_dist = float('inf')  # Initialise with infinity so any real distance wins
 
        # Check every parking spot and keep the one with the smallest distance
        for point in points:
            dist = math.sqrt((query['x'] - point['x']) ** 2 + (query['y'] - point['y']) ** 2)
            if dist < best_dist:
                best_dist = dist                # New shortest distance
                best_point = point              # Nearest parking spot so far
 
        line = "id=" + str(best_point['id']) + ", x=" + str(best_point['x']) + ", y=" + str(best_point['y']) + " for query " + str(query['id'])
        print(line)
        output_file.write(line + "\n")
 
end_time = time.time()                          # Record end time
total_time = round(end_time - start_time, 4)
time_line = "Total running time for all 200 queries: " + str(total_time) + " seconds"
print(time_line)

with open("task_1_1_output.txt", 'a') as output_file:
    output_file.write(time_line + "\n")
 

# ALGORITHM 2: BaB with R-tree
 
print("\n-------------------")
print("ALGORITHM 2: Branch-and-Bound (BaB) Algorithm")
print("-------------------\n")
 
rtree = RTree()                                 # Start with an empty tree
print("build R-Tree: ")
print("\n")
for point in points:
    rtree.insert(rtree.root, point)             # Add each parking spot to the tree
print("This is the MBR of the root", rtree.root.MBR)
print("\n")
 
start_time = time.time()                        # Record start time

with open("task_1_2_output.txt", 'w') as output_file:
    output_file.write("ALGORITHM 2: Branch-and-Bound (BaB) Algorithm\n")
    output_file.write("=" * 60 + "\n")
 
    for query in queries:
        # Find the nearest parking spot for this query using BaB on the R-tree
        best_point, best_dist = bab_nn(rtree, query)
 
        line = "id=" + str(best_point['id']) + ", x=" + str(best_point['x']) + ", y=" + str(best_point['y']) + " for query " + str(query['id'])
        print(line)
        output_file.write(line + "\n")
 
end_time = time.time()  # record end time
total_time = round(end_time - start_time, 4)
time_line = "Total running time for all 200 queries: " + str(total_time) + " seconds"
print(time_line)

with open("task_1_2_output.txt", 'a') as output_file:
    output_file.write(time_line + "\n")
 
# ALGORITHM 3: BaB with Divide-and-Conquer

print("\n-------------------")
print("ALGORITHM 3: BaB with Divide-and-Conquer")
print("-------------------\n")
 
x_min = min(p['x'] for p in points)            # Leftmost x in the dataset
x_max = max(p['x'] for p in points)            # Rightmost x in the dataset
x_split = (x_min + x_max) / 2                  # Halfway point used as the dividing line
 
left_points  = [p for p in points if p['x'] <= x_split]         # Points on or left of the line
right_points = [p for p in points if p['x'] >  x_split]         # Points strictly right of the line
 
print("Dividing dataset by X dimension:")
print("x_min =", x_min, "| x_max =", x_max, "| x_split =", x_split)
print("Left subspace :", len(left_points),  "points (x <=", x_split, ")")
print("Right subspace:", len(right_points), "points (x > ", x_split, ")")
print("\n")
 
print("Building R-tree for left subspace...")
rtree_left = RTree()                            # Empty tree for the left half
for p in left_points:
    rtree_left.insert(rtree_left.root, p)       # Populate left R-tree
print("MBR of left R-tree root:", rtree_left.root.MBR)
print("\n")
 
print("Building R-tree for right subspace...")
rtree_right = RTree()                           # Empty tree for the right half
for p in right_points:
    rtree_right.insert(rtree_right.root, p)     # Populate right R-tree
print("MBR of right R-tree root:", rtree_right.root.MBR)
print("\n")
 
left_dict  = {}  # Nearest-neighbour results from the left R-tree
right_dict = {}  # Nearest-neighbour results from the right R-tree
 
start = time.time()                             # Record start time
 
for query in queries:
    qid = str(query['id'])                      # Use query ID as the key
 
    nn_left,  dis_left  = bab_nn(rtree_left,  query)        # Search left half
    left_dict[qid]  = [nn_left,  dis_left]                  # Cache the result
 
    nn_right, dis_right = bab_nn(rtree_right, query)        # Search right half
    right_dict[qid] = [nn_right, dis_right]                 # Cache the result
 
with open("task_1_3_output.txt", 'w') as output_file:
    output_file.write("ALGORITHM 3: BaB with Divide-and-Conquer\n")
    output_file.write("=" * 60 + "\n")
 
    for query in queries:
        qid = str(query['id'])
        nn_left,  dis_left  = left_dict[qid]                # Retrieve left-half result
        nn_right, dis_right = right_dict[qid]               # Retrieve right-half result
 
        # Select the candidate with the smaller distance; left wins on a tie
        if dis_left <= dis_right:
            best_point = nn_left
        else:
            best_point = nn_right
 
        line = "id=" + str(best_point['id']) + ", x=" + str(best_point['x']) + ", y=" + str(best_point['y']) + " for query " + qid
        print(line)
        output_file.write(line + "\n")
 
end = time.time()                              # Record end time
total_time = round(end - start, 4)
time_line = "Total running time for all 200 queries: " + str(total_time) + " seconds"
print(time_line)

with open("task_1_3_output.txt", 'a') as output_file:
    output_file.write(time_line + "\n")


import sys
import math  
import time  
from tqdm import tqdm

import numpy as np  

# PART 1: R-TREE INDEX

B = 4

class Node(object):

    def __init__(self):
        self.id = 0  # Optional node identifier (not used in logic)
        self.child_nodes = []  # Non-empty for internal nodes; empty at leaves
        self.data_points = []  # Leaf entries: list of {'id', 'x', 'y'} dicts
        self.parent = None  # Parent pointer for split and re-linking
        # MBR corners; (-1, -1) until first point/child is added
        self.MBR = {
            'x1': -1,  # Left (minimum x)
            'y1': -1,  # Bottom (minimum y)
            'x2': -1,  # Right (maximum x)
            'y2': -1,  # Top (maximum y)
        }

    def perimeter(self):
        return (self.MBR['x2'] - self.MBR['x1']) + (self.MBR['y2'] - self.MBR['y1'])  # Sum of width and height

    def is_overflow(self):
        if self.is_leaf():  # Leaf stores data points
            return len(self.data_points) > B  # Overflow when too many points
        return len(self.child_nodes) > B  # Internal node: too many children

    def is_root(self):
        return self.parent is None  # Root has no parent

    def is_leaf(self):
        return len(self.child_nodes) == 0  # Leaf has no child nodes


class RTree(object):

    def __init__(self):
        self.root = Node()  # Start with a single empty root node

    def insert(self, node, point):
        if node.is_leaf():  # Reached a leaf — store the point here
            self.add_data_point(node, point)  # Append point and grow leaf MBR
            if node.is_overflow():  # Leaf exceeded capacity B
                self.handle_overflow(node)  # Split leaf and fix tree upward
        else:  # Internal node — descend to best child
            # Step: pick the child whose MBR grows least (min perimeter increase)
            sub_node = self.choose_subtree(node, point)  # Child with smallest MBR growth
            self.insert(sub_node, point)  # Recurse into that child
            self.update_mbr(sub_node)  # Refresh parent MBR after child changed

    def add_data_point(self, node, data_point):
        node.data_points.append(data_point)  # Store point in leaf entry list
        if data_point['x'] < node.MBR['x1']:  # New point extends left bound
            node.MBR['x1'] = data_point['x']  # Update minimum x
        if data_point['x'] > node.MBR['x2']:  # New point extends right bound
            node.MBR['x2'] = data_point['x']  # Update maximum x
        if data_point['y'] < node.MBR['y1']:  # New point extends bottom bound
            node.MBR['y1'] = data_point['y']  # Update minimum y
        if data_point['y'] > node.MBR['y2']:  # New point extends top bound
            node.MBR['y2'] = data_point['y']  # Update maximum y

    def add_child(self, node, child):
        node.child_nodes.append(child)  # Link child under this internal node
        child.parent = node  # Set back-pointer from child to parent
        if child.MBR['x1'] < node.MBR['x1']:  # Child extends parent left
            node.MBR['x1'] = child.MBR['x1']  # Expand parent x1
        if child.MBR['x2'] > node.MBR['x2']:  # Child extends parent right
            node.MBR['x2'] = child.MBR['x2']  # Expand parent x2
        if child.MBR['y1'] < node.MBR['y1']:  # Child extends parent bottom
            node.MBR['y1'] = child.MBR['y1']  # Expand parent y1
        if child.MBR['y2'] > node.MBR['y2']:  # Child extends parent top
            node.MBR['y2'] = child.MBR['y2']  # Expand parent y2

    def handle_overflow(self, node):
        node1, node2 = self.split(node)  # Partition entries into two nodes
        if node.is_root():  # Overflow at root — tree grows taller
            new_root = Node()  # Create new root above the two halves
            self.add_child(new_root, node1)  # First half becomes child
            self.add_child(new_root, node2)  # Second half becomes child
            self.root = new_root  # Point tree root to new node
            self.update_mbr(new_root)  # Set root MBR from both children
        else:  # Overflow at non-root — replace node in parent
            parent = node.parent  # Get parent of overflowing node
            parent.child_nodes.remove(node)  # Remove old combined node
            self.add_child(parent, node1)  # Insert first split node
            self.add_child(parent, node2)  # Insert second split node
            if parent.is_overflow():  # Parent may also exceed B
                self.handle_overflow(parent)  # Recursively split upward

    def choose_subtree(self, node, point):
        if node.is_leaf():  # Should not happen at internal routing step
            return node  # Return self if called on leaf
        min_increase = sys.maxsize  # Track best (smallest) perimeter increase
        best_child = None  # Child with minimum increase so far
        for child in node.child_nodes:  # Try each child
            if min_increase > self.peri_increase(child, point):  # Found better child
                min_increase = self.peri_increase(child, point)  # Store new minimum
                best_child = child  # Remember this child
        return best_child  # Route insert into best_child

    def peri_increase(self, node, point):
        origin_mbr = node.MBR  # Current bounding box
        x1, x2, y1, y2 = origin_mbr['x1'], origin_mbr['x2'], origin_mbr['y1'], origin_mbr['y2']  # Unpack corners
        return (max([x1, x2, point['x']]) - min([x1, x2, point['x']]) +  # New width minus old width
                max([y1, y2, point['y']]) - min([y1, y2, point['y']])) - node.perimeter()  # New height minus old

    def split(self, node):
        best_s1 = Node()  # Best first partition found so far
        best_s2 = Node()  # Best second partition found so far
        best_perimeter = sys.maxsize  # Lowest combined perimeter seen
        if node.is_leaf():  # Split leaf data points
            m = len(node.data_points)  # Number of points to partition
            # Try sorting by x and by y before cutting into two groups
            divides = [
                sorted(node.data_points, key=lambda data_point: data_point['x']),  # Sort by x
                sorted(node.data_points, key=lambda data_point: data_point['y']),  # Sort by y
            ]
            for divide in divides:  # For each sorted order
                for i in range(math.ceil(0.4 * B), m - math.ceil(0.4 * B) + 1):  # Valid split positions
                    s1 = Node()  # Candidate left node
                    s1.data_points = divide[0:i]  # First i points go left
                    self.update_mbr(s1)  # Compute left MBR
                    s2 = Node()  # Candidate right node
                    s2.data_points = divide[i:]  # Remaining points go right
                    self.update_mbr(s2)  # Compute right MBR
                    if best_perimeter > s1.perimeter() + s2.perimeter():  # Better split found
                        best_perimeter = s1.perimeter() + s2.perimeter()  # Save total perimeter
                        best_s1 = s1  # Remember best left
                        best_s2 = s2  # Remember best right
        else:  # Split internal node children
            m = len(node.child_nodes)  # Number of child subtrees
            divides = [  # Four sort keys for child MBR corners
                sorted(node.child_nodes, key=lambda child_node: child_node.MBR['x1']),
                sorted(node.child_nodes, key=lambda child_node: child_node.MBR['x2']),
                sorted(node.child_nodes, key=lambda child_node: child_node.MBR['y1']),
                sorted(node.child_nodes, key=lambda child_node: child_node.MBR['y2']),
            ]
            for divide in divides:  # Try each ordering
                for i in range(math.ceil(0.4 * B), m - math.ceil(0.4 * B) + 1):  # Each valid cut index
                    s1 = Node()  # Left internal node
                    s1.child_nodes = divide[0:i]  # First group of children
                    self.update_mbr(s1)  # MBR from children
                    s2 = Node()  # Right internal node
                    s2.child_nodes = divide[i:]  # Second group of children
                    self.update_mbr(s2)  # MBR from children
                    if best_perimeter > s1.perimeter() + s2.perimeter():  # Improve best split
                        best_perimeter = s1.perimeter() + s2.perimeter()  # Update best score
                        best_s1 = s1  # Store best left
                        best_s2 = s2  # Store best right

        for child in best_s1.child_nodes:  # Fix parent pointers in left split
            child.parent = best_s1  # Each child points to s1
        for child in best_s2.child_nodes:  # Fix parent pointers in right split
            child.parent = best_s2  # Each child points to s2

        return best_s1, best_s2  # Return the two new nodes

    def update_mbr(self, node):
        if node.is_leaf():  # Leaf: bound all data points
            x_list = [point['x'] for point in node.data_points]  # All x coordinates
            y_list = [point['y'] for point in node.data_points]  # All y coordinates
        else:  # Internal: bound all child MBR corners
            x_list = [child.MBR['x1'] for child in node.child_nodes] + [child.MBR['x2'] for child in node.child_nodes]
            y_list = [child.MBR['y1'] for child in node.child_nodes] + [child.MBR['y2'] for child in node.child_nodes]
        node.MBR = {  # Set rectangle from extrema
            'x1': min(x_list),  # Left edge
            'x2': max(x_list),  # Right edge
            'y1': min(y_list),  # Bottom edge
            'y2': max(y_list),  # Top edge
        }

    def build(self, points):
        for point in tqdm(points, desc='Building R-tree', disable=True):  # One insert per point
            self.insert(self.root, point)  # Start from root each time

# PART 2: SKYLINE ALGORITHMS

DATASET_NAME = 'city3'  # 3 + 3 + 8
DATASET_DIR = 'Datasets/Task2_Datasets'  # Input data path
OUTPUT_DIR = 'output/task2'  # Where to write result files


def load_points(filepath):
    points = []  # List to store all loaded points
    with open(filepath, 'r') as f:  # Open dataset file for reading
        for line in f:  # Process each line
            parts = line.split()  # Split line into tokens (id, x, y)
            if len(parts) < 3:  # Skip lines that do not have three fields
                continue  # Go to next line
            try:
                pid = int(parts[0])  # Parse point id as integer
            except ValueError:  # Skip non-numeric id lines
                continue  # Go to next line
            points.append({  # Store one point record
                'id': pid,  # Point identifier
                'x': float(parts[1]),  # x coordinate (minimize in skyline)
                'y': float(parts[2]),  # y coordinate (minimize in skyline)
            })
    return points  # Return full dataset


def dominates(p, q):
    return (p['x'] <= q['x'] and p['y'] <= q['y'] and  # p is no worse than q on x and y
            (p['x'] < q['x'] or p['y'] < q['y']))  # p is strictly better on at least one axis


def mbr_dominated(mbr, skyline):
    for s in skyline:  # Check each current skyline point
        if s['x'] <= mbr['x1'] and s['y'] <= mbr['y1']:  # s dominates all points in MBR
            return True  # Entire region can be skipped
    return False  # Region might still contain skyline points


def mindist_point(point):
    return point['x'] + point['y']  # Lower-left style distance for minimization


def mindist_mbr(mbr):
    return mbr['x1'] + mbr['y1']  # Best-first key for lecture-style sorted candidate list


def add_skyline_point(skyline, point):
    if any(dominates(s, point) for s in skyline):  # Already dominated by existing skyline
        return skyline  # Unchanged list
    skyline = [s for s in skyline if not dominates(point, s)]  # Drop points dominated by new one
    skyline.append(point)  # Add new skyline point
    return skyline  # Updated skyline


def sequential_skyline(points):
    n = len(points)  # Total number of points
    if n == 0:  # Empty dataset
        return []  # No skyline points
    xs = np.array([p['x'] for p in points], dtype=np.float64)  # All x values as array
    ys = np.array([p['y'] for p in points], dtype=np.float64)  # All y values as array
    is_skyline = np.ones(n, dtype=bool)  # True = candidate skyline until proven dominated
    for i in range(n):  # Outer loop: check each point i
        if not is_skyline[i]:  # Already known to be dominated
            continue  # Skip further work for i
        dominated = (  # Vectorized: does any j dominate i?
            (xs <= xs[i]) & (ys <= ys[i]) &  # j is at least as good as i on both axes
            ((xs < xs[i]) | (ys < ys[i])) &  # j is strictly better on x or y
            (np.arange(n) != i)  # Exclude comparing i with itself
        )
        if dominated.any():  # At least one dominator exists
            is_skyline[i] = False  # Point i is not on the skyline
    skyline = [points[i] for i in range(n) if is_skyline[i]]  # Collect surviving points
    skyline.sort(key=lambda p: (p['x'], p['y']))  # Sort output for consistent reporting
    return skyline  # Final skyline list


def bbs_skyline(rtree):
    skyline = []  # Running set of skyline points found so far
    candidates = []  
    
    root = rtree.root  # Start traversal from tree root
    if root.is_leaf():  # Tree is a single leaf (small dataset)
        for p in root.data_points:  # Push every point in root leaf
            candidates.append(('point', p))  # Add point candidate
    else:  # Root is internal
        for child in root.child_nodes:  # Push each top-level subtree MBR
            candidates.append(('node', child))  # Add node candidate

    while candidates:  # Process until candidate list is empty
        # Slide-style BBS: keep candidate list sorted by mindist each iteration.
        candidates.sort(key=lambda entry: mindist_mbr(entry[1].MBR) if entry[0] == 'node' else mindist_point(entry[1]))
        entry = candidates.pop(0)  # Remove most promising entry (smallest mindist)
        if entry[0] == 'node':  # Entry is an R-tree node (region)
            node = entry[1]  # Actual node object
            if mbr_dominated(node.MBR, skyline):  # Whole subtree can be pruned
                continue  # Skip this node
            if node.is_leaf():  # Leaf: enqueue individual points
                for p in node.data_points:  # Each point in leaf
                    if not any(dominates(s, p) for s in skyline):  # Not already dominated
                        candidates.append(('point', p))  # Schedule point for visit
            else:  # Internal node: enqueue children
                for child in node.child_nodes:  # Each child subtree
                    if not mbr_dominated(child.MBR, skyline):  # Child region not pruned
                        candidates.append(('node', child))  # Schedule subtree
        else:  # Entry is a single point
            p = entry[1]  # Point dict
            if not any(dominates(s, p) for s in skyline):  # Point may be on skyline
                skyline = add_skyline_point(skyline, p)  # Insert and remove dominated old points

    skyline.sort(key=lambda p: (p['x'], p['y']))  # Sort for output consistency
    return skyline  # BBS skyline result


def merge_skylines_1d(skylines):
    combined = []  # Pool of all candidate points from subproblems
    for s in skylines:  # For each partial skyline list
        combined.extend(s)  # Append all its points
    combined.sort(key=lambda p: (p['x'], p['y']))  # Sort by x then y
    result = []  # Global skyline after merge
    min_y = float('inf')  # Best (minimum) y seen so far to the left
    for p in combined:  # Scan in increasing x
        if p['y'] < min_y:  # Not dominated in y by any point with smaller x
            result.append(p)  # Keep as global skyline point
            min_y = p['y']  # Tighten minimum y for later points
    return result  # Merged skyline


def bbs_divide_and_conquer(points):
    xs = [p['x'] for p in points]  # All x coordinates for median
    median = float(np.median(xs))  # Split plane at median x
    left = [p for p in points if p['x'] <= median]  # Left half points
    right = [p for p in points if p['x'] > median]  # Right half points

    if not left:  # Edge case: nothing on left of median
        left, right = right[:len(right) // 2], right[len(right) // 2:]  # Force balanced split
    if not right:  # Edge case: nothing on right of median
        right = left[len(left) // 2:]  # Take second half as right
        left = left[:len(left) // 2]  # First half as left

    rtree_l = RTree()  # R-tree for left partition
    rtree_l.build(left)  # Insert all left points
    skyline_l = bbs_skyline(rtree_l)  # BBS skyline on left half

    rtree_r = RTree()  # R-tree for right partition
    rtree_r.build(right)  # Insert all right points
    skyline_r = bbs_skyline(rtree_r)  # BBS skyline on right half

    return merge_skylines_1d([skyline_l, skyline_r])  # Combine into global skyline


def write_output(filepath, algorithm_name, skyline, elapsed, start_time_str):
    with open(filepath, 'w', encoding='utf-8') as f:  # Open result file for writing
        f.write(f"Algorithm: {algorithm_name}\n")  # Header: which algorithm ran
        f.write(f"Dataset: {DATASET_NAME}\n")  # Header: dataset name
        f.write(f"Start time: {start_time_str}\n")  # Start timestamp for this algorithm run
        f.write(f"Skyline size: {len(skyline)}\n")  # Number of skyline points
        f.write(f"Execution time (seconds): {elapsed:.6f}\n")  # Measured runtime
        f.write("\nSkyline points (id, x, y):\n")  # Section label for point list
        for p in skyline:  # One line per skyline point
            f.write(f"id={p['id']}, x={p['x']}, y={p['y']}\n")  # Formatted coordinates


def main():
    dataset_path = f'{DATASET_DIR}/{DATASET_NAME}.txt'  # Full path to input file

    print(f"Task 2 - Skyline Search")  # Console title
    print(f"Dataset: {DATASET_NAME}")  # Show which city file is used
    print(f"Loading: {dataset_path}")  # Show load path

    points = load_points(dataset_path)  # Read all points from disk
    print(f"Loaded {len(points)} points\n")  # Report dataset size

    # --- Step A: Sequential scan ---
    print("Running sequential scan...")  # Status message
    start_seq = time.time()  # Start timestamp for output + runtime
    start_seq_str = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(start_seq))  # Start time
    t0 = time.time()  # Start timer
    skyline_seq = sequential_skyline(points)  # Run O(n^2) baseline
    t_seq = time.time() - t0  # Elapsed seconds
    out_seq = f'{OUTPUT_DIR}/task2_sequential_{DATASET_NAME}.txt'  # Output path
    write_output(out_seq, 'Sequential Scan', skyline_seq, t_seq, start_seq_str)  # Save results to file
    print(f"  Skyline size: {len(skyline_seq)}, time: {t_seq:.4f}s")  # Print summary
    print(f"  Output: {out_seq}\n")  # Print output file location

    # --- Step B: Build one R-tree, run BBS ---
    print("Running BBS with R-tree...")  # Status message
    start_bbs = time.time()  # Start timestamp for output + runtime
    start_bbs_str = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(start_bbs))  # Start time
    t0 = time.time()  # Start timer (includes tree build + search)
    rtree = RTree()  # Create empty R-tree
    rtree.build(points)  # Insert all points
    skyline_bbs = bbs_skyline(rtree)  # Best-first skyline on tree
    t_bbs = time.time() - t0  # Elapsed seconds
    out_bbs = f'{OUTPUT_DIR}/task2_bbs_{DATASET_NAME}.txt'  # Output path
    write_output(out_bbs, 'BBS Algorithm', skyline_bbs, t_bbs, start_bbs_str)  # Save results
    print(f"  Skyline size: {len(skyline_bbs)}, time: {t_bbs:.4f}s")  # Print summary
    print(f"  Output: {out_bbs}\n")  # Print output path

    # --- Step C: Divide-and-conquer + BBS per half ---
    print("Running BBS with divide-and-conquer...")  # Status message
    start_dc = time.time()  # Start timestamp for output + runtime
    start_dc_str = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(start_dc))  # Start time
    t0 = time.time()  # Start timer
    skyline_dc = bbs_divide_and_conquer(points)  # Split, BBS each half, merge
    t_dc = time.time() - t0  # Elapsed seconds
    out_dc = f'{OUTPUT_DIR}/task2_bbs_dc_{DATASET_NAME}.txt'  # Output path
    write_output(out_dc, 'BBS with Divide-and-Conquer', skyline_dc, t_dc, start_dc_str)  # Save results
    print(f"  Skyline size: {len(skyline_dc)}, time: {t_dc:.4f}s")  # Print summary
    print(f"  Output: {out_dc}\n")  # Print output path

    # --- Step D: Verify all three agree (same set of point ids) ---
    ids_seq = {p['id'] for p in skyline_seq}  # Skyline ids from sequential
    ids_bbs = {p['id'] for p in skyline_bbs}  # Skyline ids from BBS
    ids_dc = {p['id'] for p in skyline_dc}  # Skyline ids from divide-and-conquer
    if ids_seq == ids_bbs == ids_dc:  # All three sets identical
        print("All three algorithms produced the same skyline.")  # Success message
    else:  # Mismatch — debugging info
        print("WARNING: Skyline results differ between algorithms!")  # Warning header
        print(f"  Sequential: {len(ids_seq)}, BBS: {len(ids_bbs)}, DC: {len(ids_dc)}")  # Counts
        print(f"  Only in sequential: {ids_seq - ids_bbs}")  # Points unique to sequential
        print(f"  Only in BBS: {ids_bbs - ids_seq}")  # Points unique to BBS


if __name__ == '__main__':
    main()  
