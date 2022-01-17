# An Overview of the Tesseract OCR Engine

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
