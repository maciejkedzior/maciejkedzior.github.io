---
layout: post
title:  "Advent of Code 2021 Day 6"
date:   2022-01-03 22:32:00 +0100
categories: update
---


# Advent of Code 2021 Day 6

> A massive school of glowing lanternfish swims past. They must spawn quickly to reach such large numbers - maybe exponentially quickly? 
> You should model their growth rate to be sure.



## Rules of spawning:
* each lanternfish creates a new lanternfish once every 7 days
* a lanternfish that creates a new fish resets its timer to 6, not 7 (because 0 is included as a valid timer value)
* the new lanternfish starts with an internal timer of 8 and does not start counting down until the next day


## First thoughts
Puzzle looks rather innocent. As the site suggest in an example where we have Lanternfishes have these counters:


                                    [3,4,3,2,1]

after using rules mentioned before we will obtain such set of Lanternfishes after 18 days:

                [6,0,6,4,5,6,0,1,1,2,6,0,1,1,1,2,2,3,3,4,6,7,8,8,8,8]

We can see that after just 18 days population of Lanternfishes expanded to 26 fishes from 5. Quick math and we see that we got demographic explosion: 520% !

However it brings new problem - after 80 days there are 5934 fishes, so there is expansion of 118680 %. There is no way that with proper input we will be able to quickly obtain number of fishes after 80 days. Nothing to say about memory usage. Even if we wanted to model puzzle as a simple array of `char`s of size 8 bit it is clear that array size will be enormous. There has to be other way. And there is.


## Smart idea - counter object
Let's think about our starting conditions other way. If we have input of [3,4,3,2,1] we can count how many fishes are at certain level.

                        Day 0       count: [0 1 1 2 1 0 0 0 0]
                                    level: [0 1 2 3 4 5 6 7 8]

If we would try to model step from the day zero to day one we will obtain these results:

                        Day 1       count: [1 1 2 1 0 0 0 0 0]
                                    level: [0 1 2 3 4 5 6 7 8]

Then few more days:

                        Day 2       count: [0 2 1 0 0 0 1 0 1]
                                    level: [0 1 2 3 4 5 6 7 8]

                        Day 3       count: [2 1 0 0 0 1 0 1 0]
                                    level: [0 1 2 3 4 5 6 7 8]

                        Day 4       count: [1 0 0 0 1 0 3 0 1]
                                    level: [0 1 2 3 4 5 6 7 8]
                        .........................................
                        Day 17      count: [4 3 5 3 2 2 1 1 1]
                                    level: [0 1 2 3 4 5 6 7 8]

                        Day 18      count: [3 5 3 2 2 1 5 1 4]
                                    level: [0 1 2 3 4 5 6 7 8]

Here we see something interesting. Basically day after day we "move" all counts one to left number of Lanternfishes at level zero add to level six and set level eight to that value.

With that approach we don't have to care about memory usage: our container only needs nine pairs!
However new question arises: what kind of data structure should we use so the "moving" operation will be quick?

## Choosing data structure

First thing that came to my mind was that we don't actually have to store pairs, because we already know what are keys: numbers in range [0,8]. Thanks to that fact we can simply use some **Array** container with indexing and we will store values at first nine indexes.

If we would try to create algorithm for simple C-like array we see that "moving" is problematic - we have to store zeroth value in variable and then copy in for-loop values from i-th index to i-1-th index. Each day we have to do it And it is rather tedious. There is a better way: **Linked List**.


## Some Linked list theory and implementation in C

We remember from Data Structure course that **Linked list** is a data container that stores nodes which contain value and pointer to next node.

                    Head -> [4, ->] [3, ->] NULL
Inserting new value at the beginning of list is done in constant time, at the k-th index linear. Same with deleting nodes. 

So, if we want to create some easy implementation of that data structure in C we would write code like this:

{% highlight c%}

struct node_t {
    int64_t count;
    struct node_t *next;
};
struct node_t* linked_list_create_node(int64_t val){
    struct node_t *head = malloc(sizeof(struct node_t *));
    head->count = val;
    head->next = NULL;

    return head;
}

struct node_t* linked_list_head_init(int64_t val){
    return linked_list_create_node(val);
}

void linked_list_push_back(struct node_t *head, int64_t val){
    struct node_t *iter = head;
    struct node_t *added = linked_list_create_node(val);

    while (iter->next != NULL)
        iter = iter->next;
    iter->next = added;
}

struct node_t* linked_list_pop_front(struct node_t *head, int64_t *result){
    *result = head->count;
    struct node_t *temp = head;
    head = head->next;
    free(temp);
    return head;
}

void linked_list_destroy(struct node_t *head){
    int64_t f;
    while (head != NULL)
        linked_list_pop_front(head, &f);
}

void linked_list_increment(struct node_t *head, int64_t pos, int64_t val){
    int64_t i=0;
    struct node_t *iter = head;
    while (i++ != pos)
        iter = iter->next;
    iter->count += val;
}

void linked_list_print(struct node_t *head){
    struct node_t *iter = head;
    while (iter != NULL){
        printf("%I64d -> ", iter->count);
        iter = iter->next;
    }
    printf("\n");
}
{% endhighlight %}

Pretty quick and simple implementation. Now, back to the topic.


## Solution

We see that each day we have to:
1. Store in variable value stored at zeroth index
2. Move all values to left
3. Add to sixth index number stored in variable
4. Assign to eighth index number stored in variable

So translating it to our approach with Linked list:
1. Initialize linked list with nine nodes, each with value 0, where i-th index represents count of i-th level Lanternfishes
2. Properly fill linked list with data from input
3. Each day:
   1. Call `linked_list_pop_front(head, &res)` where `res` is an `int64_t` variable
   2. Call `linked_list_increment(head, 6, res)`
   3. Call `linked_list_push_back(head, res)`
4. After all days pass call `linked_list_sum(head)` and we obtain our result!

## But what about Part 2?
Of course, as with any puzzle in Advent of Code there is part 2. However if we read through it, we see that with our approach it is a free star: "How many lanternfish would there be after 256 days?" We just increase number of loops and we still get our anwser with no problem. Only one major important thing: as you could see in my linked list implementation, I used variables of 64-bit size. Why is that? Because population increases in an exponentional way so the anwser for part 2 will overflow if we simply use int.

Thanks for reading my article and I hope I will see you soon :D
Maciej