---
excerpt: Information about the use of artificial neural networks for branch prediction, and a simple simulation in python. 
title: The Neural Network in Your CPU
---

Did you know that most modern processors manufactured today have an artificial neural network implemented in hardware within it? The first time I heard this I thought through all of the different features that modern CPU's implement, but couldn't put my finger on exactly which feature would benefit from an artificial neural network. Most aspects of the processor make exact calculations, so an artificial neural network doesn't seem obviously useful. For example, an artificial neural network wouldn't improve the instruction decode step of the pipeline. My thoughts turned to cache replacement policies. Maybe a neural network could help the processor decide what information to cache? 

The answer turned out to be the branch predictor, whose performance can be greatly improved through the use of a neural network implemented in hardware. An artificial neural network is able to quickly learn correlations between previous branches and the current branch instruction in order to make very accurate predictions.  

### What is Branch Prediction?

 Modern CPU's have long pipelines, [some as long as 20-24 stages](http://www.bit-tech.net/hardware/cpus/2008/11/03/intel-core-i7-nehalem-architecture-dive/5), which allow them to execute many instructions at different stages of completion simultaneously. That works perfectly well for sequential instructions, but what instructions should the pipeline be computing before a conditional instruction is fully processed? 
 
The processor actually attempts to predict whether the branch instruction will be taken or not. If the processor's prediction is correct, execution proceeds as normal. If the branch is incorrect, the processor has to perform a time consuming operation known as ["flushing the pipeline"](https://en.wikipedia.org/wiki/Pipeline_flush), where all of the partially executed instructions in the pipeline have to be removed. Thus, a better method of branch prediction directly impacts how quickly programs will run on your processor. As branch instructions make up [as many around 22% of all instructions](http://web.engr.oregonstate.edu/~benl/Projects/branch_pred/), it turns out that accurate branch prediction is one of the most critical drivers of CPU performance.

Because of their impact on performance, branch predictors are closely guarded trade secrets. Nevertheless, press releases provide [evidence](http://www.theregister.co.uk/2016/08/22/samsung_m1_core/) that some modern processors actually use neural network techniques.  

### How Does a Neural Network Branch Predictor Work?

A simple artificial neural network implementation for improving processor performance can be found in [this](https://www.cs.utexas.edu/~lin/papers/hpca01.pdf) 2001 paper. It uses a single layer perceptron network with integer weight values, which is easy to implement in hardware. A branch predictor in a state of the art processor is certainly far more complex than this this, but this is a good starting point for understanding how an artificial neural network can make accurate predictions. I'll show a simple implementation in a python simulation to explain how the algorithm works. This simple implementation is able to make branch predictions on a real dataset with a **98.47% accuracy rate!** This compares favorably with other simple branch prediction algorithms, like a saturating counter. 

The algorithm involves only a few different pieces of information. One is a queue which holds the past N branch instruction outcomes. Another is a table of perceptron weights. In an ideal world, each branch instruction at a given memory address would have its own set of weights.  

A high level explanation of how the algorithm works: 

1. A branch statement is encountered
2. The memory address of the branch statement is hashed to select a set of perceptron weights of a table of perceptrons  
3. The global branch history is multiplied by the perceptron's weights to make a prediction
4. When the true value of the branch statement is known, the perceptron's weights are updated for future predictions and the outcome value is added to the global branch history queue. 

Note that the artificial neural network is not trained in advance -- it is entirely trained on the fly. Updating the perceptron to "learn" the correlations between the global branch history and the branch being predicted is the key. When an entry in the global branch history table matches the correct prediction, +1 is added to the corresponding history weight. When the entry doesn't match the prediction -1 is added to the history weight. 

Here's a python implementation of a simple single layer perceptron using integer weights:

```python
class Perceptron:
    weights = []
    N = 0
    bias = 0
    threshold = 0

    def __init__(self, N):
        self.N = N
        self.bias = 0
        self.threshold = 2 * N + 14                 # optimal threshold depends on history length
        self.weights = [0] * N      

    def predict(self, global_branch_history):
        running_sum = self.bias
        for i in range(0, self.N):                  # dot product of branch history with the weights
            running_sum += global_branch_history[i] * self.weights[i]
        prediction = -1 if running_sum < 0 else 1
        return (prediction, running_sum)

    def update(self, prediction, actual, global_branch_history, running_sum):
        if (prediction != actual) or (abs(running_sum) < self.threshold):   
            self.bias = self.bias + (1 * actual)
            for i in range(0, self.N):
                self.weights[i] = self.weights[i] + (actual * global_branch_history[i])

    def statistics(self):
        print "bias is: " + str(self.bias) + " weights are: " + str(self.weights)
```

And here's a full branch predictor using a separate perceptron for every branch at a different memory location. A real branch predictor will have to deal with collisions in the hash table. Besides that consideration, this is a reasonably good simulation of a real branch predictor and we can use it to test with other branch prediction methods!

```python
def perceptron_pred(trace, l=1):

    global_branch_history = deque([])
    global_branch_history.extend([0]*l)

    p_list = {}
    num_correct = 0

    for br in trace:            # iterating through each branch
        if br[0] not in p_list:     # if no previous branch from this memory location 
            p_list[br[0]] = Perceptron(l)
        results = p_list[br[0]].predict(global_branch_history)
        pr = results[0]
        running_sum = results [1]
        actual_value = 1 if br[1] else -1
        p_list[br[0]].update(pr, actual_value, global_branch_history, running_sum)
        global_branch_history.appendleft(actual_value)
        global_branch_history.pop()
        if pr == actual_value:
            num_correct += 1
            
    return num_correct
```

Running the branch predictor with some real code compiled using GCC and with a perceptron depth of 32 shows a 98.47% accuracy rate! Why does this branch predictor work so well?

One reason is because it can deal with relatively long branch history. Other methods, like a saturating counter only look at the past several previous branches.  

Second, the perceptron quickly learns the correlations between the historical branch outcomes and the current branch instruction. Branch histories that are not correllated with the branch outcome will tend to be updated +1 and -1 evenly, and so the weight attached will be fairly low. Branch histories that are heavily correlated with the outcome of the branch prediction will be weighted heavily. Thus, the effective predictors are quickly identified and disproportionately used in making the prediction.

See the full code with several other branch prediction techniques and the dataset used to test on github [here](https://github.com/edougal/nn_bp_test).

Output:

```
|Predictor|         |gcc accuracy|         |mcf accuracy|
saturating counter     0.96754             0.89850
perceptron (depth 8)   0.98125             0.91216
perceptron (depth 16)  0.98454             0.91225
perceptron (depth 32)  0.98471             0.91196
```


### References:

Thanks to [this](https://github.com/nihakue/branch_sim) github repository for the datasets used to test. 

The paper that the artificial neural network comes from: 
https://www.cs.utexas.edu/~lin/papers/hpca01.pdf

Mention of an artificial neural network branch predictor in an actual processor. 
http://www.theregister.co.uk/2016/08/22/samsung_m1_core/

Information about the length of the pipeline in a modern processor.
http://www.bit-tech.net/hardware/cpus/2008/11/03/intel-core-i7-nehalem-architecture-dive/5

Information on the percentage of instructions which are branches. 
http://web.engr.oregonstate.edu/~benl/Projects/branch_pred/
