---
layout: post
title:  "JavaScript Image Processing (1) - Getting Started"
date:   2014-01-04 23:45:00
uid: 3
categories: Image Processing
---

Intro
-----
Over the last semester, I had worked on several image processing projects in my digital image class. Though I was familiar with many of the concepts and techniques taught in the class, I did managed to learn something new -- I learned to code in JavaScript through doing the homework projects. 

I have to admit it is fun to use JavaScript, simply because it is a much purer coding experience. The best part is to get away from compiling, since I am so fed up with the tedious compile-run-debug loop after more than 5 years of C++ coding.

To be honest, I was a little worried about the performance of JavaScript -- I have heard about it before that JavaScript runs tremendously slow. I was not sure if I would be satisfied with the execution speed of my applications, but I decided the best way to find out is to plumb into writing one project or two. It turns out I use *ONLY* JavaScript for all 8 class projects, even with one involving the solution of linear systems of over 100k unknowns. The performance of modern JavaScript engine is quite impressive!

Anyway, here I am, wrapping up what I have done over the past few months. 

Let's get started.

Manipulating Image with JavaScript
-----
First thing first, we *need* to have image to work on before we can actually develop our fancy image processing applications. But how? How do we get the image data? and how do you store the data and manipulate the data? Well, let's first figure out what we really need to have:

1. An image class. I can't see a reason not to have this.
2. A color/pixel class. This will greatly ease up the work, since we will be spending most of the time playing with pixels.
3. Image loader. We will also need to have a systematic way of loading images into our webapp.

That's it. That's all we need to get started.

Now what? We have already figured out our recipe, why don't we start cooking?

Let's first build up the color/pixel class:
{% highlight javascript %}
function Color(r, g, b, a)
{
    this.r = r; this.g = g; this.b = b; this.a = a;
}

Color.prototype.setColor = function(that)
{
    if( that != null && that.constructor === Color )
    {
        this.r = that.r; this.g = that.g; this.b = that.b; this.a = that.a;
        return this;
    }
    else
        return null;
};
{% endhighlight %}
These two are constructors. Basic stuff.

Next, we add the most frequently used functions to the set, the arithmetic operators.
{% highlight javascript %}
Color.prototype.equal = function( that ) {
    return (this.r == that.r && this.g == that.g && this.b == that.b);
};

Color.prototype.add = function(that) {
    return new Color(this.r + that.r, this.g + that.g, this.b + that.b, this.a + that.a);
};

// only r, g, b channels are modified
Color.prototype.addc = function(that) {
    return new Color(this.r + that.r, this.g + that.g, this.b + that.b, this.a);
};

Color.prototype.sub = function(that) {
    return new Color(this.r - that.r, this.g - that.g, this.b - that.b, this.a - that.a);
};

// only r, g, b channels are modified
Color.prototype.subc = function(that) {
    return new Color(this.r - that.r, this.g - that.g, this.b - that.b, this.a);
};

Color.prototype.mul = function(c)
{
    return new Color(this.r * c, this.g * c, this.b * c, this.a * c);
};

// only r, g, b channels are modified
Color.prototype.mulc = function(c) {
    return new Color(this.r * c, this.g * c, this.b * c, this.a);
};

Color.prototype.div = function(c) {
    var invC = 1.0 / c;
    return this.mul(invC);
};

Color.prototype.divc = function(c) {
    var invC = 1.0 / c;
    return this.mulc(invC);
};

{% endhighlight %}

We will also use some other utility functions:
{% highlight javascript %}
Color.prototype.normalize = function() {
    var invA = 1.0 / this.a;
    this.r *= invA;
    this.g *= invA;
    this.b *= invA;

    return this;
};

Color.prototype.clamp = function() {
    this.r = clamp(this.r, 0, 255);
    this.g = clamp(this.g, 0, 255);
    this.b = clamp(this.b, 0, 255);
    this.a = clamp(this.a, 0, 255);
    return this;
};

Color.prototype.round = function() {
    this.r = Math.round(this.r);
    this.g = Math.round(this.g);
    this.b = Math.round(this.b);
    this.a = Math.round(this.a);
    return this;
};

/* Euclidean distance between 2 colors */
Color.prototype.distance = function( that ) {
    var dr = this.r - that.r;
    var dg = this.g - that.g;
    var db = this.b - that.b;

    return (dr * dr + dg * dg + db * db);
};

{% endhighlight %}

The last thing is several special constructors, including interpolation, random color and frequently used colors:
{% highlight javascript %}

Color.interpolate = function(c1, c2, t)
{
    return c1.mul(t).add(c2.mul(1-t));
};

Color.rand = function() {
    return new Color(
        Math.random() * 255,
        Math.random() * 255,
        Math.random() * 255,
        Math.random() * 255
    );
};

Color.zero = function() {
    return new Color(0, 0, 0, 0);
};

Color.RED = new Color(255, 0, 0, 255);
Color.GREEN = new Color(0, 255, 0, 255);
Color.BLUE = new Color(0, 0, 255, 255);
Color.YELLOW = new Color(255, 255, 0, 255);
Color.PURPLE = new Color(255, 0, 255, 255);
Color.CYAN = new Color(0, 255, 255, 255);
Color.WHITE = new Color(255, 255, 255, 255);
Color.BLACK = new Color(0, 0, 0, 255);
Color.GRAY = new Color(128, 128, 128, 255);
{% endhighlight %}

What's next? The image class! We'll get to that in the [next post]
({% post_url 2014-01-05-ImageProcJS2 %}).

