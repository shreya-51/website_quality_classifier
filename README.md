# website quality classifier

## Overview:
 1. a custom "coherency" metric was defined to label thousands of urls quickly
    - motivated by: if "high quality" webpages increase a language model's reasoning ability --> then high logical reasoning must be present in these webpages --> then "high quality" webpages would have more related content
        - ex: YouTube video --> "low quality" --> text from webpage: ['How to Make Coffee', 'subscribe']
        - ex: Medium article --> "high quality" --> text from webpage: ['Coffee can be made on the stove.', 'Coffee can also be made in a french press.']
 2. base model trained using this custom metric as labels
 3. given new incoming data, base model can then be updated to adjust the model
 4. goal: over a longer period of time, model would deviate from emulating the custom metric to emulating the real function that defines the users' ideas of "high quality" and "low quality" webpages
 * extra: provided function to tune between precision and recall while choosing model


Note: Found it very hard to quantify "logical reasoning". An interesting approach attempted: poking the latent space of an autoencoder to find the "logical reasoning axis", then projecting various passages along this access to see how much "logical reasoning" it had. Could possibly work if "logical resoning" was further defined + more data + more computational resources.


## Data Preperation

Given a text file of urls, html and text information was parsed. Then, the following features were calculated:
1. word count
2. sentence count
3. a custom metric

## Custom Metric

A custom metric was calculated in order to provide proxy labels to train the base model.

For each url, the metric was calculated as follows:
1. a set of two consecutive sentences was chosen at random
2. these sentences were embedded and compared using a cross-encoder, which provided a sentence similarity score between the two sentences
3. steps 1+2 were repeated 10 times
4. average of the 10 sentence similarity scores were taken
5. metric = sigmoid(avg_sentence_similarity * sentence_count * scaling_factor=0.01)

This method was chosen to find a metric that (1) could be calculated (given limited resources) and (2) in some way lightly represented how "coherent" different parts of the text were. This was motivated by [this].

Other techniques attempted:
 - Averaging similarities between all setnences (too computationally demanding for available resources)
 - Sentence embeddings using various embedding models (upon manual inspected, cross-encoder provided slightly better results)

## Training base model

Features: [here]<br>
Labels: for each url, if metric >= 0.51 then "high quality", else "low quality"<br>
Model: RandomForest

Note: Essentially, we are here trying to model the function used to provide the metric, as this is what was used to label the data.

Other techniques attempted:
 - logistic regression (unable to capture non-linearity of sigmoid)
 - neural networks (too many params + not enough data = risk of overfitting)
 - SGDClassifier (used in order to address the addition of new data later, also unable to capture non-linearity, but technically can be adapted to add a sigmoid at the end using log-loss)
 - using html and text embeddings as features for each webpage (too computationally demanding for available resources)
 - using html and text summaries in embeddings as features for each webpage (still too computationally demanding for available resources)

## Adding more data

In order to add more data (i.e. adjust as per user feedback), the RandomForestClassifier was refited on new data.

Pros: worked quickly with the already defined RandomForest model<br>
Cons: can quickly increase complexity as additional trees are added each time we add data

Better solution: use online learning with a non-linear model

## Tuning precision and recall

A function is provided to use GridSearch to find the precision and recall for various parameters of the model.

# In the repo:

notebooks:
 - prepare_data.ipynb: given a list of urls (urls_test.txt), calculates the features to be used in the classifier (required for both training and inference)
 - main.ipynb: trains base model, inference of base model, model update based on new data, and precision/recall tuning
 - unused_classifiers.ipynb: some of the other sample classifiers used throughout the journey

sample run directory: 
 - urls.txt contains our starting list of urls (two suggested urls used for demonstration purposes)
 - prepare_data.ipynb takes the list of urls and outputs features_test.csv (a list of features for each url)
 - main.ipynb uses the trained model (rf_model.joblib) and used scalar (scalar.joblib) to generate the predictions for the urls in urls_test.txt (output stored in inference_out.csv)

sample run outputs (shown in inference_out.csv):
 - https://waitbutwhy.com/2017/04/neuralink.html
    - inference label: 1 (i.e. "high quality") 
 - https://www.deviantart.com/estigiakinslayer/art/Snow-Leopard-993653025
    - inference label: 0 (i.e. "low quality")

Notes on improvement of the base model specifically:
 - The base model assumes that longer texts have more logical reasoning. Clearly, this is not always true, so long nonsensical texts may also be classified as "high quality".
    - ex. the landing page of a news website, where there is large amount of text due to various article options
    - possible solution: differenciate between text pulled from different kinds of html sections, note size of html component as relative importance measure
 - The entire pipeline is also largely dependent on how well the html and text parsing is. Improving this would allow for more control on the training side, as the features would be more reliable. Datasets can also be parsed for this information, but would require some filtering of the data first.
 - There are a few failure cases that come from the vauge definition of "logical reasoning", i.e. a landing page of a product, where there is lots of related text, but not really much logical reasoning.
 - Given more computational power, embedding either all, a summary, or some part of the html and/or text could provide interesting results.

