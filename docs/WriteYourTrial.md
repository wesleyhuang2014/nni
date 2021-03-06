**Write a Trial which can Run on NNI**
===
There would be only a few changes on your existing trial(model) code to make the code runnable on NNI. We provide two approaches for you to modify your code: `Python annotation` and `NNI APIs for trial`

## NNI APIs 
We also support NNI APIs for trial code. By using this approach, you should first prepare a search space file. An example is shown below: 
```
{
    "dropout_rate":{"_type":"uniform","_value":[0.1,0.5]},
    "conv_size":{"_type":"choice","_value":[2,3,5,7]},
    "hidden_size":{"_type":"choice","_value":[124, 512, 1024]},
    "learning_rate":{"_type":"uniform","_value":[0.0001, 0.1]}
}
```
You can refer to [here](SearchSpaceSpec.md) for the tutorial of search space.

Then, include `import nni` in your trial code to use NNI APIs. Using the line: 
```
RECEIVED_PARAMS = nni.get_parameters()
```
to get hyper-parameters' values assigned by tuner. `RECEIVED_PARAMS` is an object, for example: 
```
{"conv_size": 2, "hidden_size": 124, "learning_rate": 0.0307, "dropout_rate": 0.2029}
```

On the other hand, you can use the API: `nni.report_intermediate_result(accuracy)` to send `accuracy` to assessor. And use `nni.report_final_result(accuracy)` to send `accuracy` to tuner. Here `accuracy` could be any python data type, but **NOTE that if you use built-in tuner/assessor, `accuracy` should be a numerical variable(e.g. float, int)**.

The assessor will decide which trial should early stop based on the history performance of trial(intermediate result of one trial).
The tuner will generate next parameters/architecture based on the explore history(final result of all trials).

In the yaml configure file, you need two lines to enable NNI APIs:
```
useAnnotation: false
searchSpacePath: /path/to/your/search_space.json
```

You can refer to [here](../examples/trials/README.md) for more information about how to write trial code using NNI APIs.

## NNI Annotation
We designed a new syntax for users to annotate the variables they want to tune and in what range they want to tune the variables. Also, they can annotate which variable they want to report as intermediate result to `assessor`, and which variable to report as the final result (e.g. model accuracy) to `tuner`. A really appealing feature of our NNI annotation is that it exists as comments in your code, which means you can run your code as before without NNI. Let's look at an example, below is a piece of tensorflow code:
```diff
with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())
+   """@nni.variable(nni.choice(50, 250, 500), name=batch_size)"""
    batch_size = 128
    for i in range(10000):
        batch = mnist.train.next_batch(batch_size)
+       """@nni.variable(nni.choice(1, 5), name=dropout_rate)"""
        dropout_rate = 0.5
        mnist_network.train_step.run(feed_dict={mnist_network.images: batch[0],
                                                mnist_network.labels: batch[1],
                                                mnist_network.keep_prob: dropout_rate})
        if i % 100 == 0:
            test_acc = mnist_network.accuracy.eval(
                feed_dict={mnist_network.images: mnist.test.images,
                            mnist_network.labels: mnist.test.labels,
                            mnist_network.keep_prob: 1.0})
+           """@nni.report_intermediate_result(test_acc)"""

    test_acc = mnist_network.accuracy.eval(
        feed_dict={mnist_network.images: mnist.test.images,
                    mnist_network.labels: mnist.test.labels,
                    mnist_network.keep_prob: 1.0})
+   """@nni.report_final_result(test_acc)"""
```

Let's say you want to tune batch\_size and dropout\_rate, and report test\_acc every 100 steps, at last report test\_acc as final result. With our NNI annotation, your code would look like below:


Simply adding four lines would make your code runnable on NNI. You can still run your code independently. `@nni.variable` works on its next line assignment, and `@nni.report_intermediate_result`/`@nni.report_final_result` would send the data to assessor/tuner at that line. Please refer to [here](../tools/annotation/README.md) for more annotation syntax and more powerful usage. In the yaml configure file, you need one line to enable NNI annotation:
```
useAnnotation: true
```

For users to correctly leverage NNI annotation, we briefly introduce how NNI annotation works here: NNI precompiles users' trial code to find all the annotations each of which is one line with `"""@nni` at the head of the line. Then NNI replaces each annotation with a corresponding NNI API at the location where the annotation is.

**Note that: in your trial code, you can use either one of NNI APIs and NNI annotation, but not both of them simultaneously.** 