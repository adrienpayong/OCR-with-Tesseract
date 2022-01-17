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
