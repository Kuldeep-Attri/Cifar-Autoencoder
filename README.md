# Convolutional Autoencoders for the Cifar10 Dataset

Making an autoencoder for the MNIST dataset is almost too easy nowadays.  Even a simple 3 hidden layer network made of fully-connected layers can get good results after less than a minute of training on a CPU:

![MNIST 100-100-100-784](/images/100-100-100-784-10epochs.png)

(MNIST images are on the left and autoencoder-reconstructed images are on the right) The network architecture here consisted of fully-connected layers of sizes 100, 100, 100, 784, respectively.  You might notice that these numbers weren't carefully chosen --- indeed, I've gotten similar results on networks with many fewer hidden units as well as networks with only 1 or 2 hidden layers.  

The same can't be said for the Cifar datasets.  The reason for this isn't so much the 3 color channels or the slightly larger pixel count, but rather the internal complexity of the Cifar images.  Compared to simple handwritten digits, there's much more going on, and much more to keep track of, in each Cifar image.  I decided to try a few popular architectures to see how they stacked up against each other.  Here's what I found:

 - Fully connected layers always hurt
 - Reducing the image size too many times (down to 4 x 4 x num_channels, say) destroys performance, regardless of pooling method
 - Max Pooling is OK, but strided convolutions work better
 - Batch Normalization layers do give small but noticable benefits
 - Shallow autoencoders give good results and train quickly
 - Deep convolutional autoencoders actually get *worse* results than shallow 

A word before moving on:  Throughout, all activations are ReLU except for the last layer, which is sigmoid.  All training images have been normalized so their pixel values lie between 0 and 1.  All networks were trained for 10 or fewer epochs, and many were trained for only 2 epochs.  I didn't carry out any sort of hyperparameter searches, although I did increase the number of hidden units in a layer from time to time, when the network seemed to be having real trouble learning.  I found that mean squared error worked better as a loss function than categorical or binary crossentropy, so that's what I used.  Each network used the Adam optimizer available in Keras.  

Also a quick note on the purpose of this autoencoder:  only some of the models below achieve a representation of smaller size than the input image was.  My goal here isn't to come up with a data-compression scheme, so I'm fine with this.  Rather, I'd like to build a noise-cancelling autoencoder, so I'm very happy with whatever works, even if the latent representation is several times larger than the original image.

Finally, my sample sizes are far too small to be sure of any of my above conclusions.  Anyone interested should play around with different sizes, numbers, and types of layers to see for themselves what works and what doesn't work.  Also, these images are so small that once an autoencoder becomes "good enough", it's hard for the human eye to see the difference between one model and another.  We could look at the loss function, but mean-squared-error leaves a lot to be desired and probably won't help us discriminate between the best models.  


# Some Poor-Performance Autoencoders
## Fully Connected
I wanted to start with a straightforward comparison between the simplicity of the MNIST dataset versus the complexity of the Cifar datasets.  So, I tried several autoencoders made solely of fully connected layers.  The results were not great - all models constructed blurry images from which it would be difficult to extract class labels.  One slightly surprising result here is that the more layers, the worse the performance.  

![Dense w/ One Hidden Layer](/images/1024-3072-Dense-5epochs.png)
This model has just one hidden layer with 1024 units.  Trained for 5 epochs.



![Dense w/ Three Hidden Layers](/images/3072-2048-1024-3072-Dense-5epochs.png)
This model has three hidden layers with 3072, 2048, 1024 units, respectively.  Trained for 5 epochs.

To be fair, the larger model may simply need more training time to catch up to the smaller one.  

## Convolutional + Fully Connected
Next I tried models which begin and end with convolution layers, but which have some dense layers in the middle.  The main idea was that convolutional filters should be able to pick up on certain geometric features, and the dense layers might be able to figure out a representation for these features.  Again the results were not very promising, but maybe more training time would improve the models.  The following image came from a model with structure Conv->Pool->Conv->Pool->Conv->Dense->Dense->Upsample->Conv->Upsample->Conv, where each convolution except the final one has num_channels=32, kernel_size=3, strides=1, padding=same, activation=relu.  The final convolution is a 1x1 convolution with sigmoid activation.  This model was trained for 5 epochs.

![Convolutional + Dense](/images/ConvDense-8-8-32-minsize.png)


## Convolutional + Fully Connected + Batch Normalization 
Finally I wanted to see if we could save the previous model by throwing in another convolutional layer before the fully connected layers, and inserting some batch normalization layers here and there.  Some of these models performed similarly to the previous models, some performed worse, and one in particular performed so poorly I had to include the image here:

![Conv + FC + BN](/images/Everything-including-kitchen-sink.png)

# Performant Autoencoders

Performance improved drastically as soon as I removed all fully-connected layers.  However, deeper convolutional autoencoders perform about as well as our first 1-hidden-layer fully-connected model.  For this reason I'm starting this section with a shallow model - the entire autoencoder looks like Conv->MaxPool->Conv->Upsample->Conv->Conv, and was trained for 2 epochs.

![Shallow Conv](/images/Shallow-Conv.png)

We get another performance boost by replacing max pooling layers with strided convolutions.  The MaxPool layer above is replaced with a Conv with num_channels=32, kernel_size=3 and strides=2.  Again we train for 2 epochs.

![Shallow Conv w/ Strided Convs](/images/Shallow-Conv-Strided-Convs.png)

Next I started adding on more and more convolution layers with strided convolutions in place of pooling, and the results actually started getting worse and worse.  Feel free to try this on your own.  However, a few batch normalization layers here and there helped with performance.  Even so, the models with more layers still do worse than the simpler, shorter models, at least with the small amount of training time I'm able to put into them.  Anyone with access to a GPU should try and see what is possible with a deeper model and a lot more training.  The best performance I was able to achieve was by combining shallow convolutions + strided convolutions + batch normalization.  The reconstructed images are so good that I can't tell the difference between autoencoder input and output.  We can keep trying to reduce the loss by playing with hyperparameters and network architecture, but at this resolution it really won't make a visual difference.  It would be interesting to continue this process on images of higher resolution, simply to see what sort of qualitative changes in the reconstructed images emerge from certain choices of regularization, architecture, etc.

![Conv + Strided Conv + BN](/images/shallow-conv-with-BN-4epochs.png)

# A De-noising Autoencoder

As a quick application, I trained the most performant model above as a pair of denoising autoencoders.  The first one is trained on the Cifar training set, normalized to the interval [0,1], with noise coming from a normal distribution with mean 0 and standard deviation 1:

![Noisy and denoised images, std dev 1](/images/denoise-with-bn-noise100-9epochs.png)

The second one was trained to a more reasonable task - the normal distribution now has mean 0 and standard deviation 0.3.

![Noisy and denoised images, std dev 0.3](/images/denoise-with-bn-noise30-9epochs.png)

Finally we look at what happens when we use the previous (std. dev. 0.3 noise) autoencoder on a set of test images which have not had noise added to them. 

![Cifar and 'denoised' images](/images/denoising-images-with-no-noise.png)

As you can see, this model is doing some sort of probabilistic "smearing out" of the image so that the effects of noise cancel each other out.  That is, this model doesn't know what a Cifar image ought to look like, and instead just does something like a simple weighted average over neighbors to determine the output intensities for a given input image.










