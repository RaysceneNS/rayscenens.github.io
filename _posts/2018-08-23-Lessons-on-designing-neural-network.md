---
title: "Notebook: What I learned designing a neural network that plays an arcade game"
tags: [Machine Learning]
---

My goal for this project was to become more familiar with how neural networks operate, and to understand the sorts of problems that they can be applied to for future projects. That is why for this project I opted not to use the a prebuilt machine learning library framework, but instead opted for an approach where I build up the math types using raw c#. Luckily I had a project left over from many years ago in which I developed a Matrix type and coded many of the math operators.  I felt that going down this road gave me a deeper insight into how a neural network works at the bare metal.

## Input layer

This network layer is comprised of a matrix with a size of [9,9] nodes. The nodes 1-8 are wired to represent signals from the 8 polar directions of the ship looking out into space. The signal level of the node ranging from 0.0 -> 1.0 dependent on how close an asteroid is in that particular direction. To determine if a threat signal is present a series of points along the ray projected from the ship in the direction of sight are tested for intersections with the asteroids. The signals are calculated as the inverse of the distance and are limited by how far the ship can see in any direction (else the sight would wrap around the board).

[vision test](/assets/images/2018/08/24/visiontest.png)
This shows how a ray is traced from the ship coordinate through the asteroid field to produce a measure of how close an asteroid is to the ship in a given direction.

## Output layer

The output layer nodes are wired to invoke the controls of the ship. Forward, Left, Right and Fire are mapped to the various nodes in the output layer. To convert the signals to an action we test to see if the signal strength is greater than 0.8, if it is then we invoke the particular action of the ship that the output node is tasked with.

## Hidden layer

The hidden layer has a size of [16,16] nodes. This is the layer that we train through every generation of the model. The training attempts to select for traits that produce desirable outcomes. To quantify this there is a fitness mechanism that gives scores to the players for their performance. In our network we test for traits such as activeness, high score or possibly lifespan. Those players that perform higher are randomly chosen from the population, we then combine the neural networks of these players together to form a new player that will be placed into the next generation. In this way we mimic a genetic system.

## Next steps

When running the program there are a large number of dud players, these are networks whose response to the stimuli never evoke an output greater than our threshold. It would be nice to cull these from the population before they get a chance to move to the next generation.

The vision mechanism is poor reflection of the game state. The problem with the vision layer is apparent when you consider the mechanics of the game. As it stands now the signal received into this layer for an asteroid that is moving towards the ship is identical to the signal for an asteroid that is travelling past or away from the ship. What is needed is some mechanism to encode the motion vector relative to the ship so that the player AI can determine the threat level and respond accordingly.

The project files are available on [GitHub](https://github.com/RaysceneNS/Stroids/tree/master/Stroids)
