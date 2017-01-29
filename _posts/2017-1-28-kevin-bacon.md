---
layout: post
title: Six Degrees of Kevin Bacon
---

## The Problem

Six Degrees of Kevin Bacon is a game where you have to link a given actor with Kevin Bacon in 6 movies or less. If you think of each actor as a 'node', and the edges connecting nodes as a movie two actors were both in, then the game turns into a graph problem. In this post, I will walk through (using Ruby) how to:

1. Create a tree of actor nodes, connected by movies the actors were both in, represented by an adjacency matrix
2. Use breadth first search to find the shortest path between the desired actor and Kevin Bacon.

The path will be limited to how much data I was willing to take the time to insert, but the concepts are the same nonetheless.

Before I move on, I'd like to just make a quick comment on overcoming obstacles. Whenever I read a blog post or explanation of a programming concept, it can be easy to assume the author breezed through everything. This can be demoralizing - until you come to the realization that the author probably banged his/her head against the wall for hours or days before finally figuring it out. This was certainly the case with me and the following solution. If you showed me the final code base that I ultimately wrote before I started, I would've curled up in a ball of self loathing and lamented that I could never write that. The bottom line is that real learning comes from looking at something that we cannot do, and having the grit and determination to plug away until we can. I've only been coding for about a year now, and one of the most important realizations I've come to is that you need to just start. Break things. Be curious. And in the end you will reach the finish line.

Alright, enough rah rah speech from me, let's find the Bacon.

## The graph

Developing a graph that contains all necessary information is the most crucial part of this exercise. Once you have this, the breadth first search algorithm is fairly straight forward to implement. I opted to use an adjacency matrix, since this allowed for very clear associations between actors. One downside of an adjacency matrix is that they get very big very quickly (n^2 associations will need to be stored for n actors). For this purpose, it worked just fine though.

First, we need to create a Node class to represent the actors. Each node will need the following pieces of information:

1. Name
2. Index
3. Boolean variable to determine if the node has been visited yet during the breadth first search
4. A back pointer that links the node to its parent node.
5. An array of movies they have acted in

Following is the Node class:

```ruby
class Node
  attr_accessor :index, :name, :visited, :back_pointer, :movies
  
  def initialize(name)
    @name = name
    @visited = false
    @index = nil
    @movies = []
  end
end
```

Alright, we've got our node class. Now how are we going to go about creating the adjacency matrix? First, let's create a class `AdjMatrix`, which is initialized with the root actor represented by a string. Who is the root? The center of the universe, Kevin Bacon.

```ruby
class AdjMatrix
  attr_accessor :actors_array, :matrix, :film_hash
  
  def initialize(name)
    root = Node.new(name)
    root.index = 0
    @actors_array = [root]
    @matrix = [[0]]
    @film_hash = {}
  end
end
```

The adjacency matrix object is initialized with the following instance variables:

1. `@actors_array` : This will store an array of actors. Their index location here will be how we determine their matrix lookup.
2. `@matrix` : This will be the adjacency matrix. The matrix will simply be an n x n matrix (n being the number of unique actors we'e inserted), with either a 1 or a 0 in each location. A 1 represents that the two actors have appeared in a movie together, while a 0 represents that they have not.
3. `@film_hash` : A hash that will store each movie as the key, and then an array of actors in that movie as the value

Before I move on with the rest of the adjacency matrix, following is how I will store the movie data:

```ruby
footloose = ["Kevin Bacon", "Lori Singer", "John Lithgow", "Dianne West", "Chris Penn", "Sarah Jessica Parker", 
              "John Laughlin", "Elizabeth Gorcey", "Frances Lee McCain", "Jim Youngs", "Douglas Dirkson", "Lynne Marta", 
              "Arthur Rosenburg", "Timothy Scott", "Alan Haufrect"]

interstellar = ["Ellen Burstyn", "Matthew McConaughey", "Mackenzie Foy", "John Lithgow", "Timothee Chalamet", "David Oyelowo",
                "Collette Wolfe", "Francis McCarthy", "Bill Irwin", "Anne Hathaway", "Andrew Borba", "Wes Bentley", "William Devane",
                "Michael Caine", "David Gyasi"]

wolf_of_wallstreet = ["Leonardo DiCaprio", "Jonah Hill", "Margot Robbie", "Matthew McConaughey", "Kyle Chandler", "Rob Reiner",
                      "Jon Bernthal", "Jon Favreau", "Jean Dujardin", "Joanna Lumley", "Cristin Milloti", "Christine Ebersole", 
                      "Shea Whigham", "Katarina Cas", "P.J. Byrne"]
```

The next step is to insert this data into the adjacency matrix class. In order to do this, I will create a method `#add_film`, as follows:

```ruby
def add_film(name, actors)
  actors_node_array = actors.map { |actor| Node.new(actor) }
  @film_hash[name] = actors
  current_actors = @actors_array.length
  actors_to_add = 0
  
  actors_node_array.each do |actor_node|
    if (find_index(actor_node.name) == nil)
      actors_to_add += 1
      actor_node.index = @actors_array.length
      @actors_array << actor_node
      actor_node.movies << name
      @matrix += [[0] * current_actors] 
    else
      actor_index = find_index(actor_node.name)
      actors_array[actor_index].movies << name
    end
  end
  @matrix.each_with_index do |array, index|
    @matrix[index] = array + [0] * (actors_to_add)
  end

  actors_node_array.each do |actor_to_insert|
    @film_hash.each do |film, film_array|
      # If the actor is in a film, give association with every actor in that film
      if (film_array.index(actor_to_insert.name) != nil)
        film_array.each do |actor|
          if (actor_to_insert.name == actor)
            next
          end
          actor_to_insert_index = find_index(actor_to_insert.name)
          actor_index = find_index(actor)
          matrix[actor_to_insert_index][actor_index] = 1
          matrix[actor_index][actor_to_insert_index] = 1
        end
      end
    end
  end
end
```

There is a whooooole lot going on here, so let's break it down piece by piece.

Since the data is stored as an array of names represented by strings, we will need to first create nodes for each actor. This is done with the following line of code:

`actors_node_array = actors.map { |actor| Node.new(actor) }`

We then add the array of actors into @film_hash:

`@film_hash[name] = actors`

The next step is a little tricky. We need to expand the matrix to accommodate any actor that has not yet been inserted into the matrix. Forget about assigning 1's and 0's for a minute - the task right now is to expand the matrix by the number of unique actors in the film we inserted, and assign each of them an appropriate index for lookup. This is accomplished with the following section of code:

```ruby
current_actors = @actors_array.length
actors_to_add = 0

actors_node_array.each do |actor_node|
  if (find_index(actor_node.name) == nil)
    actors_to_add += 1
    actor_node.index = @actors_array.length
    @actors_array << actor_node
    actor_node.movies << name
    @matrix += [[0] * current_actors] 
  else
    actor_index = find_index(actor_node.name)
    actors_array[actor_index].movies << name
  end
end
@matrix.each_with_index do |array, index|
  @matrix[index] = array + [0] * (actors_to_add)
end
```
A local variable `current_actors` is set to the length of the `@actors` array. Basically, it's how many actors we currently have stored in the matrix. The next bit of code loops through each of the actors in the movie being inserted. The helper method `#find_index` determines first if the actor is already in the matrix. If they aren't, then we assign a new index, add them to the `@actors_array`, and then create a new row in the matrix with `current_actors` number of 0's. We also add the movie to the node's `@movies` array. If the actor is already in the matrix, then we just add the movie to their `@movies` array.

Following is the code for the `#find_index` helper method I alluded to:

```ruby
def find_index(actor_name)
  current_index = 0
  actor_found = false
  while (current_index < @actors_array.length)
    if (actor_name == @actors_array[current_index].name)
      actor_found = true
      break
    end
    current_index += 1
  end
  
  return actor_found ? current_index : nil
  
end
```

There's one problem - we haven't added columns for our new actors! Luckily, we kept track of the number of actors to add as we looped through the array in a local variable `actors_to_add`. For each row, we add this many columns of '0', and our matrix has been resized to accommodate all of the new, unique actors.

```ruby
@matrix.each_with_index do |array, index|
  @matrix[index] = array + [0] * (actors_to_add)
end
```

Our last task is to assign 1's at matrix locations where two actors have appeared in a movie together. This is accomplished as follows:

```ruby
actors_node_array.each do |actor_to_insert|
  @film_hash.each do |film, film_array|
    # If the actor is in a film, give association with every actor in that film
    if (film_array.index(actor_to_insert.name) != nil)
      film_array.each do |actor|
        if (actor_to_insert.name == actor)
          next
        end
        actor_to_insert_index = find_index(actor_to_insert.name)
        actor_index = find_index(actor)
        matrix[actor_to_insert_index][actor_index] = 1
        matrix[actor_index][actor_to_insert_index] = 1
      end
    end
  end
end
```

Here, we loop through each actor in the movie we're adding. For each actor, we loop through each movie in the `@film_hash`, and determine if they were in that movie. If they were, they get a '1' for each actor in that movie. We don't want a '1' association for an actor with him/herself, so we skip such iterations.

PHEW! We've now got our matrix. At this point, my brain was a little scrambled, so I wrote a method to print a nice clean version of the matrix to help me visualize what was going on. Here is the resulting `#print_matrix` method:

```ruby
def print_matrix
  @matrix.each do |row|
    index = @matrix.index(row)
    if index < 10
      print "#{index}:  #{@actors_array[index].name}" + " " * (25 - @actors_array[index].name.length)
    else
      print "#{index}: #{@actors_array[index].name}" + " " * (25 - @actors_array[index].name.length)
    end
    puts (row.collect{|i| i.to_s}).join('  ')    
  end
end
```

The 25 is just a length longer than the length of the longest name I inserted. This function will break down as we add more movies, but for just a couple movies it works just fine to visualize what is happening. Here is the output once we add Footloose and Interstellar:

```
0:  Kevin Bacon              0  1  1  1  1  1  1  1  1  1  1  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
1:  Lori Singer              1  0  1  1  1  1  1  1  1  1  1  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
2:  John Lithgow             1  1  0  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1  1
3:  Dianne West              1  1  1  0  1  1  1  1  1  1  1  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
4:  Chris Penn               1  1  1  1  0  1  1  1  1  1  1  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
5:  Sarah Jessica Parker     1  1  1  1  1  0  1  1  1  1  1  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
6:  John Laughlin            1  1  1  1  1  1  0  1  1  1  1  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
7:  Elizabeth Gorcey         1  1  1  1  1  1  1  0  1  1  1  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
8:  Frances Lee McCain       1  1  1  1  1  1  1  1  0  1  1  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
9:  Jim Youngs               1  1  1  1  1  1  1  1  1  0  1  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
10: Douglas Dirkson          1  1  1  1  1  1  1  1  1  1  0  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
11: Lynne Marta              1  1  1  1  1  1  1  1  1  1  1  0  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
12: Arthur Rosenburg         1  1  1  1  1  1  1  1  1  1  1  1  0  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
13: Timothy Scott            1  1  1  1  1  1  1  1  1  1  1  1  1  0  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0
14: Alan Haufrect            1  1  1  1  1  1  1  1  1  1  1  1  1  1  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0
15: Ellen Burstyn            0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  1  1  1  1  1  1  1  1  1
16: Matthew McConaughey      0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  0  1  1  1  1  1  1  1  1  1  1  1  1
17: Mackenzie Foy            0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  0  1  1  1  1  1  1  1  1  1  1  1
18: Timothee Chalamet        0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  0  1  1  1  1  1  1  1  1  1  1
19: David Oyelowo            0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  0  1  1  1  1  1  1  1  1  1
20: Collette Wolfe           0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  1  0  1  1  1  1  1  1  1  1
21: Francis McCarthy         0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  1  1  0  1  1  1  1  1  1  1
22: Bill Irwin               0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  1  1  1  0  1  1  1  1  1  1
23: Anne Hathaway            0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  1  1  1  1  0  1  1  1  1  1
24: Andrew Borba             0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  1  1  1  1  1  0  1  1  1  1
25: Wes Bentley              0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  1  1  1  1  1  1  0  1  1  1
26: William Devane           0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  1  1  1  1  1  1  1  0  1  1
27: Michael Caine            0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  1  1  1  1  1  1  1  1  0  1
28: David Gyasi              0  0  1  0  0  0  0  0  0  0  0  0  0  0  0  1  1  1  1  1  1  1  1  1  1  1  1  1  0
```

Don't know about you, but my OCD approves of this matrix. We can see that our 1 link between Footloose and Interstellar is John Lithgow - he is the key link to the Bacon.

## The Search

Now for the search. Just for fun, I opted to return a string that describes the path to Kevin Bacon. For instance, after entering the 3 movies shown above, the path to Kevin Bacon from Leonardo DiCaprio is Wolf of Wallstreet with Matthew McConaughey, who was in Interstellar with John Lithgow, who was in Footloose with Kevin Bacon. The output of the `#find_kevin_bacon` function is thus as follows:

```
Leonardo DiCaprio acted in Wolf of Wallstreet with Matthew McConaughey
Matthew McConaughey acted in Interstellar with John Lithgow
John Lithgow acted in Footloose with Kevin Bacon
```

The implementation of the `#find_kevin_bacon` function follows:

```ruby
def find_kevin_bacon(actor_name)
  node_queue = [0]
  
  loop do
    curr_node = node_queue.pop
    
    return false if curr_node == nil
    if curr_node == find_index(actor_name)
      node = @actors_array[curr_node]
      actors = [node.name]
      while (node.back_pointer != 0)
        actors << @actors_array[node.back_pointer].name
        node = @actors_array[node.back_pointer]
      end
      
      actors << @actors_array[0].name
      return_string = ""
      index = 0
      
      while (index < actors.length - 1)
      actor1 = actors[index]
      actor2 = actors[index + 1]
        common_movie = find_common_movie(actor1, actor2)
        string = "#{actor1} acted in #{common_movie} with #{actor2}\n"
        return_string += string
        index += 1
      end
      return return_string
    end
    matrix_size = @matrix.length
    children = (0..matrix_size-1).to_a.select do |i|
      @matrix[curr_node][i] == 1
    end
    
    children.each do |child|
      unless @actors_array[child].visited == true
        @actors_array[child].back_pointer = curr_node
        @actors_array[child].visited = true
      end
    end
    
    node_queue = children + node_queue
    
  end
end
```
So what's going on here? First, we create a queue with the root node as the only element:

`node_queue = [0]`

We then start a loop that will search for our target node until it is found. We start by grabbing the last element in the queue:

`curr_node = node_queue.pop`

Since trying to figure out how to get from Kevin Bacon to himself isn't particularly interesting, let's jump down to the case where this node does not equal our target node. Here, we find all 'children' nodes of Kevin Bacon - i.e. all actors who have acted in a movie directly with Kevin Bacon. All of these nodes are then added to the queue.

```ruby
matrix_size = @matrix.length
children = (0..matrix_size-1).to_a.select do |i|
  @matrix[curr_node][i] == 1
end
``` 

While this will ensure that we can ultimately find a path to our target node (if one exists), it will not allow us to trace what the path IS once the target node is ultimately found. That's where the following bit of code comes in:

```ruby
children.each do |child|
  unless @actors_array[child].visited == true
    @actors_array[child].back_pointer = curr_node
    @actors_array[child].visited = true
  end
end

node_queue = children + node_queue
```

If the node has not yet been visited, then when know that the current node in the recrusion is the child node's parent. We thus set the `@back_pointer` attribute of the node to the current node, and set the `@visiited` boolean value to `true` before ultimately adding all of the children to the queue.

Now, once the target actor is located, the hard part is over. We have all of the information that we need, and just need to do some fun loops and string interpolation. Here's the code that will run once the target actor is found:

```ruby
if curr_node == find_index(actor_name)
  node = @actors_array[curr_node]
  actors = [node.name]
  while (node.back_pointer != 0)
    actors << @actors_array[node.back_pointer].name
    node = @actors_array[node.back_pointer]
  end
  
  actors << @actors_array[0].name
  return_string = ""
  index = 0
  
  while (index < actors.length - 1)
  actor1 = actors[index]
  actor2 = actors[index + 1]
    common_movie = find_common_movie(actor1, actor2)
    string = "#{actor1} acted in #{common_movie} with #{actor2}\n"
    return_string += string
    index += 1
  end
  return return_string
end
```

First, we create a local variable `actors` that will house an array of actors leading from the target actor to Kevin Bacon. Then, we create while loop that goes through all of the `back_pointer` attributes and adds them to the `actors` array, until we get to Kevin Bacon. Once we have that, we'll loop through each item in the array and use the `#find_common_movie` helper method I wrote to find a common movie between each array item and the item that follows it. Here is the `#find_common_movie` helper method:

```ruby
def find_common_movie(actor1, actor2)
  actor1_index = find_index(actor1)
  actor2_index = find_index(actor2)
  
  actor1_node = @actors_array[actor1_index]
  actor2_node = @actors_array[actor2_index]
  
  actor1_node.movies.each do |movie|
    if actor2_node.movies.index(movie) != nil
      return movie
    end
  end
end
```

Now the finish line is in sight! We create an empty string, and then loop through each actor to add to the string the actor's name, what movie they have in common with the following actor, and then a new line.

From here, all that's left to do is create an adjacency matrix, add the movies, and then find Kevin Bacon!

```ruby
matrix = AdjMatrix.new("Kevin Bacon")
matrix.add_film("Footloose", footloose)
matrix.add_film("Interstellar", interstellar)
matrix.add_film("Wolf of Wallstreet", wolf_of_wallstreet)

puts "We're gonna find Leonardo DiCaprio's Bacon connection..."
puts "#{matrix.find_kevin_bacon("Leonardo DiCaprio")}"
```

And the output is:

```
We're gonna find Leonardo DiCaprio's Bacon connection...
Leonardo DiCaprio acted in Wolf of Wallstreet with Matthew McConaughey
Matthew McConaughey acted in Interstellar with John Lithgow
John Lithgow acted in Footloose with Kevin Bacon
```

And we're done! Any comments, questions, or feedback are greatly appreciated.