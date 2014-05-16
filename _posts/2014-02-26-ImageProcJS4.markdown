---
layout: post
title:  "JavaScript Image Processing (4) - Histogram Equalization"
date:   2014-02-26 11:29:00
uid: 6
categories: Image Processing
---

It is time we work on something more sophisticated. Well, not actually that sophisticated, but complex enough compared
to what we have done so far. Histogram equalization. It is a commonly used technique to save many many many poorly
exposed images.

### Histogram and Cumulative Distribution Function
First of all, what exactly is a <a href="http://en.wikipedia.org/wiki/Histogram">histogram</a>? Basically it is a statistics
telling us about the distribution of the pixels values in a given image - how many pixels are bright, how many are dark, etc.

Let's consider a grayscale image for the moment. We know that there are typically 256 gray scales (i.e. pixel values) in such images. It
is not very hard to come up with a histogram for such kind of image - we can simply count the number of pixels having different
pixels values. Say we find $n\_0$ pixels have gray scale 0, $n\_1$ pixels have value 1, ..., $n\_i$ pixels have value $i$, and so on and so on.
After counting the pixels of all 256 possible gray scales, we get ourselves a histogram! Easy huh?

We can also think of the creation of a histogram as a classification of pixels according to their values, and the histogram
is nothing more than a table recording the number of the pixels in each of the 256 groups (or more commonly, 'bins'). Therefore,
a histogram provides a description of the pixel value distribution in an image - how many pixels are bright, how many pixels are
dark, and how many mid-tone pixels, etc. This is very useful for us to design image processing algorithms because we
can use it as a compact representation of an image (think about 1 million pixels versus 256 numbers!). The downside is we
can not get much out of a histogram if we have a complex problem to solve, since we only have a very high level (and also very vague)
summary of brightness distribution over an image.

Just a side note, we don't necessarily have to use 256 bins for the counting - any number of bins, as long as it is sufficient for the
application at hand, can be used. One typical case using a very small number of bins for histogram computation is color
reduction, where we try to reduce the number of colors in images while maintaining their original visual appearance as much as possible.

For color images, say RGB images, things are just as straightforward. The only difference is that we have 3 channels, i.e. R, G, B channels,
instead of just a single brightness channel. So what? We can just create 3 histograms for these channels and we are done! Of
course, sometimes we are only interested in the brightness of an image. In those cases, we can convert the color image to
  a grayscale one first, and compute the histogram for the grayscale image. How do we convert a color image to a grayscale one?
Simple! We can use a rough estimation with [the equation](http://en.wikipedia.org/wiki/Grayscale) below:
$$ l = 0.299 \times r + 0.587 \times g + 0.114 \times b $$
{% highlight javascript %}
var filters = {
    grayscale : function( src ) {
        return src.map(
            function(r, g, b, a) {
                var lev = Math.round(r * 0.299 + g * 0.587 + b * 0.114);
                return new Color(lev, lev, lev, a);
            }
        );
    }
};
{% endhighlight %}

<div align="center">
<table>
<tr>
<td><img class="post exampleimg" src="/projects/imageprocjs/seal.jpg"></td>
<td><img class="post exampleimg" src="/projects/imageprocjs/images/grayscale.png"></td>
</tr>
<tr>
<td>Source Image</td><td>Converted Grayscale Image</td>
</tr>
</table>
</div>
A conversion using the above equation works just fine, and we are ready to get compute a histogram for any image!
The computation of histogram is pretty straightforward:
{% highlight javascript %}
// build histogram of specified image region
function histogram(img, x1, y1, x2, y2, num_bins)
{
    if( num_bins == undefined )
        num_bins = 256;

    var h = img.h, w = img.w;
    var hist = [];
    var i, x, y, idx, val;

    // initialize the histogram
    for(i=0;i<num_bins;++i)
        hist[i] = 0;

    // loop over every single pixel
    for(y=y1,idx=0;y<y2;++y)
    {
        for(x=x1;x<x2;++x,idx+=4)
        {
            // figure out which bin it is in
            val = Math.floor((img.data[idx] / 255.0) * (num_bins-1));
            ++hist[val];
        }
    }

    return hist;
}
{% endhighlight %}
<div align="center">
<table>
<tr><td><img class="post exampleimg" src="/projects/imageprocjs/images/hist0.png"></td></tr>
<tr><td>Seal and its histogram</td></tr>
</table>
</div>

Yes, the jagged blue area is the histogram. The horizontal axis represents the pixel values, and the vertical axis is
 their frequency. Not so shabby, right? You may ask what on earth is the red curve in the plot, we
 will get to that in a bit.

Let's first formalize the definition of the histogram. The histogram can be viewed as a function of pixel values $h(p): R \to R$.
This function maps a given pixel value $p$ to its frequency, $\frac{n(p)}{N}$, in the entire image. Here we use
$n(p)$ for as the number of pixels having a brightness value $p$, and $N$ as the total number of pixels in an image. Since
the number of pixels in an image is a normalizing factor, we can also just use $n(p)$ as the frequency as well.

Now what exactly is the red curve? It is the so-call [cumulative distribution function](http://en.wikipedia.org/wiki/Cumulative_distribution_function)(CDF).
Don't be scared away by its long name, it is just a partial sum of the histogram we just build:
$$c(p) = \sum_{i=0}^p n(i)$$
Yes, $c(p)$ means the number of pixels with a value smaller or equal than $p$. So $c(0)$ means the number of the pure black (0, 0, 0) pixels,
 while $c(255)$ equals to the total number of pixels. If we are using a normalized version of histogram, we will have

$$c(p) = \frac{1}{N} \sum_{i=0}^p n(i)$$

In this case, we have $c(256)=1$. Now let's take a look at the histogram we just build. For display purpose the histogram
 is scaled slightly differently, but the CDF is normalized with the total pixel number. The vertical axis has a range of [0, 1],
 while the horizontal axis has a range of [0, 255]. Things looks just about right.

### Histogram Equalization

<div align="center">
<table>
<tr><td><img class="post exampleimg" src="/projects/imageprocjs/images/histL0.png"></td></tr>
<tr><td>Source Image</td></tr>
<tr><td><img class="post exampleimg" src="/projects/imageprocjs/images/histL1.png"></td></tr>
<tr><td>Histogram Equalized Image</td></tr>
</table>
</div>
