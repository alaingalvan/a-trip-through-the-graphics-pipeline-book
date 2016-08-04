# Primitive Assembly, Clip/Cull, Projection, and Viewport Transform

After the last post about texture samplers, we’re now back in the 3D frontend. We’re done with vertex shading, so now we can start actually rendering stuff, right? Well, not quite. You see, there’s a bunch still left to do before we actually start rasterizing primitives. So much so in fact that we’re not going to see any rasterization in this post – that’ll have to wait until next time.

### Primitive Assembly

When we left the vertex pipeline, we had just gotten a block of shaded vertices back from the shader units, with the implicit promise that this block contains an integral number of primitives – i.e., we don’t allow triangles, lines or patches to be split across multiple blocks. This is important, because it means we can truly treat each block independently and never need to buffer more than one block of shader output – we can, of course, but we don’t have to.

The next step is to assemble all the vertices belonging to a single primitive (hence “primitive assembly”). If that primitive happens to be a point, this just reads exactly one vertex and passes it on. If it’s lines, it reads two vertices. If it’s triangles, three. And so on for patches with larger numbers of control points.

In short, all that happens here is that we gather vertices. We can either do this by reading the original index buffer and keeping a copy of our vertex index->cache position map around (as I described), or we can store the indices for the fully expanded primitives along with the shaded vertex data, which might take a bit more space for the output buffer but means we don’t have to read the indices again here. Either way works fine.

And now we have expanded out all the vertices that make up a primitive. In other words, we now have complete triangles, not just a bunch of vertices. So can we rasterize them already? Not quite.

### Viewport culling and clipping

Oh yeah, that. Yeah, I guess we’d better do that first, huh? This is one part of pipeline that really does exactly what you’d expect, pretty much the way you would expect it too (i.e. the way it’s explained in the docs). So I’m not gonna explain polygon clipping in general here, you can look that up in any computer graphics textbook, although most make a terrible mess of it; if you want a good explanation, use Jim Blinn’s (chapter 13 of [this book](http://www.amazon.com/Jim-Blinns-Corner-Graphics-Pipeline/dp/1558603875)), although you probably want to pass on his alternative [0,w] clip space these days, to avoid confusion if nothing else.

Anyway, clipping. The short version is this: Your vertex shader returns vertex positions on homogeneous clip space. Clip space is chosen to make the equations that describe the view frustum as simple as possible; in the case of D3D, they are $$-w \le x \le w$$, $$-w \le y \le w$$, $$0 \le z \le w$$, and $$0 < w$$; note that all the last equation really does is exclude the homogeneous point (0,0,0,0), which is something of a degenerate case.

We first need to find out if the triangle is partially or even completely outside any of these clip planes. This can be done very efficiently using [Cohen-Sutherland](http://en.wikipedia.org/wiki/Cohen%E2%80%93Sutherland)-style out-codes. You compute the clip out-code (or just clip-code) for each vertex (this can be done at vertex shading time and stored along with the positions, for example). Then, for each primitive, the bitwise AND of the clip-codes will tell you all the view-frustum planes that _all_ vertices in the primitive are on the wrong side of (if there’s any, that means the primitive is completely outside the view frustum and can be thrown away), and the bitwise OR of the clip-codes will tell you the planes that you need to clip the primitive against. Given the clipcodes, all this is just a few gates worth of hardware – simple stuff.

Additionally, the shaders can also generate a set of “cull distances” (a triangle will be discarded if any one cull distance for all vertices is less than zero), and a set of “clip distances” (which define additional clipping planes). These get considered for primitive rejection/clip testing too.

The actual clipping process, if invoked, can take one of two forms: we can either use an actual polygon clipping algorithm (which adds extra vertices and triangles), or we can add the clipping planes as extra edge equations to the rasterizer (if that sounds like gibberish to you, wait until the next part where I explain rasterization – it’ll ask make sense eventually). The latter is more elegant and doesn’t require an actual polygon clipper at all, but we need to be able to handle all normalized 32-bit floating point values as valid vertex coordinates; there might be a trick for building a fast HW rasterizer that does this, but it seems tricky to say the least. So I’m assuming there’s an actual clipper, with all that involves (generation of extra triangles etc). This is a nuisance, but it’s also very infrequent (more so than you think, I’ll get to that in a second), so it’s not a big deal. Not sure if that’s special hardware either, or if that path grabs a shader unit to do the actual clipping; depends on whether dispatching a new vertex shading load at this stage is awkward or not, how big a dedicated clipping unit is, and how many of them you need. I don’t know the answer to these questions, but at least from the performance side of things, it doesn’t much matter: we don’t really clip that often. That’s because we can use guard-band clipping.

### Guard-band clipping

The name is something of a misnomer; it’s not a fancy way of doing clipping. In fact, it’s quite the opposite: a straight-forward way of not doing clipping. :)

The underlying idea is very simple: Most primitives that are partially outside the left, right, top and bottom clip planes don’t need to be clipped at all. Triangle rasterization on GPUs works by, in effect, scanning over the full screen area (or more precisely, the scissor rect) and asking for every pixel: “is this pixel covered by the current triangle?” (In reality it’s a bit more complicated and way more efficient than that, but that’s the general idea). And that works just as well for triangles completely within the viewport as it does for triangles that extend past, say, the right and top clipping planes. As long as our triangle coverage test is reliable, we don’t need to clip against the left, right, top and bottom planes at all!

That test is usually done in integer arithmetic with some fixed precision. And eventually, as you move say one triangle vertex further and further out, you’ll get integer overflows and wrong test results. I think we can all agree that the rasterizer producing pixels that aren’t actually inside the triangle is, at the very least, extremely offensive behavior and should be illegal! Which it in fact is – hardware that does this is in violation of the spec.

There’s two solutions for this problem: The first is to make sure that your triangle tests never, ever generate the wrong results, no matter how your input triangle looks. If you manage that, then you don’t ever need to clip against the aforementioned four planes. This is called “infinite guard-band” because, well, the guard-band is effectively infinite. Solution two is to clip triangles eventually, just as they’re about to go outside the safe range where the rasterizer calculations can’t overflow. For example, say that your rasterizer has enough internal bits to deal with integer triangle coordinates that have $$-32768 \le X \le 32767$$, $$-32768 \le Y \le 32767$$ (note I’m using capital X and Y to denote screen-space positions; I’ll stick with this convention). You still do your viewport cull test (i.e. “is this triangle outside the view frustum”) with the regular view planes, but only actually clip against the guard-band clip planes which are chosen so that after the projection and viewport transforms, the resulting coordinates are in the safe range. I guess it’s time for an image:

<div data-shortcode="caption" id="attachment_309" style="width: 500px" class="wp-caption aligncenter">[![Guard-band clipping illustration](https://fgiesen.files.wordpress.com/2011/07/guardband_clip.png?w=497 "Guard-band clipping")](https://fgiesen.files.wordpress.com/2011/07/guardband_clip.png)

Guard-band clipping

</div>

The small white rectangle with blue outline that’s roughly in the middle represents our viewport, while the big salmon-colored area around it is our guard band. It looks like a small viewport in this image, but I actually picked a huge one so you can see anything! With our -32768 .. 32767 guard-band clip range, that viewport would be about 5500 pixels wide – yes, that’s some huge triangles right there :). Anyway, the triangles show off some of the important cases. The yellow triangle is the most common case – a triangle that extends outside the viewport but not the guard band. This just gets passed straight through, no further processing necessary. The green triangle is fully within the guard band, but outside the viewport region, so it would never get here – it’s been rejected above by the viewport cull. The blue triangle extends outside the guard-band clip region and would need to be clipped, but again it’s fully outside the viewport region and gets rejected by the viewport cull. Finally, the purple triangle extends both inside the viewport and outside the guard band, and so actually needs to be clipped.

As you can see, the kinds of triangles you need to actually have to clip against the four side planes are pretty extreme. As said, it’s infrequent – don’t worry about it.

### Aside: Getting clipping right

None of this should be terribly surprising; nor should it sound too difficult, at least if you’re familiar with the algorithms. But the devil’s in the details, always. Here’s some of the non-obvious rules the triangle clipper has to obey in practice. If it ever breaks _any_ of these rules, there’s cases where it will produce cracks between adjacent triangles that share an edge. This isn’t allowed.

*   Vertex positions that are inside the view frustum must be preserved, bit-exact, by the clipper.
*   Clipping an edge AB against a plane must produce the same results, bit-exact, as clipping the edge BA (orientation reversed) against that plane. (This can be ensured by either making the math completely symmetric, or always clipping an edge in the same direction, say from the outside in).
*   Primitives that are clipped against multiple planes must always clip against planes in the same order. (Either that or clip against all planes at once)
*   If you use a guard band, you _must_ clip against the guard band planes; you can’t use a guard band for some triangles but then clip against the original viewport planes if you actually need to clip. Again, failing to do this will cause cracks – and if I remember correctly there was actually a piece of graphics hardware in the bad old days that shipped with this bug enshrined in silicon. Oops. :)

### Those pesky near and far planes

Okay, so we have a really nice quick solution for the 4 side planes, but what about near and far? Particularly the near plane is bothersome, since with all the stuff that’s only slightly outside the viewport handled, that’s the plane we do most of our clipping for. So what can we do? A z guard band? But how would that work – we’re not actually rasterizing along the z axis at all! In fact, it’s just some value we interpolate over the triangle, damn!

On the plus side, though, it’s just some value we interpolate over the triangle. And in fact the z-near test ($$Z < 0$$) is _really easy_ to do once you interpolate Z – it’s just the sign bit. z-far ($$Z > 1$$) is an extra compare though (not I’m using Z not z here, i.e. these are “screen” or post-projection coordinates). But still, we’re doing Z-compares per pixel anyway (Z test!), so it’s not a big extra expense. It depends, but doing z-clip this way is definitely an option. And you need to be able to skip z-near/z-far clipping if you want to support things like NVidias ‘depth clamp’ OpenGL extension; in fact, I would argue the existence of that extension is a pretty good hint that they’re doing this, or at least used to for a while.

So we’re down to one of the regular clip planes: $$0 < w$$. Can we get rid of this one too? The answer is yes, with a rasterization algorithm that works in homogeneous coordinates, e.g. [this one](http://www.cs.unc.edu/~olano/papers/2dh-tri/). I’m not sure whether hardware uses that one though. It’s nice an elegant, but it seems like it would be hard to obey the (very strict!) D3D11 rasterization rules to the letter using that algorithm. But maybe there’s some cool tricks that I’m not aware of. Anyway, that’s about it with clipping.

### Projection and viewport transform

Projection just takes the x, y and z coordinates and divides them by w (unless you’re using a homogeneous rasterizer which doesn’t actually project – but I’ll ignore that possibility in the following). This gives us normalized device coordinates, or NDCs, between -1 and 1\. We then apply the viewport transform which maps the projected x and y to pixel coordinates (which I’ll call X and Y) and the projected z into the range [0,1] (I’ll call this value Z), such that at the z-near plane Z=0 and at the z-far plane Z=1.

At this point, we also snap pixels to fractional coordinates on the sub-pixel grid. As of D3D11, hardware is required to have exactly 8 bits of subpixel precision for triangle coordinates. This snapping turns some _very_ thin slivers (which would otherwise cause problems) into degenerate triangles (which don’t need to be rendered at all).

### Back-face and other triangle culling

Once we have X and Y for all vertices, we can calculate the signed triangle area using a cross product of the edge vectors. If the area is negative, the triangle is wound counter-clockwise (here, negative areas correspond to counter-clockwise because we’re now in the pixel coordinate space, and in D3D pixel space y increases downwards not upwards, so signs are inverted). If the area is positive, it’s wound clockwise. If it’s zero, it’s degenerate and doesn’t cover any pixels, so it can be safely culled. At this point, we know the triangle orientation so we can do back-face culling (if enabled).

And that’s it! We’re now ready for rasterization… almost. Actually we have to do triangle setup first. But doing that requires some knowledge of how rasterization will be performed, so I’ll put that off until the next part… see you then!

### Final remarks

Again, I skipped some parts and simplified others, so here’s the usual reminder that things are a bit more complicated in reality: For example, I pretended that you just use the regular homogeneous clipping algorithm. Mostly, you do – but you can have some vertex shader attributes flagged as using screen-space linear instead of perspective-correct interpolation. Now, the regular homogeneous clip always does perspective-correct interpolation; in the case of screen-space linear attributes, you actually need to do some extra work to make it not perspective-correct. :)

I talk about primitives some of the time, but mostly I’m just focusing on triangles here. Points and lines aren’t hard, but let’s be honest, they’re not what we’re here for either. You can work out the details if you’re interested. :)

There’s tons of rasterization algorithms out there, some of which (like Olanos 2DH method that I cited) allow you to skip nearly all clipping, but as I mentioned, D3D11 has very strict requirements on the triangle rasterizer so there’s not much wiggle room for HW implementations; I’m not sure if those methods can be tweaked to exactly follow the spec (there’s a lot of subtle points that I’ll cover next time). So here and in the following I’m assuming you can’t do the ultra-sleek thing; then again, the not-quite-so-sleek approaches I’m running with have slightly less math per pixel in the rasterizer, so they might win for HW implementations anyway. And of course I might be missing the magic pixie dust right around the corner that solves all of these problems. That occurs surprisingly often in graphics. If you know an awesome solution, give me a shout in the comments!

Lastly, the triangle culling I’m describing here is the bare minimum; for example, the class of triangles that will generate zero pixels upon rasterization is much larger than just zero-area tris, and if you can find it out quickly enough (or with few enough gates), you can drop the triangle immediately and don’t need to go through triangle setup. This is the last point where you can cull cheaply before going through triangle setup and at least some rasterization – finding other ways to early-reject tris pays off handsomely here.

