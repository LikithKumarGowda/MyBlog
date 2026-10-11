+++
title = "Machine Learning Basics"
date = 2026-07-27
description = "What machine learning is, how it works, and how we know a model is actually learning."
template = "blog/page.html"

[taxonomies]
tags = ["machineLearning", "basics"]
+++

### What is Machine Learning?

Machine learning is the field of study that uses data and algorithms to build models that can make predictions or decisions without being explicitly programmed for each scenario.

In traditional programming, I write the rules and the computer applies them to data to produce answers. In machine learning, I give the computer the data and the answers, and it figures out the rules on its own.

```
Traditional programming:   Rules + Data    -->  Answers
Machine learning:          Data + Answers  -->  Rules (the model)
```

For example, if we try writing rules to detect spam (Eg: "if the email contains the word *lottery* then mark it as spam") it won't work for long, because spammers change their wording and approach. With machine learning, I provide thousands of emails already labelled spam or not spam, and it would learn the patterns itself.

### How does it work?

A machine learning system identifies patterns in historical data and uses those patterns to make reliable predictions on new inputs.

The basic loop looks like this:

1. **Collect data:** For ex, house size and the prices they were sold for.
2. **Train a model:** The algorithm adjusts the model's internal parameters so its predictions get closer to the known answers.
3. **Evaluate:** Test the model on data it has never seen.
4. **Predict:** Use the trained model on new inputs. This step is called *inference*.

A few terms that come up everywhere:

- **Features:** the inputs the model looks at (house size, number of rooms).
- **Label / target:** the answer we want the model to predict (price).
- **Model:** the function that maps features to a prediction.
- **Training:** the process of tuning the model using data.

### Types of machine learning

| Type | Data it learns from |
|---|---|
| Supervised learning | Inputs with known labels |
| Unsupervised learning | Inputs with no labels | 
| Reinforcement learning | Rewards and penalties from trial and error |

Supervised learning is where most people start, and it splits into two common problems:

- **Regression:** predict a number (price, temperature).
- **Classification:** predict a category (spam or not spam, cat or dog).

### How do we know we are on the right path?

We measure success by checking whether performance on a clearly defined task improves with experience. A common way to state this: a program learns from experience **E** with respect to a task **T** and a performance measure **P**, if its performance on T, as measured by P, improves with E.

Take training a machine to play chess:

a. **Task (T):** Playing chess games, for example choosing moves that lead to a win.

b. **Performance (P):** How well it does, measured for example by its win rate against human players or other programs.

c. **Experience (E):** The games it learns from, such as historical games, board positions and outcomes, or games it plays against itself.

If the win rate keeps going up as the machine sees more games, it is learning.

### Training vs. testing

A model can score perfectly on data it has already seen simply by memorizing it. That tells us nothing about how it will behave on new data. So we split the data:

- **Training set:** used to teach the model.
- **Test set:** kept hidden until the end, used only to measure real performance.

This leads to the two classic problems:

- **Underfitting:** the model is too simple and performs poorly even on the training data.
- **Overfitting:** the model memorizes the training data, including its noise, and performs poorly on new data.

The goal is a model that **generalizes**: it does well on data it has never seen.

### Summary

- Machine learning builds models that learn patterns from data instead of following hand-written rules.
- Training tunes the model on known examples. And we get the model prediction on new untrained data(Inference).
- Progress is measured with a defined task(T), a performance measure(P), and experience(E).
- We always evaluate the model on unseen data that hasn't been trained on, to catch overfitting.

Next up: **Linear Regression**, the simplest supervised learning algorithm, and a good first model to build.