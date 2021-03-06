Ruby bindings for [liblinear](http://www.csie.ntu.edu.tw/~cjlin/liblinear/)

Quick Start
=====

### Loading a problem in the libsvm format

    RubyLinear::Problem.load_file("/path/to/file",bias)
    
### Defining a problem from an array of samples

    samples = [{1 => 1, 2=> 0.2}, {3 => 1, 4=> 0.2}, {2 => 1, 3 => 0.3}]
    max_feature = samples.map {|sample| sample.keys.max}.max
        
    labels = [1,2,1]
    RubyLinear::Problem.new labels, samples, 1.0, max_feature
    
Your sample can of course be sparse: you only need to name features with a non-zero associated value
    

### Loading a model from a file
  
    RubyLinear::Model.load_file('/path/to/file')
    

### Training a model from parameters and a problem
    
    RubyLinear::Model.new(problem, :solver => RubyLinear::L1R_L2LOSS_SVC)

### Changing the default parameters    

    RubyLinear::Model.new(problem, :solver => RubyLinear::L1R_L2LOSS_SVC, 
                                   :c => 1.1, :eps => 0.02, :weights => {2 => 0.9})
    #use C=1.1, eps = 0.02 and apply a weight of 0.9 to class 2
    
### Predicting a value

    sample = {1 => 0.3, 4 => 0.1}
    model.predict(sample)
   
### Predicting a value and getting back a score for each class
    sample = {1 => 0.3, 4 => 0.1}
    winner, scores = model.predict_values(sample)
    # winner is 1 (ie the sample has class 1)
    # scores is {1=>0.10716629302903406, 2=>0.0}: class 1 scored 0.107... and  class 2 scored 0

What is this
============

Liblinear is a linear classifier. From its home page:

> LIBLINEAR is a linear classifier for data with millions of instances and features. It supports
> L2-regularized classifiers 
> L2-loss linear SVM, L1-loss linear SVM, and logistic regression (LR)
> L1-regularized classifiers (after version 1.4) 
> L2-loss linear SVM and logistic regression (LR)

In a nutshell if you provide a bunch of examples and what classes they should fall into, Liblinear will predict what classes future datapoints should fall in. Examples include classifying text into spam or not spam, sorting news articles into the list of topics or determining whether a tweet is expressing something positive or negative.

## Classifying text

### Creating the problem
If you are classifying text, you first need to break up your text into tokens. This might be individual words, pairs of words (bigrams) or triples etc. Each one of these is a feature. For example if the 3 pieces of text in our training set were

    s1 = "Hello world"
    s2 = "Bonjour"
    s3 = "Guten tag"
    s4 = "tag world"
    
and we were using individual words as our features then our dictionary contains the words `Hello, world, Bonjour, Guten, tag`. The feature indexes are the 1-based indexes into this list of words so s1 has features `1,2` s2 has feature `3`, s3 has features `4,5` and s4  has features `2,5`. You need to keep hold of the mapping of these words to feature numbers - You'll need this later on.

You must then assign labels: these are the classes you are trying to sort things into. For example 1 is english, 2 is french and 3 is german (these are arbitrary numbers - they don't need to be consecutive or start from 1). The code to setup such a sample set is

    samples = [
      { 1 => 1, 2 => 1},
      { 3 => 1},
      { 4 => 1, 5 => 1},
      { 2 => 1, 5 => 1}
    ]
    labels = [1,2,3,1]

    problem = RubyLinear::Problem.new labels, samples, 1.0, 5

The 3rd parameter is the bias term - check the LibLinear documentation for its interpretation. The last parameter is the maximum feature index (5 in this case). If you have saved problem data in the libsvm format, you can load it with `RubyLinear::Problem.load_file`. 

#### On weights.

The weights for features are the values in the sample hashes. Here we've just used 1 to indicate the presence of a feature and nothing to indicate its absence. A more sophisticated approach would have different values depending on how relevant the feature is, for example by using tf-idf so that features that appear more often in a document or are more significant generally have higher significance.

### Training the model

Once you have a problem instance, you can train a model:

    model = RubyLinear::Model.new(problem, :solver => RubyLinear::L1R_L2LOSS_SVC)
    
will create and train a model from your sample data. Liblinear provides a bunch of different solvers: `RubyLinear::L2R_LR`, `RubyLinear::L2R_L2LOSS_SVC_DUAL`, `RubyLinear::L2R_L2LOSS_SVC`, `RubyLinear::L2R_L1LOSS_SVC_DUAL`, `RubyLinear::MCSVM_CS`, `RubyLinear::L1R_L2LOSS_SVC`, `RubyLinear::L1R_LR`, `RubyLinear::L2R_LR_DUAL`. The Liblinear website has more information on the differences between them.

You can set liblinear's `C` and `eps` options (the defaults are 1 and 0.01 respectively)

    model = RubyLinear::Model.new(problem, :solver => RubyLinear::L1R_L2LOSS_SVC, :c => 1, :eps => 0.01)
    
Models can be saved to disk (`model.save(path_to_file)`) and loaded (`RubyLinear::Model.load_file(path_to_file)`). These use the standard liblinear formats so you should be able to use models trained with the liblinear command line tools.

Once you have a model you can feed it samples and it will return the class it thinks is the best match. To do this you need to turn the text into a hash of feature indexes to weights. If you have a word which isn't in your mapping (ie you come across a word which wasn't in your training set, just ignore it). For example if the text to test was "Hello bob", you would do

    model.predict({1 => 1})

Feature 1 is present and so is added to the hash, but  "bob" is an unknown word and so gets dropped. The returned value is the label of the predicted class.

