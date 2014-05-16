---
layout: post
title:  "JavaScript Image Processing (2) - The Image Class"
date:   2014-01-05 16:32:00
uid: 4
categories: Image Processing
---

The next essential thing we need for the image processing webapp is *the image class*. We need a handy class
to facilitate all kinds of operations we are going to apply on the images.

First and as always, the constructor. Nothing special, just specify the width and height, and maybe pass in the pixel map too.
{% highlight javascript %}
function RGBAImage( w, h, data )
{
	this.type = 'RGBAImage';
    this.w = w;
    this.h = h;
    this.data = new Uint8Array(w*h*4);	
    data && this.data.set(data);	
}
{% endhighlight %}

The most basic operation to alter an image is to do it pixel wise, and we need a way to access the pixels in the image.
Here we go, the getter and setter.
{% highlight javascript %}
// get a pixel from the image
RGBAImage.prototype.getPixel = function(x, y) {
    var idx = (y * this.w + x) * 4;
    return new Color(
        this.data[idx+0],
        this.data[idx+1],
        this.data[idx+2],
        this.data[idx+3]
    );
};

// set a pixel value in the image
RGBAImage.prototype.setPixel = function(x, y, c) {
    var idx = (y * this.w + x) * 4;
    this.data[idx] = c.r;
    this.data[idx+1] = c.g;
    this.data[idx+2] = c.b;
    this.data[idx+3] = c.a;
};
{% endhighlight %}

For many operations, accessing single pixel does not meet our requirement. We probably need to go to sub-pixel level.
Bilinear sampling allows us to obtain image data on subpixel level. Suppose we need to get the color information at position
$(x, y)$, where $x$ and $y$ are floating point numbers. We are not going to hit exactly on a single pixel, but somewhere among
4 adjacent pixels. Apparently the color we get should be a combination of the 4 neighboring pixels, and the color should be
determined by the spatial relationship between the position and the 4 pixels. The closer the position is to a pixel, the closer should
the color be to that pixel. This can be achieved with bilinear sampling - linearly interpolating the pixel value in both $x$ and $y$
directions. Suppose the 4 neighbor pixels are $(x\_0, y\_0)$, $(x\_1, y\_0)$, $(x\_0, y\_1)$ and $(x\_1, y\_1)$ (where$x\_1=x\_0+1,y\_1=y\_0+1$), the color at $(x, y)$ is given by:
$$c(x,y) = (1-r)(1-t)\times c(x\_0,y\_0) + r(1-t)\times c(x\_1,y\_0) + (1-r)t\times c(x\_0,y\_1) + rt\times c(x\_1,y\_1)$$
Here $r$ and $t$ are interpolation weights given by:
$$r = x - x\_0$$
and
$$t = y - y\_0$$


{% highlight javascript %}
// bilinear sample of the image
RGBAImage.prototype.sample = function(x, y) {
    var w = this.w, h = this.h;
    var y0 = Math.floor(y);
    var y1 = Math.ceil(y);

    var x0 = Math.floor(x);
    var x1 = Math.ceil(x);

    var fx = x - x0;
    var fy = y - y0;

    var c = this.getPixel(x0, y0).mul((1-fy) * (1-fx))
        .add(this.getPixel(x0, y1).mul(fy * (1-fx)))
        .add(this.getPixel(x1, y0).mul((1-fy) * fx))
        .add(this.getPixel(x1, y1).mul(fy * fx));

    c.clamp();

    return c;
};
{% endhighlight %}

With the functions we have so far, we are ready to perform per-pixel operation with the following functions. The first
one, *apply*, alters every single pixel in the image and stores the result back to the source image. The operation is
unspecified - it is passed in as an argument which produce a new color given an input color.
{% highlight javascript %}
RGBAImage.prototype.apply = function( f ) {
    for(var y=0;y<this.h;y++) {
        for(var x=0;x<this.w;x++) {
            this.setPixel(x, y, f(this.getPixel(x, y)));
        }
    }
    return this;
};
{% endhighlight %}

The *map* function is very similar except the result is stored in a new image. It is also optimized a little bit to improve
performance.
{% highlight javascript %}
RGBAImage.prototype.map = function( f ) {
    var w = this.w, h = this.h;
    var dst = new RGBAImage(w, h);
    var data = this.data;
	for(var y = 0,idx=0;y<this.h;++y) {
		for(var x=0;x<this.w;++x,++idx) {
            dst.setPixel(x, y, f(
                data[idx],
                data[++idx],
                data[++idx],
                data[++idx],
                x, y, w, h
            ));
		}
	}
	return dst;
};
{% endhighlight %}

Other utility functions are the resizing function and rendering related functions.
{% highlight javascript %}
// utility function
// resize image
RGBAImage.prototype.resize = function(w, h) {
    var iw = this.w, ih = this.h;
    // bilinear interpolation
    var dst = new RGBAImage(w, h);

    var ystep = 1.0 / (h-1);
    var xstep = 1.0 / (w-1);
    for(var i=0;i<h;i++) {
        var y = i * ystep;
        for(var j=0;j<w;j++) {
            var x = j * xstep;
            dst.setPixel(j, i, this.sample(x * (iw-1), y * (ih-1)));
        }
    }
    return dst;
};

RGBAImage.prototype.resize_longedge = function( L ) {
    var nw, nh;
    if( this.w > this.h && this.w > L ) {
        nw = L;
        nh = Math.round((L / this.w) * this.h);
        return this.resize(nw, nh);
    }
    else if( this.h > L ){
        nh = L;
        nw = Math.round((L / this.h) * this.w);
        return this.resize(nw, nh);
    }
    else return this;
};
{% endhighlight %}

{% highlight javascript %}
// for web-gl
RGBAImage.prototype.uploadTexture = function( ctx, texId )
{
    var w = this.w;
    var h = this.h;

    ctx.bindTexture(ctx.TEXTURE_2D, texId);
    ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_MIN_FILTER, ctx.NEAREST);
    ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_MAG_FILTER, ctx.NEAREST);
    ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_WRAP_S, ctx.CLAMP_TO_EDGE);
    ctx.texParameteri(ctx.TEXTURE_2D, ctx.TEXTURE_WRAP_T, ctx.CLAMP_TO_EDGE);
    ctx.texImage2D(ctx.TEXTURE_2D, 0,  ctx.RGBA, w, h, 0, ctx.RGBA, ctx.UNSIGNED_BYTE, this.data);
};

// for html canvas
RGBAImage.prototype.toImageData = function( ctx ) {
    var imgData = ctx.createImageData(this.w, this.h);
    imgData.data.set(this.data);
    return imgData;
};

/* render the image to the passed canvas */
RGBAImage.prototype.render = function( cvs ) {
	canvas.width = this.w;
	canvas.height = this.h;
	context.putImageData(this.toImageData(context), 0, 0);
};

/* get RGBA image data from the passed image object */
RGBAImage.fromImage = function( img, cvs ) {
    var w = img.width;
    var h = img.height;

    // resize the canvas for drawing
    cvs.width = w;
	cvs.height = h;
	var ctx = cvs.getContext('2d');

    // render the image to the canvas in order to obtain image data
    ctx.drawImage(img, 0, 0);
    var imgData = ctx.getImageData(0, 0, w, h);
    var newImage = new RGBAImage(w, h, imgData.data);
    imgData = null;

    // clear up the canvas
    ctx.clearRect(0, 0, w, h);
    return newImage;
};
{% endhighlight %}

With the image class established, we can now create [a simple webpage]
                                                    ({{site.url}}/projects/imageprocjs/demo_showimage.html) that loads an image from the server and displays
it.