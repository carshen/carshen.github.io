---
layout: post
title: "Implementation of the Aho-Corasick Algorithm in Python"
date: 2014-04-07 00:11:54
categories: jekyll update
---

In a previous article, we covered the intuition behind the Aho-Corasick string matching algorithm. Now, I will explain its implementation in Python. There are a variety of ways to do this. I will explain the way which uses an adjacency list to store the trie. Arguably, it is much cleaner to use classes and objects, but I decided not to do that because that's way overdone, and I'm a hipster (if you want that version, you can email me for it – it's quite easy to change it to use classes).

Recall from last time that we needed to construct the trie, and then set its failure transitions. After the trie is constructed, we traverse the trie as we are reading in the input text and output the positions that we find the keywords at. Essentially, these three parts form the structure of the algorithm.

The trie is represented as an adjacency list. One row of the adjacency list represents one node, and the index of the row is the unique id of that node. The row contains a dict `{'value':'', 'next_states':[],'fail_state':0,'output':[]}` where value is the character the node represents ('a','b', '#', '$', etc), `next_states` is a list of the id's of the child nodes, `fail_state` is the id of the fail state, and output is a list of all the complete keywords we have encountered so far as we have gone through the input text (in this implementation, we can add the same word multiple times in the trie).

We initialize the trie, called `AdjList`, and add the root node. We have the keywords, which we will add one by one into the trie.

{% highlight python linenos %}
from collections import deque
AdjList = []

def init_trie(keywords):
	""" creates a trie of keywords, then sets fail transitions """
	create_empty_trie()
	add_keywords(keywords)
	set_fail_transitions()

def create_empty_trie():
	""" initalize the root of the trie """
	AdjList.append({'value':'', 'next_states':[],'fail_state':0,'output':[]})

def add_keywords(keywords):
	""" add all keywords in list of keywords """
	for keyword in keywords:
		add_keyword(keyword)
{% endhighlight %}

We also write a helper find_next_state which takes a node and a value, and returns the id of the child of that node whose value matches value, or else None if none found.

{% highlight python linenos %}
def find_next_state(current_state, value):
	for node in AdjList[current_state]["next_states"]:
		if AdjList[node]["value"] == value:
			return node
	return None
{% endhighlight %}

Note that this trie only handles lowercase words, for simplicity for my testing.
To add a keyword into the trie, we traverse the longest prefix of the keyword that exists in the trie starting from the root, then we add the characters of the rest of the keyword as nodes in the trie, in a chain.

{% highlight python linenos %}
def add_keyword(keyword):
	""" add a keyword to the trie and mark output at the last node """
	current_state = 0
	j = 0
	keyword = keyword.lower()
	child = find_next_state(current_state, keyword[j])
	while child != None:
		current_state = child
		j = j + 1
		if j < len(keyword):
			child = find_next_state(current_state, keyword[j])
		else:
			break
	for i in range(j, len(keyword)):
		node = {'value':keyword[i],'next_states':[],'fail_state':0,'output':[]}
		AdjList.append(node)
		AdjList[current_state]["next_states"].append(len(AdjList) - 1)
		current_state = len(AdjList) - 1
	AdjList[current_state]["output"].append(keyword)
{% endhighlight %}

The while loop finds the longest prefix of the keyword which exists in the trie so far, and will exit when we can no longer match more characters at index j. The for loop goes through the rest of the keyword, creating a new node for each character and appending it to `AdjList`. `len(AdjList) – 1` gives the id of the node we are appending, since we are adding to the end of `AdjList`.

When we have completed adding the keyword in the trie, `AdjList[current_state]["output"].append(keyword)` will append the keyword to the output of the last node, to mark the end of the keyword at that node.

Now, to set the fail transitions. We will do a breadth first search over the trie and set the failure state of each node. First, we set all the children of the root to have the failure state of the root, since the longest strict suffix of a character would be the empty string, represented by the root. The failure state of the root doesn't matter, since when we get to the root, we will just leave the loop, but we can just set it to be the root itself.

Remember that the failure state indicates the end of the next longest proper suffix of the string that we have currently matched.

Consider the node `r`. We are setting the failure state for node `child` of `r`. Initially the potential parent of the fail state of `child`, `state` will be the next longest proper suffix, which is marked by `r`'s fail state. If there is no transition from `r`'s fail state to a node with the same value as `child`, then we go to the next longest proper suffix, which is the fail state of `r`'s fail state, and so on, until we find one which works, or we are at the root.
We set `child`'s fail state to be this fail state.

We append the output of the fail state to `child`'s output because since the fail state is a suffix of the string which ends at `child`, whatever matched words found at the fail state will also occur at `child`. If we did not keep this line, we would miss out on substrings of the currently matched string which are keywords.

{% highlight python linenos %}
def set_fail_transitions():
	queue = deque()
	child = 0
	for node in AdjList[0]["next_states"]:
		queue.append(node)
		AdjList[node]["fail_state"] = 0
	while len(queue) != 0:
		r = queue.popleft()
		for child in AdjList[r]["next_states"]:
			queue.append(child)
			state = AdjList[r]["fail_state"]
			while find_next_state(state, AdjList[child]["value"]) == None \
 and state != 0:
				state = AdjList[state]["fail_state"]
			AdjList[child]["fail_state"] = find_next_state(state, 
AdjList[child]["value"])
			if AdjList[child]["fail_state"] is None:
				AdjList[child]["fail_state"] = 0
			AdjList[child]["output"] = AdjList[child]["output"] + 
AdjList[AdjList[child]["fail_state"]]["output"]

{% endhighlight %}

Finally, our trie is constructed. Given an input, line, we iterate through each character in line, going up to the fail state when we no longer match the next character in line. At each node, we check to see if there is any output, and we will capture all the outputted words and their respective indices. `(i-len(j) + 1` is for writing an index at the beginning of the word)

{% highlight python linenos %}
def get_keywords_found(line):
	""" returns true if line contains any keywords in trie """
	line = line.lower()
	current_state = 0
	keywords_found = []

	for i in range(0, len(line)):
		while find_next_state(current_state, line[i]) is None and current_state != 0:
			current_state = AdjList[current_state]["fail_state"]
		current_state = find_next_state(current_state, line[i])
		if current_state is None:
			current_state = 0
		else:
			for j in AdjList[current_state]["output"]:
				keywords_found.append({"index":i-len(j) + 1,"word":j})
	return keywords_found
{% endhighlight %}

Yay! We are done!

Test it like so:

{% highlight python linenos %}
init_trie(['cash', 'shew', 'ew'])
print get_keywords_found("cashew")
{% endhighlight %}

As always, leave questions and concerns in the comments below. See you next time!



