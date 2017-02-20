+++
title = "Extracting text from images is not easy (who would have guessed?)"
Categories = ["Mozilla"]
Tags = ["IoT", "Computer Vision", "Project Lighthouse", "Open-source", "Mozilla",
  "Connected Devices", "OCR", "Rust"]
Description = ""
date = "2017-01-01T23:44:19+01:00"

+++


At its core, Lighthouse is an idea we have been discussing in Connected Devices: can we build a device that will help people with partial or total vision
disabilities?

From there, we started a number of experiments. I figured out it was time to
braindump some of them.

## Our problem


Consider the following example:

![](https://cloud.githubusercontent.com/assets/10190/18631303/80cce95a-7e71-11e6-8a4e-bed542e6e7bb.png)

How do we get from this beautiful picture of Mozilla's Paris office to the text "PRIDE and PREJUDICE", "Jane Austen", "Great Books", "Great Prices", "$9.99", "Super livres", "Super prix"? That's Canadian dollars, if you wonder.

Too easy? What about this one?

![](https://cloud.githubusercontent.com/assets/10190/18631577/13481e2a-7e73-11e6-9057-72aa048d7040.jpeg)

Or that one?

![](https://cloud.githubusercontent.com/assets/10190/18631784/45764bd2-7e74-11e6-9b2c-5ef823fd129a.jpeg)

Not very easy to read, right? Now, remember one thing: these pictures were taken by people who can use a camera correctly because, well, they can see the image. If we want to build a device that will be useful for blind or vision-impaired users, we need a tool that can perform text recognition on images that are taken in much worse conditions or light or angle – or at least, we must be able to tell the user what's the problem with the image.

Actually, extracting text from an image is a Hard Problem (tm). Sufficiently hard that there are tracks in research conferences that deal with this kind of problem. But, surely, in this age of Deep Learning, there must be a Magic Cloud API (tm) that can do that, right?


## Looking for a Cloud API

So, we set out to benchmarkg existing APIs that claim that they can perform text recognition on images. This includes Google's APIs, IBM/AlchemyVision's APIs, CloudSight's APIs, Microsoft's Vision APIs.

If you are curious, I invite you to try with the three samples above. The results are limited. In some cases, we get acceptable recognition. In most cases, the APIs just can't help.

To make things worse, these Magic Cloud APIs are, of course, not free. Using them comes at a cost, in terms of device bandwith, data plan, and actual per-request dollars.

*Bottom line* These Magic Cloud APIs are acceptable for a demo, but are far from sufficient if we want to build a tool useful for an actual blind person.

## Looking for an offline API

These days, the recommended OCR library in the open-source world seems to be Tesseract. Testing it against any of the images above yields strictly nothing usable.

This is not a surprise: Tesseract is fine-tuned for use in a scanner. Give it black letters, white paper, no textures, and a simple layout, and Tesseract will return text with an acceptable quality. Give it anything else and Tesseract will assume that it's black letters, white paper, no textures and a simple layout, and will attempt to interpret every single line, every single intersection as a letter. And fail.

*Bottom line* On its own, Tesseract is not usable.

## Looking for a way out

To summarize, I believe that we have the following problems:

1. Magic Cloud APIs attempt to solve all the issues in the world at once and are consequently great at nothing;
2. Tesseract is too specific to a single use case which is not ours, and consequently not usable as-is for our project.

### Working around the limitations of Tesseract

Let's start with problem 2., because there are many options to solve it. If Tesseract cannot be used on unclean text, we _just_ need to clean up the text, using well-known or not-so-well-known Computer Vision algorithms, before feeding it to Tesseract. A quick look at Stack Overflow indicates that we are hardly the first project attempting to do this and that there are a number of solutions that _should_ work. Sadly, not much feedback on whether they _actually_ work, but hey, that's what experimenting is all about, right?

If we succeed, we will have at hand a super-Tesseract, well-adapted to _some categories of images_. A super-Tesseract that we can further train to adapt it to the images in which we are interested. Say, text recognition on packaging, with the lighting conditions available in a supermarket.

*We have a plan* we don't need to build a full OCR engine, we just need a flexible text cleanup/extraction tool.

### Working around poor lights, angle, ...

Let's be pessimistic and assume that, despite our best efforts, our super-Tesseract will not be able to handle poor light, poor angle, etc. any better than the Magic Cloud APIs.

Super-Tesseract will still have a few aces up its metaphoric sleeve:

1. If we run it locally, on the user's device, it is not limited by bandwidth, data plan or per-request cost;
2. While performing actual text recognition is relatively slow on a smartphone-style device, we have hopes that performing text detection and cleanup can be made quite fast.
3. As part of super-Tesseract, we can build algorithms that can actually determine whether the angle is wrong, or the lightning is bad, etc. All sorts of information that the Magic Cloud APIs do not give us.

Put together, these two aces open several possibilities:

- We can afford to perform at least the first few steps of super-Tesseract *on videos*, rather than on a single image, which will give us more samples from which to extract clean text.
- We can afford to perform these same steps several times, using several different filters and parameters, to offset the light, angle, etc.
- We can provide feedback for the users, actually telling them what's wrong, if they need to turn the cereal bar, or move the camera closer, or if the text is just simply unreadable because of too many textures.

*Bottom line*: We need a very fast implementation of text clean-up/extraction that can also diagnose its own problems.

## Towards super-Tesseract

At this stage, my hopes are to perform the following steps:

a. locating the text in an image;
b. removing everything that isn't the text;
c. turning it if necessary to make sure that it's horizontal;
d. now that the text is as clean as in a book, feed it to Tesseract.

During the past few weeks, I have been experimenting with algorithms for these things. All the problems are tricky, but I am making progress.

a. An algorithm based on https://www.microsoft.com/en-us/research/publication/stroke-width-transform/[Stroke Width Transforms] (SWT) has very good results on locating the text in an image. I have tested the implementation of this algorithm in the https://github.com/liuliu/ccv[CCV] library and it has very good performance, both in terms of speed and in terms of results. So far, so good. As I am prototyping in Rust, I have published https://github.com/Yoric/ccv.rs[very early bindings] – feel free to use them if you want to toy with this algorithm.

b. I have tested several algorithms for cleaning up everything that isn't text. Unfortunately, so far, these algorithms perform quite poorly, both in terms of speed and accuracy. Looking harder at SWT, I believe that this algorithm can be extended to perform the cleanup. I am currently working such an extension, but it might be a few weeks before I can seriously test it.

c. An algorithm called Deskewing is rumored to have good results. I have located an implementation of this as part of a library called
http://www.leptonica.org/[Leptonica]. The implementation is very fast, but it doesn't work too well on my samples. I need to investigate this further. It may be a problem with my samples, which are not sufficiently cleaned up at this stage, or with the algorithm, or even with me misunderstanding the results of the algorithm. Again, I have published https://github.com/Yoric/lepton.rs[minimal Rust bindings for Leptonica], to simplify experimentation.

## Things not text

Byproduct of the study: there are several very important pieces of
information on boxes and clothes that would be useful for blind or vision-impaired users but that most likely cannot be treated by OCR.

I'm thinking of logos (e.g. no amount of OCR will recognize
Coca Cola, Kellogs, ...) and other symbols (e.g. non-recycle plastic,
toxicity warning, hand wash only, ...)

At some point, we will certainly want to recognize them. I suspect that neural networks/deep learning would be pretty good for that purpose, but I have no idea how much effort that would take. I also suspect that Magic Cloud APIs already handle these quite nicely already, so at least for a prototype, it might be useful to use these as a baseline.

# Commenting

You can leave comments https://github.com/Yoric/yoric.github.io/issues/1[on github] or using Disqus below. I haven't decided what's best yet.