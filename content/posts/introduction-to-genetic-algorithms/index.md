+++
categories = ["genetic algorithms", "machine learning", "programming", "tutorial", "beginner"]
date = 2017-08-27T19:17:56Z
description = ""
draft = false
slug = "introduction-to-genetic-algorithms"
tags = ["genetic algorithms", "machine learning", "programming", "tutorial", "beginner"]
title = "Introduction to Genetic Algorithms"

+++


> “A breakthrough in machine learning would be worth ten Microsofts” 
>
> Bill Gates, Chairman, Microsoft



We live in the era of machine learning. Scientists and researchers from around the world are using it to improve our lives in ways we never imagined and advertisers to reach the most relevant people to sell their products. It's not a recent idea but it scaled so much lately due to great breakthroughs in computer hardware. There are many techniques in machine learning field and most of them are based in simple ideas. As an example, the topic we will discuss today is genetic algorithms and how we can create a basic demonstation program that takes advantage of it.

___

### What is a "Genetic Algorithm"

Genetic Algorithm or GA is similar to Natural Selection and Evolution in nature. It searches for a solution that fits the best in our problem. So, it will lead to at least a near-optimal solution in much shorter time than if a different approach was used.

### How it works

The concept of the algorithm is straightforward. The problem we want to solve is our system. Our actions, things we do act as an input to the system(analogously to chromosomes). The result of our actions come as the output. By observing which actions lead results closer to our goal and which don't, we know in which "direction" the answer will be found. Similarly to evolution, we keep our best fitting moves and we vary them slightly, trying to find an even better outcome.

### How it works in computer terms

The first and very important step is to find a way to represent a "chromosome". A chromosome consists of genes, which are actually inputs to our system. Inaccurate representation will lead to a poorly performing algorithm. The most common ways are:

##### Binary

Each gene is represented as a logical 0 or 1 (True or False). This encoding is not completely natural (meaning that you won't find it in nature) but find applications in our problems. One of them is the Knapsack Problem: Given a set of items, each with a weight and a value, determine the number of each item to include in a collection so that the total weight is less than or equal to a given limit and the total value is as large as possible - [Wikipedia](https://en.wikipedia.org/wiki/Knapsack_problem). With 0 we can illustrate the absence of the item in the knapshack and with 1 that it's included.

##### Real Value

Each gene is represented as a real value (continuous variable). This value can be anything from chars,integers or floating point numbers to more complicated entities. One problem that can be solved using this approach is estimating the weights of a neural network, where the weights for input of neurons must be found to train the network for wanted output (Most systems use "weights" to change the parameters of the throughput and the varying connections to the neurons. - [Wikipedia](https://en.wikipedia.org/wiki/Artificial_neural_network#Types))

##### Integer

This is used mostly to represent discrete variables encoded as integers. An example for this is a maze solver where our moves are up,down,left or right or as genes 0,1,2 and 3.

and finally,

##### Permutation Representation

Sometimes, the solution can be found by following the actions in specific order. The most typical example is the Travelling salesman problem (*"Given a list of cities and the distances between each pair of cities, what is the shortest possible route that visits each city exactly once and returns to the origin city?"* - [Wikipedia](https://en.wikipedia.org/wiki/Travelling_salesman_problem)) where a chromosome correspond to the order of cities, and that's why this method is used.

Now, I would like to do a small parenthesis and mention that I'll use as an example a small program I wrote in Go to demostrate the rest of the concepts used in GAs. Why in Go? Because I wanted to learn this language eventually and the best way to get started is to code something that actually does something (That's also my excuse for the, probably, wrong methodology used in my code, coming from Python 😀). The program is based on the Infinite monkey theorem which states that a monkey hitting keys at random on a typewriter keyboard for an infinite amount of time will almost surely type a given text, such as the complete works of William Shakespeare ([Wikipedia](https://en.wikipedia.org/wiki/Infinite_monkey_theorem)), and is a good GA problem to begin with. Of course, our "genetically modified monkeys" won't try to type the whole scripts of Shakespeare but a small phrase like "O Romeo, Romeo, wherefore art thou Romeo?"

So, let's begin. 

First of all we need an array that will serve as the population of our system. I gave it the size of 1000 and will contain the strings "typed from the monkeys". We use a function to initialize the population with random strings so we will have a state to begin with.

```go
func initializePopulation(initialPopulation []string) {
	var generatedName string
	for i := 0; i < len(initialPopulation); i++ {
     generatedName = ""
		for j := 0; j < len(nameToGuess); j++ {
        generatedName = s.Join([]string{generatedName,
				string(rand.Intn(90) + 32)}, "")
		}
    initialPopulation[i] = generatedName
	}
}
```

As you can see from the table below, with this 

```go
string(rand.Intn(90) + 32)
```

we pick a random ASCII character from 32 to 122

![ascii table](https://www.asciitable.com/asciifull.gif)

Next, we need a function to calculate how fit is a genome, or to express it in our problem domain, how close is a given string to our goal. Here, for every correct letter in the correct position the string gets a "point of fitness". The function is, again, simple to implement

```go
func calculateAndReturnFitness(guessString string) int {
	fitness := 0
	for i := 0; i < len(stringToGuess); i++ {
		if i > len(name) {
			fitness = 0
			break
		}
		if stringToGuess[i] == guessString[i] {
			fitness++
		}
	}
	return fitness
}
```

If by accident we get any string bigger than the string we try to guess, we immediately set its fitness to zero.

After calculating fitness for every string it's time to keep the fittest of them and create a new generation with the trait we want. To do that, we must decide which chromosomes will be in our "mating pool" and provide their genes. There are many methods to decide the "parents" of the new strings. The one we will use is called "Roulette Wheel Selection" and it's actually three simple steps

1. Calculate the sum of fitnesses, let's call it S

2. Generate a random number between 0 and S, let's call it R

3. Start for beginning of population and add the fitness of each chromosome to R. When R > S, stop and add that specific chromosome to the pool.

   Of course, you have to repeat this process to add more chromosomes to your pool. It's named like this because it looks like a spinning roulette wheel, only that in this case the ball stays still and the wheel is spinning around it, giving more chances to fittest strings to be in the pool, because of their bigger fitness value.

```go
func generateMatingPool(fitnessSum int) {
   	var populationIndex int = 0
   	var randomInitialPosition int = 0
   	for i := 0; i < len(population); i++ {
   		randomInitialPosition = rand.Intn(fitnessSum)
   		populationIndex = 0
   		for j := randomInitialPosition; j < fitnessSum; j += fitnessScore[populationIndex%len(population)] {
   			populationIndex += 1
   		}
   		matingPool[i] = population[populationIndex%len(population)]
   	}
   }
   ```

The next step is to actually pick two parents from the pool we just created. Again, there are many methods but we will just simply put our faith in luck and choose two random parents from the pool, given the fact that the strings with higher fitness will appear more frequently there and that we should never disqualify lower fitness strings because the provide the variation we need to help our strings evolve further. 

```go
func giveBirth() {
	var firstOffspring string
	var secondOffspring string
	var firstRandomParent int
	var secondRandomParent int
	for i := 0; i < len(matingPool); i += 2 {
		firstRandomParent = rand.Intn(len(matingPool))
		secondRandomParent = rand.Intn(len(matingPool))
		firstOffspring, secondOffspring = createOffspring(
			matingPool[firstRandomParent], matingPool[secondRandomParent])
		population[i] = firstOffspring
		population[i+1] = secondOffspring
	}
}
```

And here we come to the most interesting part. You see that little function we call? I'm talking about  

```go
createOffspring()
```

This is how we combine our two parents together and create children from them having similar features but the necessary variation we need to reach our target eventually. There are, again, many methods to use. Here, I'll present you two of these which are both included in the source (links at the end). The first method is the "Uniform crossover". Here, imagine we have a coin and for every character in the string we flip it. If the outcome is tails we take the string from the first parent, otherwise from the second. But we also have a second child which has the exactly opposite decisions from the previous one (So for tails we use the char from the second parent). This gives a 50/50 percent chance to every parent to contribute to the resulting children. We can also be biased towards the parent with the higher fitness and choose more genes from there, but in this case we don't. The second technique is called "Single-point crossover". We take a random point in the arrays and we split them, combining the left part of the first parent with the right part of the second parent and correspondingly the other two parts. So we end up again with two offsprings. 

```go
func createOffspring(firstParent string, secondParent string) (string, string) {
	firstOffspring := ""
	secondOffspring := ""
	for i := 0; i < len(stringToGuess); i++ {
		if rand.Intn(2) == 1 {
			// 0.1% mutation chance with random character in random position
			if rand.Intn(1000) == 5 {
				firstOffspring = s.Join([]string{firstOffspring,
					string(rand.Intn(90) + 32)}, "")
			} else {
				firstOffspring = s.Join([]string{firstOffspring,
					string(secondParent[i])}, "")
			}
			if rand.Intn(1000) == 6 {
				secondOffspring = s.Join([]string{secondOffspring,
					string(rand.Intn(90) + 32)}, "")
			} else {
				secondOffspring = s.Join([]string{secondOffspring,
					string(firstParent[i])}, "")
			}
		} else {
			if rand.Intn(1000) == 5 {
				firstOffspring = s.Join([]string{firstOffspring,
					string(rand.Intn(90) + 32)}, "")
			} else {
				firstOffspring = s.Join([]string{firstOffspring,
					string(firstParent[i])}, "")
			}
			if rand.Intn(1000) == 6 {
				secondOffspring = s.Join([]string{secondOffspring,
					string(rand.Intn(90) + 32)}, "")
			} else {
				secondOffspring = s.Join([]string{secondOffspring,
					string(secondParent[i])}, "")
			}
		}
	}
	return firstOffspring, secondOffspring
}
```

You will immediately notice these lines in the source code

```go
if rand.Intn(1000) == 5 {
	firstOffspring = s.Join([]string{firstOffspring,
	string(rand.Intn(90) + 32)}, "")
}
```

This is what we call a mutation. We expect every 1000 genes a gene will turn into something completely random, maintaining our genetic diversity and helping our GA to come to a better solution. Again, to mutate a gene there are many techniques, varying from randomly changing the value (like in our example) to shuffle two values in the chromosome. The importance of mutation can be seen if we disable it completely. After some point, the population of our program will end up being the same and there will be no way for new values to be produced. Having a high percentage of mutations is also not helpful because our program will not be able to follow the "direction" of high fitness. 

This is actually the main idea of the genetic algorithms you probably hear so often. As you can see, it's not so complicated as it sounds and it has many applications in bioinformatics, in any kind network design, in timetabling problems etc. The example above demonstrates a very basic example but experimenting with it you can make it fit in more complex problems and situations. 

The full source of the examples can be found on my [GitHub](https://github.com/steelgr/GuessTheString).

