# Dissecting Tesseract OCR engine

Tesseract is a free and open-source OCR engine created by HP between 1984 and 1994.
It arrived out of nowhere for the 1995 UNLV Annual Test of OCR Accuracy , shined brilliantly with its findings, and then disappeared behind the same veil of secrecy that it had been constructed under. Details of the architecture and algorithms may now be exposed for the first time.

Tesseract originated as a PhD research project at HP Labs in Bristol and quickly garnered traction as a potential software and/or hardware add-on for HP's series of flatbed scanners.The fact that commercial OCR engines were in their infancy at the time offered motivation, since they failed spectacularly on anything but the highest quality print. Tesseract had a large accuracy advantage over commercial engines after a collaborative effort between HP Labs Bristol and HP's scanning business in Colorado, but it did not become a product.
The next step of its evolution took place at HP Labs Bristol as a research of OCR for compression.
The work focused on enhancing rejection efficiency rather than base-level accuracy.
Development on this project came to a halt by the end of 1994.
The engine was sent to UNLV for the 1995 Annual Test of OCR Accuracy, where it outperformed commercial engines at the time.
HP published Tesseract as open source in late 2005.  It is now available
at https://github.com/tesseract-ocr.

## Architecture

Tesseract never required its own page layout analysis because HP had separately developed page layout analysis technology that was utilized in products (and hence not provided for open-source), and Tesseract never needed its own page layout analysis.
As a result, Tesseract believes its input is a binary image with optional polygonal text sections specified.

Processing follows a standard step-by-step pipeline, however some of the phases were odd at the time and may still be so now.
The first phase is a linked component analysis, in which the component outlines are recorded.
This was a computationally costly design choice at the time, but it had a huge advantage: it is straightforward to discover inverted text and recognize it as simply as black-on-white text by inspecting the nesting of outlines and the number of child and grandchild outlines.
Tesseract was most likely the first OCR engine capable of handling white-on-black text so easily.
At this point, outlines are simply nested together to form Blobs. 

Blobs are structured into text lines, and the lines and regions are evaluated to determine if the text is fixed pitch or proportional.
Depending on the kind of character spacing, text lines are divided into words in a variety of ways.
Character cells instantly cut fixed pitch text. Proportional text is divided into words with definite and fuzzy spacing.

The recognition procedure is then repeated twice.
The first pass attempts to identify each word individually.
Each suitable term is sent into an adaptive classifier as training data.
The adaptive classifier is then given the opportunity to detect text farther down the page more correctly. Because the adaptive classifier may have learnt anything important too late to contribute at the top of the page, a second pass through the page is performed, in which words that were not identified well enough are recognized again. In the last step, fuzzy spaces are resolved and multiple assumptions for the x-height are tested in order to identify small-cap tex. 

##  Line and Word Finding
## Line Finding
One of the few aspects of Tesseract that has previously been disclosed is the line detection method .
The line finding technique is meant to detect a skewed page without having to de-skew it, preserving image quality.
Blob filtering and line creation are critical steps in the process.

Assuming that page layout analysis has previously produced text areas of fairly equal text size, a basic percentile height filter eliminates drop-caps and vertically touching characters.
Because the median height approximates the text size in the area, it is safe to filter away blobs smaller than some percentage of the median height, which are most likely punctuation, diacritical markings, and noise.

The filtered blobs are more likely to match a model of parallel but sloping non-overlapping lines.
Sorting and processing the blobs by x-coordinate allows you to assign each blob to a distinct text line while monitoring the slope throughout the page, with a far lower risk of assigning to the wrong text line in the case of skew.
Once the filtered blobs have been allocated to lines, the baselines are estimated using a least median of squares fit , and the filtered-out blobs are fitted back into the relevant lines.

The last phase in the line construction procedure joins blobs that overlap by at least half horizontally, linking diacritical markings with the right base and appropriately correlating sections of certain damaged letters cap tex. 

## Baseline Fitting

Once the text lines have been identified, the baselines are more accurately fitted using a quadratic spline.
This was yet another first for an OCR system, allowing Tesseract to handle pages with curved baselines , which are a frequent artifact in scanning, not only at book bindings].

The baselines are fitted by grouping the blobs with a fairly consistent deviation from the initial straight baseline.
A least squares fit is used to fit a quadratic spline to the most populated partition (assumed to be the baseline).
The quadratic spline has the benefit of being generally stable in this computation, but it has the problem of causing discontinuities when several spline segments are necessary.
A more classic cubic spline  could be preferred. 

![source](https://github.com/adrienpayong/OCRproject/blob/main/Capture1.PNG)

Figure 1 depicts a text line with a fitted baseline, descender line, meanline, and ascender line.
These lines are all "parallel" (the y separation is constant over the length) and slightly bent.

The ascender line is cyan (prints as light gray), whereas the black line above it is straight.
A closer look reveals that the cyan/gray line is bent in relation to the straight black line above it. 

## Fixed Pitch Detection and Chopping

Tesseract analyzes the text lines to see whether they are fixed pitch.
Tesseract slices the words into characters using the pitch when it detects fixed pitch text and disables the chopper and associator on these words for the word recognition stage.Figure 2 is an example of a fixed-pitch word. 

![source](https://github.com/adrienpayong/OCRproject/blob/main/Capture2.PNG)

## Proportional Word Finding

Non-fixed-pitch or proportional text spacing is a difficult undertaking.
Figure 3 depicts some common difficulties.
The space between the tens and units in '11.9 percent' is comparable to the general space, and is unquestionably greater than the kerned space between 'erated' and 'trash'.
There is no horizontal space between the boundary boxes 'of' and 'financial.'
Tesseract overcomes the majority of these issues by measuring gaps in a narrow vertical range between the baseline and the mean line.
At this step, spaces around the threshold are rendered fuzzy so that a final judgment may be made following word recognition. 

![source](https://github.com/adrienpayong/OCRproject/blob/main/Capture3.PNG)

## Word Recognition

Identifying how a word should be divided into characters is part of the recognition process for any character recognition engine.
First, the line finding segmentation output is classified.
The remainder of the word recognition stage is solely applicable to non-fixed-pitch text. 

### Chopping Joined Character

While the output from a word (see is unsatisfying, Tesseract tries to enhance it by slicing the blob with the lowest confidence from the character classifier.
Concave vertices of a polygonal approximation  of the outline are used to find candidate chop sites, which may contain another concave vertex opposite or a line segment.
To correctly separate connected characters from the ASCII set, up to three pairs of chop points may be required.

![source](https://github.com/adrienpayong/OCRproject/blob/main/Captureoutline.PNG)

Figure 4 depicts a series of possible chop spots with arrows and the chosen chop as a line across the outline where the 'r' meets the'm'.
Chops are done in descending order of priority.
Any chop that fails to increase the associator's confidence in the outcome is undone, but not fully discarded, so that the chop may be re-used by the associator later if necessary. 

### Associating Broken Characters

When all possible chops have been explored, the word is delivered to the associator if it is still not good enough.
The associator searches the segmentation network for feasible combinations of the maximum sliced blobs into candidate characters using an A* (best first) search.
It does this without actually constructing the segmentation graph, instead keeping a hash table of visited states.
The A* search works by selecting candidate new states from a priority queue and assessing them by classifying unclassified fragment combinations.

It may be argued that this fully-chop-then-associate strategy is at best inefficient, and at worst prone to missing key chops, and that could be correct.
The chop-then-associate technique has the benefit of simplifying the data structures necessary to maintain the whole segmentation graph. Tesseract's precision on broken characters was well ahead of the commercial engines of the time when the A* segmentation search was initially deployed around 1989.
Figure 5 is an illustration of this.
The character classifier, which could effortlessly identify broken characters, was a critical component of that accomplishment. 

![source](https://github.com/adrienpayong/OCRproject/blob/main/Capture23.PNG)

## Static Character Classifier
### Features

An early version of Tesseract made use of topological properties provided by Shillman et al.

Although these properties are beautifully independent of font and size, they are not resilient to the difficulties observed in real-life images, as Bokser demonstrates.
An intermediate solution proposed using polygonal approximation segments as features, however this technique is likewise not resistant to broken characters.
For example, in Fig. 6(a), the right side of the shaft is divided into two major portions, but in Fig. 6(b), there is just one component.

![source](https://github.com/adrienpayong/OCRproject/blob/main/CaptureZE.PNG)



The revolutionary solution is the notion that the characteristics in the unknown do not have to be the same as the features in the training data.
The segments of a polygonal approximation are utilized for features during training, but in recognition, features of a tiny, fixed length (in normalized units) are retrieved from the outline and matched many-to-one against the clustered prototype features of the training data.
The short, thick lines in Fig. 6(c) are the extracted characteristics from the unknown, while the thin, longer lines are the clustered segments of the polygonal approximation that are utilized as prototypes.
One prototype that connects the two pieces is absolutely unique.
Three features on one side and two on the other are mismatched, but every other prototype and feature is nicely matched.

This example demonstrates how this approach of matching minor characteristics to huge prototypes can easily handle the identification of damaged images.
Its fundamental issue is that determining the distance between an unknown and a prototype has a very high computational cost. The features retrieved from the unknown are therefore three-dimensional (x, y position, angle), with 50-100 features in a character, but the prototype features are four-dimensional (x, y, position, angle, length), with 10-20 characteristics in a prototype configuration. 

### Classification

Classification is a two-step procedure.
A class pruner generates a shortlist of character classes that the unknown may match in the first stage.
Each feature retrieves a bit-vector of classes that it may match from a coarsely quantized 3-dimensional look-up table, and the bit-vectors are averaged over all features.
The classes with the highest counts (after adjusting for the predicted number of features) are selected for the following stage.

Each unknown characteristic searches for a bit vector of prototypes of the specified class that it potentially match, and then computes the actual similarity between them.
Because each prototype character class is represented by a logical sum-of-product expression with each term referred to as a configuration, the distance calculation procedure maintains track of the total similarity evidence of each feature in each configuration as well as of each prototype.
The best combined distance, determined from the summed feature and prototype evidences, is the best across all of the class's recorded configurations.

### Training Data

Because the classifier can readily recognize damaged characters, it was not trained on damaged characters.
In reality, the classifier was trained on only 20 samples of 94 characters from eight fonts in a single size, but with four attributes (normal, bold, italic, bold italic), for a total of 60160 training samples.

This stands in stark contrast to other published classifiers, such as the Calera classifier, which has over a million training data, and Baird's 100-font classifier, which has 1175000 training samples. 

## Linguistic Analysis
Tesseract provides just a little amount of language analysis.
The linguistic module (misnamed the permuter) selects the best available word string in each of the following categories whenever the word recognition module considers a new segmentation:
Top dictionary term, Top numeric word, Top UPPER case word, Top lower case word (with optional starting upper), Top classifier choice word
The final selection for a particular segmentation is simply the word with the lowest overall distance rating, multiplied by a separate constant for each of the aforementioned categories.

Words from various segmentations may include a variable number of characters.
Even when a classifier promises to provide probabilities, which Tesseract does not, it is difficult to compare these terms directly.
Tesseract solves this challenge by creating two integers for each character classification.
The first, known as the confidence, is the normalized distance from the prototype minus the confidence.
This allows it to be a "confidence" in the sense that larger numbers are better, but it is still a distance, since the farther away from zero, the greater the distance.
The rating is calculated by multiplying the normalized distance from the prototype by the entire outline length of the unknown character.
Because the overall outline length for all characters inside a word is always the same, ratings for characters within a word may be meaningfully totaled. 

## Adaptive Classifier
It has been proposed and proved that the use of an adaptive classifier may help OCR engines.
Because the static classifier must be capable of generalizing to any font, its ability to distinguish between distinct characters or between characters and non-characters suffers.
In situations when the number of fonts is restricted, a more font-sensitive adaptive classifier trained from the output of the static classifier is usually utilized to achieve more differentiation within each page.

Tesseract does not utilize a template classifier, instead using the same features and classifier as the static classifier.
Apart from the training data, the only significant difference between the static and adaptive classifiers is that the adaptive classifier uses isotropic baseline/x-height normalization, whereas the static classifier normalizes characters by the centroid (first moments) for position and second moments for anisotropic size normalization.

![source](https://github.com/adrienpayong/OCRproject/blob/main/Capturecl.PNG)

The baseline/x-height normalization improves resilience to noise specks while also making it simpler to discern upper and lower case characters.
The major advantage of character moment normalization is that it removes font aspect ratio and certain font stroke width.
It also makes subscript and superscript detection easier, although it needs an extra classifier feature to identify certain upper and lower case letters.
Figure 7 depicts three letters in baseline/x-height normalized and moment normalized form. 

Reference: ![source](https://static.googleusercontent.com/media/research.google.com/fr//pubs/archive/33418.pdf)

