---
layout: post
title: A guide to glTexImage2D - type and format
tags: [opengl]
---

`glTexImage2D` is used to transfer pixel data a two-dimensional texture image.
Unfortunately, the 
[OpenGL reference pages](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glTexImage2D.xhtml)
are incomplete regarding the interplay
of the parameters `internalFormat`, `format`, `type` and `data`.
This article is a user reference that summarizes
section 8.4 of the
[OpenGL 4.6 Core Profile](https://www.khronos.org/registry/OpenGL/specs/gl/glspec46.core.pdf)<sup>1</sup>
("the specification" from now on)
and the following articles from Khronos' OpenGL wiki:
[Pixel Transfer](https://www.khronos.org/opengl/wiki/Pixel_Transfer),
[Image Format](https://www.khronos.org/opengl/wiki/Image_Format),
and shows some examples of using `glTexImage2D`<sup>2</sup>.

[1] OpenGL `>= 3.0` is OK for most of the article.

[2] Note that the specification of `TexImage3D` is most often referenced
  in this article, as
  "for the pusposes of decoding the texture image,
  **TexImage2D** is equivalent to calling **TexImage3D**
  with corresponding arguments and _depth_ of 1,
  except that `UNPACK_SKIP_IMAGES` is ignored." 
  (p. 216 of the specfication).

* toc
{:toc}

## Introduction

```cpp
glTexImage2D(GLenum target,
             GLint level,
             GLint internalFormat,
             GLsizei width,
             GLsizei height,
             GLint border,
             GLenum format,
             GLenum type,
             const void* data);
```

`glTexImage2D` transfers user pixel data to OpenGL textures.
The specfication divides this transfer operation into three steps:
_Unpack_, _Convert of Float_ and _Expansion_.

It's the user's job to properly specify how OpenGL will unpack 
the pixel data.
More specifically,
the values of the pixel data are grouped into sets (groups/elements).
Each set corresponds to one pixel of the image
and each value of a group is associated with a channel
(red, etc.).

Afterwards, OpenGL will convert the data to the internal format
(more on that later).
If the target buffer is the color buffer,
each set is expanded to a set of four values
by setting missing color values to `0.0`
and missing alpha values to `1.0`.

The reference pages state that `data`
"Specifies a pointer to the image data in memory."
In fact,
the `data` parameter specifies a client memory pointer
(points to the pixel data array),
or, if a buffer object is bound to `GL_PIXEL_UNPACK_BUFFER`,
an offset into the bound buffer,
from which the data is read.

## Pixel data: format and type

The _pixel data type_ is specified by the `GLenum type` parameter.
Typical values are `GL_UNSIGNED_BYTE` (for `GLubyte`), `GL_UNSIGNED_INT` (for `GLuint`), etc.
`data` is treated as a sequence of unsigned bytes values, or 32bit unsigned integer values, etc.
A list of common types is found in Table 8.2 (p. 193) of the specfication.
There are some other accepted symbolic values,
such as `GL_UNSIGNED_SHORT_5_6_5`, etc.
These are pixel data types with _special interpretation_,
which are discussed in
[Pixel data types with special interpretation](#pixel-data-types-with-special-interpretation).

The `GLenum format` parameter
indicates what data the values of `data` represent.
Typical values are `GL_RGB`, `GL_RGBA`, `GL_STENCIL_INDEX`, etc.
According to the reference pages,
the "values [of `data`] are grouped into sets of one, two, three, or four values, depending on _format_, to form elements".
For example, `GL_RGB` would group the values of `data` into sets of three values,
`GL_RGBA` into sets of four.

But beyond the grouping of values to sets,
the pixel data format also specifies the _meaning_ and _order_ of the values,
and the _target buffer_ (color or depth/stencil).
For common pixel data formats, this information is found in Table 8.3 (p. 194) of the specfication.

_Meaning_ refers to what channel or index each value is mapped. 
Candidates are R, G, B, A, iR, iG, iB, iA, Depth (or D), Stencil Index (or S).
For example, the three values of each set of pixel data with format `GL_RGB` have the meanings R, G, B (in that order),
`GL_BGR` would instead yield the opposite order.

A prefixed _i_ stands for _integer_.
As the specfication states, 
"Not all combinations of `format` and `type` are valid." (p. 190).
For details,
see [Integral texture format](#integral-texture-format).

## Internal texture format

According to the reference pages, 
the `internalFormat` parameter "specifies the number of color components in the texture"
(or depth and stencil components).
The internal format also specifies the number of bits used for storage.

In fact,
the _base internal format_ of the target texture is derived from `internalFormal`
(one of the following):
* `GL_DEPTH_COMPONENT`
* `GL_DEPTH_STENCIL`
* `GL_RED`
* `GL_RG`
* `GL_RGB`
* `GL_RGBA`
* `GL_STENCIL_INDEX`

The _base internal format_ specifies
how the components of the sets of values are mapped to _internal components_:
_D_, _S_, _R_, _G_, _B_, _A_ 
(which stand for the internal depth, stencil, green, blue and alpha components).
Nevertheless, table 8.11 (p. 205) of the specfication lists these mappings.
In other words, the base internal format
specifies the number and type of components in the texture.

All base internal formats are valid values for `interalFormat`,
but not all combinations of format and internal format are valid:

> Textures with a base internal format of `DEPTH_COMPONENT` or `DEPTH_STENCIL`
> require either depth component data or depth/stencil component data.
> Textures with other base internal formats require RGBA component data.
> [p. 205]


The `internalFormat` parameter may be specified as _sized internal format_,
for example `GL_RGBA32` or `GL_DEPTH24_STENCIL8`,
or as _generic/specific compressed internal format_ `GL_COMPRESSED_*`,
for example `GL_COMPRESSED_RGBA` (generic)
or `GL_COMPRESSED_RGB8_PUNCHTHROUGH_ALPHA1_ETC2` (specific).
Internal sized format enumerants obey the following syntax:
```
GL_[components][size][type]
```
For details, see
[Color formats](https://www.khronos.org/opengl/wiki/Image_Format#Color_formats).

Not all formats are necessarily supported by every
implementation.
Every sized internal format or generic/specific compressed internal format
has a base internal format
(`GL_RGBA` for `GL_RGBA32`, `GL_DEPTH_STENCIL` for `GL_DEPTH24_STENICL8`).<sup>1</sup>
They may also be found in tables 8.12-8.14 of the specfication.
{:.note}

Sized internal formats also define the _internal component resolution_,
"the number of bits allocated to each value in a texture image" 
(p. 206 of the specfication).
Tables 8.12-8.13 of the specification
show the internal component resolution
based on sized internal format.
If, instead of a sized internal format,
only a base internal format is specified,
OpenGL will use an internal component resolution of its own choosing,
the _effective internal format_,
which may also depend on `format` and `type` (p. 206).

It is good practice to try to specify a sized internal format instead of
just a base internal format wherever possible.
Otherwise, the effective internal format may be platform or hardware
dependent
(see [Image precision](https://www.khronos.org/opengl/wiki/Common_Mistakes#Image_precision)).
{:.note}

When using a generic/specific compressed internal format,
OpenGL will try to compress the texture data on GPU memory.
This may or may not work.
In fact, according to p. 206 of the specfication,

> Generic compressed internal formats are never used directly as the
> internal formats of texture images. If _internalformat_ is one of the
> six generic compressed internal formats, its value is replaced by the
> symbolic constant for a specific compressed internal format of the
> GL’s choosing with the same base internal format. If no specific
> compressed format is available, _internalformat_ is instead replaced by
> the corresponding base internal format. If _internalformat_ is given as or
> mapped to a specific compressed internal format, but the GL can not
> support images compressed in the chosen internal format for any reason
> (e.g., the compression format might not support 3D textures),
> _internalformat_ is replaced by the corresponding base internal format and
> the texture image will not be compressed by the GL.

The internal component resolution varies with the type of compression.
Details are omitted.

[1] "If a sized internal format is specified, the mapping of the R, G,
  B, A, depth, and stencil values to texture components is equivalent to
  the mapping of the corresponding base internal format’s components
  [...]" (p. 206),
  "If a compressed internal format is specified, the mapping of the R,
  G, B, and A values to texture components is equivalent to the mapping of
  the corresponding base internal format’s components [...]"
  (p. 210).

## Pixel data types with special interpretation

Some pixel data types are pixel data types _with special interpretation_,
or _packed pixel formats_,
for example `GL_UNSIGNED_BYTE_3_3_2`, `UNSIGNED_INT_8_8_8_8`, `UNSIGNED_INT_10_10_10_2`.
These, too, correspond to a OpenGL data type,
but specify `glTexImage2D` to reinterpret each value by splitting it 
into _multiple_ values according to a scheme
that can either be inferred from the type's name
or looked up in the tables 8.6-8.9 in the specfication.

For example,
if `type` is `GL_UNSIGNED_INT_8_8_8_8`,
then every value of `data` is split into four values
(one for each `8`).
The first value of each set is
defined by the eight _most significant bits_ of `GLuint`
(the leading `_8`).
The second value of each set is
defined by next eight bits, etc.
Assuming `format = GL_RGBA`, 
a value of `0xff000fff` will yield `255` (red), `0` (green), `15` (blue) and `255` (alpha) <sup>1</sup>.
In particular,
`GL_UNSIGNED_INT` and `GL_UNSIGNED_INT_8_8_8_8` are not the same thing.

In other words,
packed pixel formats split every value of `data` into multiple values,
which then form an element.
Therefore,
packed pixel formats are only compatible with pixel data formats with the same number of components.
The matching formats are found in table 8.5 of the specification.

The target buffer for packed pixel formats with only two components
is always depth/stencil (not red/green).
{:.note title="Beware!"}

`GL_UNSIGNED_INT_5_9_9_9_REV` is the only exception to the rules above,
as it may only be combined with `GL_RGB`.
The RGB values (each with nine bits) are not encoded linearly,
and the last component (the one with five bits) is used as exponent to restore
the linear
(p. 202 of the specification).
{:.note}

Every packed pixel format may be reversed by adding the postfix `_REV` to its name
_and_ reversing the numbers (from `UNSIGNED_INT_10_10_10_2` to `UNSIGNED_INT_2_10_10_10_REV`).
For example, 
assuming `type = GL_UNSIGNED_INT_8_8_8_8_REV` and `format = GL_RGBA`,
a value of `0xff000fff` will yield `255` (red), `15` (green), `0` (blue), `255` (alpha).
However,
the packet sizes remain in the order given by the packed pixel format's designator
(the first pack of `GL_UNSIGNED_INT_10F_11F_11F_REV` has size 10).

The placement of the most significant byte is architecture-specific!
In particular,
these _is_ a difference between `GL_UNSIGNED_INT_8_8_8_8` and `GL_UNSIGNED_BYTE`.
On big endian machines,
where the most significant byte  is that with the lowest address,
they are equivalent,
but on little endian machines (x86 machines, for instance),
you'll end up with sets in the reverse order.
For a solution, see
[Endian issues](https://www.khronos.org/opengl/wiki/Pixel_Transfer#Endian_issues).
{:.note title="Beware!"}

[1] During the conversion to floats (if applicable),
  these values are _scaled_ into the range `[0, 1]`.
  In particular,
  after conversion the _red_ value would be `1.0`.

## Required image formats

In OpenGL 3.0, the notion of _required image format_ in introduced.
Required image formats must be supported by every implementation of OpenGL.
The list of required image formats may be found in
[Required formats](https://www.khronos.org/opengl/wiki/Image_Format#Required_formats)
or section 8.5.1 or the specfication (p. 206).
Use to required image formats wherever possible for maximum portability!

Some formats are only required to be supported for renderbuffers.
For instance,
`GL_STENCIL_INDEX8`,
is a required image format in OpenGL 4.3 and higher
(renderbuffer only in 4.3, texture and renderbuffer in 4.4 and higher).
{:.note}

## Remarks on conversion to the internal format

As [Color formats](https://www.khronos.org/opengl/wiki/Image_Format#Color_formats) states,
`RGBA` colors may be stored in one of three types of format:
normalized integers, floating-point, or integral.
The _conversion to float_ step in pixel transfer
converts the pixel data to data of one of these types.

Unsigned _normalized integers_
(or _normalized fixed-point numbers_)
with bitdepth $$B$$ store an unsigned integer in the interval $$[0, 2^B)$$,
but are interpreted as floats by virtue of the mapping

$$
\begin{aligned}
  i &\mapsto \frac{i}{2^B - 1}
  .
\end{aligned}
$$

The advantage of normalized integers over run-of-the-mill floats
is that their precision remains constant across their range.
Signed normalized integers are treated similarly.
For details,  
see [Normalized Integer](https://www.khronos.org/opengl/wiki/Normalized_Integer).

The specification states how the data is converted:

> The selected groups are transferred to the GL as described in section 8.4.4 and then clamped to the representable range of the internal format as follows:
> * If the internal format of the texture is signed or unsigned integer, components are clamped to $$[−2^n−1, 2^n−1 − 1]$$ or $$[0, 2^n − 1]$$, respectively, where n is the number of bits per component.
> * For color component groups, if the internalformat of the texture is signed or unsigned normalized fixed-point:
> – If the type of the data is a floating-point type (as defined in table 8.2), it is clamped to $$[−1, 1]$$ or $$[0, 1]$$, respectively.
> – Otherwise, it is clamped to to [sic!] $$[−2^n−1, 2^n−1−1]$$ or $$[0, 2^n−1]$$, respectively, where $$n$$ is the number of bits in the normalized representation.
> * For depth component groups, the depth value is clamped to $$[0, 1]$$.
> * Stencil index values are masked by $$2^n − 1$$, where n is the number of stencil
> bits in the internal format resolution (see below).
> * Otherwise, values are not modified. [p. 204]

Note that this means that, for example,
when using pixel data type `GL_UNSIGNED_BYTE` and format `GL_RED` (a gray-scale image),
then a value of `255` specifies maximum intensity and `0` minimum intensity (black).

## Integer sized internal formats

Integer sized internal formats obey the syntax:
```
GL_[component][size][type],
```
where `type` is `I` or `UI`.
Such internal formats expect pixel data whose meaning is iR, iG, iB, iA
(see [Pixel data: Format and type](#pixel-data-format-and-type)),
in other words: those ending in `_INTEGER`.
They must not be combined with pixel data formats that use R, G, B, A, S or D channels:

> An `INVALID_OPERATION` error is generated if _format_ is one of the `INTEGER` component formats
> [...] and _type_ is one of the floating-point types [...]." [p. 190f]

Also note that sampling an integer texture with a `sampler2D` is undefined behavior.
Use `isampler2D` for signed integer textures
and `usampler2D` (not `uisampler2D`!) for unsigned integer textures:

> Fixed-point and floating-point textures return a floating-point value and integer textures return
> signed or unsigned integer values. The fragment shader is responsible for interpreting the result
> of a texture lookup as the correct data type, otherwise the result is undefined. [p. 178]

> So for samplers, floating-point samplers begin with "sampler". Signed integer samplers begin with "isampler", and unsigned integer samplers begin with "usampler". If you attempt to read from a sampler where the texture's Image Format doesn't match the sampler's basic format (usampler2D with a GL_R8I, or sampler1D with GL_R8UI, for example), all reads will produce undefined values. 
> [Sampler types](https://www.khronos.org/opengl/wiki/Sampler_(GLSL)#Sampler_types)

Using the wrong texture sampler object is undefined behavior
and may result in a black screen.
{:.note title="Beware!"}

## Examples

In all examples except [From buffer](#from-buffer),
the pixel data is always provided by an instance of `std::vector<T>`,
where `T` is `unsigned char, float`, etc.
For the sake of simplicity, 
we're using `GL_TEXTURE_2D` as target and mipmap level 0.
Most examples run the following shaders:

```glsl
// shader.vert
#version 330 core

layout (location = 0) in vec3 aPosition;
layout (location = 1) in vec2 aTexCoords;

out vec2 texCoords;

void main() {
  gl_Position = vec4(aPosition.xyz, 1.0);
  texCoords = aTexCoords;
}

// shader.frag
#version 330 core
out vec4 FragColor;
in vec2 texCoords;
uniform sampler2D sampler;

void main() {
  FragColor = texture(sampler, texCoords);
}
```

### Color data from unsigned char

To transfer pixel data of type `unsigned char`,
use `type = GL_UNSIGNED_BYTE`.
The `format` depends on the number and order of colors in `pixel`.
For most use-cases, it's `GL_RGB`
or `GL_RGBA`
(three or four values in `pixel` form a texel<sup>1</sup>).

```cpp
std::vector<unsigned char> pixel = {
  //R    G    B
  255,   0,   0,  // texel 1 (lower left) is red
  0,   255,   0,  // texel 2 is green
  0,     0, 255,  // texel 3 is blue
  /* ... */
};
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_RGBA,
             height, width, 0,
             GL_RGB,
             GL_UNSIGNED_BYTE,
             pixel.data());

std::vector<unsigned char> pixel = {
  //R    G    B    A
  255,   0,   0, 255 // texel 1 (lower left) is solid red
  0,   255,   0, 255 // texel 2 is solid green
  0,     0, 255, 255 // texel 3 is solid blue
  /* ... */
};
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_RGBA,
             height, width, 0,
             GL_RGBA,
             GL_UNSIGNED_BYTE,
             pixel.data());
```

Changing the `format` parameter to `GL_BGR` or `GL_BGRA` will result in swapped colors
(left-most texel is solid blue, etc.).
Other choices of `internalFormat` are possible,
`GL_RGBA32F`, for example
(recall that `32F` only refers to the format used to store the data in the texture).

### Gray-scale textrues with unsigned char

Depending on the internal format,
the pixel data format may also vary between color formats with fewer channels than the internal
format.
Note that this affects the number of values per texel in `pixel`.

```cpp
std::vector<unsigned char> pixel = {
  255, 127, 0,  // First pixel white, second gray, third black
  /* ... */
};
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_RED,
             height, width, 0,
             GL_RED,
             GL_UNSIGNED_BYTE,
             pixel.data());
```

Note that for rendering the actual gray-scale colors (instead of scales red-scale colors),
you require the following fragment shader:

```glsl
sampler2D sampler;
in texCoords;
out vec4 FragColor;

void main() {
  float red = texture(sampler, texCoords).r;
  vec3 grey = vec3(red);
  FragColor = vec4(grey, 1.0);
}
```

### Color data from float

To transfer pixel data of type `GLfloat`,
use `type = GL_FLOAT`.

```cpp
std::vector<GLfloat> pixel = {
  //R    G    B
  1.0,   0,   0,  // texel 1 (lower left) is red
  0,   1.0,   0,  // texel 2 is green
  0,     0, 1.0,  // texel 3 is blue
  /* ... */
};
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_RGBA,
             height, width, 0,
             GL_RGB,
             GL_FLOAT,
             pixel.data());
```

(Similar for other formats and gray-scale.)

### Interior integer format from unsigned char

To user an interior integer format, 

```cpp
std::vector<GLubyte> pixel = {
  255,   0,   0, 255 // texel 1 (lower left) is solid red
  0,   255,   0, 255 // texel 2 is solid green
  0,     0, 255, 255 // texel 3 is solid blue
  /* ... */
};
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_RGBA32I,
             height, width, 0,
             GL_RGBA_INTEGER,
             GL_UNSIGNED_BYTE,
             pixel.data());
```

Using an invalid combination of internal format and pixel data format
will fail spectacularly:
For instance, the pixel data sets describe iR, iG, iB, iA (`GL_RGBA_INTEGER`),
while the internal format would expect R, G, B, A.
Note that the pixel data format is somewhat restricted.
For example, using `GL_FLOAT` will provoke a `GL_INVALID_OPERATION` error.

To run this example, recall that the fragment shader must use a sampler
of type `isampler2D`:

```glsl
isampler2D sampler;
in texCoords;
out vec4 FragColor;

void main() {
  FragColor = texture(sampler, texCoords);
}
```


### Depth textures

To transfer to the depth component,
use `format = GL_DEPTH_COMPONENT`,
and the base internal format `GL_DEPTH_COMPONENT`.
Any of the following sized internal depth format will work:
* `GL_DEPTH_COMPONENT_16`
* `GL_DEPTH_COMPONENT_24`
* `GL_DEPTH_COMPONENT_32`
* `GL_DEPTH_COMPONENT_32F`

Note that not all of these may be required formats.

```cpp
std::vector<GLfloat> pixel = {
  0.0, 0.5, 1.0 // Depth value of texels 1, 2, 3
  /* ... */
};
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_DEPTH_COMPONENT,
             height, width, 0,
             GL_DEPTH_COMPONENT,
             GL_UNSIGNED_FLOAT,
             pixel.data())
```

(Similar for other types.)

After sampling,
the depth values may be rendered as gray-scales:

```glsl
sampler2D sampler;
in texCoords;
out vec4 FragColor;

void main() {
  float red = texture(sampler, texCoords).r;
  vec3 gray = vec3(red);
  FragColor = vec4(gray, 1.0);
}
```

Thus, the first three texels will be black, gray, white.

### Depth/stencil textures

There are only two internal sized formats for depth/stencil textures,
`GL_DEPTH24_STENCIL8` and `GL_DEPTH32F_STENCIL8`.
Using the internal format `GL_DEPTH_STENCIL` will result in selection 
of either of these as effective texture format.

The pixel data format must be `GL_DEPTH_STENCIL`
and the only available pixel data types are `GL_UNSIGNED_INT_24_8`
and `GL_FLOAT_32_UNSIGNED_INT_24_8_REV`.
Thus, every value of `pixel` is split into two values,
made from the six most significant and two least significant bytes,
and specify depth and stencil index, resp.

```cpp
std::vector<GLuint> pixel = {
//  DDDDDDSS
  0x000000ff,  // texel 1, depth 0.0, stencil 255
  0x82eb2100,  // texel 2, depth 0.5, stencil 0
  0xffffffff,  // texel 3, depth 1.0, stencil 255
  /* ... */
};
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_DEPTH_STENCIL,  // GL_DEPTH24_STENCIL8, GL_DEPTH32F_STENCIL8
             height, width, 0,
             GL_DEPTH_STENCIL,
             GL_UNSIGNED_INT_24_8,
             pixel.data());
```

Rendering the resulting texture with the shader
from [Depth textures](depth-textures) will show the depth levels in gray-scale.
To verify the stencil values in a straightforward manner,
use [stencil texturing](https://www.khronos.org/opengl/wiki/Texture#Stencil_texturing)
in OpenGL `>= 4.3`.
In lower version,
render to a framebuffer with the depth/stencil texture attached,
then render the framebuffer's texture to the screen.

### Stencil textures

Available internal sized formats for stencil textures are
`GL_STENCIL_INDEX#`, where `#` is `1, 4, 8, 16`.
Khronos dissuades using any other than `8`,
most likely as `GL_STENCIL_INDEX8` is the only required texture format (as of OpenGL 4.4).

```cpp
std::vector<GLuint> pixel = {0, 1, /* ... */};
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_STENCIL_INDEX,  // GL_STENCIL_INDEX8
             height, width, 0,
             GL_STENCIL_INDEX,
             GL_UNSIGNED_BYTE,
             pixel.data());
```
Using `GL_UNSIGNED_BYTE` in combination with `GL_STENCIL_INDEX8`
is the only fathomable choice (and `GL_BYTE`).
Other data types (larger than 8-bit) will be masked with `0xff`,
effectively converting them to `GL_UNSIGNED_BYTE`.

Note that if `GL_STENCIL_INDEX` is used as internal format,
then the bitmask is a function of the choice of effective internal format of OpenGL's choosing.
In particular,
the interpretation of the pixel data may vary from system to system depending on the 
available sized internal formats.
This example demonstrates why it's good to specify the sized internal format,
rather than leave its choice to chance.
In other words: _Always use `GL_STENCIL_INDEX8`_.

Note that OpenGL 4.4 or `ARB_texture_stencil8` is required to use stencil formats
(see [Stencil only](https://www.khronos.org/opengl/wiki/Image_Format#Stencil_only)).
For details on sampling stencil textures, see
[Depth/stencil textures](#depthstencil-textures).

### Packed pixel formats

```cpp
std::vector<GLuint> pixel = {0x00ff00ff, /* ... */};
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_RGBA,
             height, width, 0,
             GL_RGBA,
             GL_UNSIGNED_INT_8_8_8_8,
             pixel.data());
```

What's the color of the lower left pixel?
The format specifies that the R, G, B, A components are each given by
eight bits,
with the most significant bits associated with R, etc.
Thus, `R = 0`, `G = 255`, `B = 0`, `A = 255`.
Conversion to float yields `(0.0, 1.0, 0.0, 1.0)`, green.

```cpp
std::vector<GLuint> pixel = {0xffc00fff, /* ... */};
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_RGBA,
             height, width, 0,
             GL_RGBA,
             GL_UNSIGNED_INT_10_10_10_2,
             pixel.data());
```

`pixel[0]` is equal to
```
11111111 11000000 00001111 11111111
```
(ordered by decreasing significance).
The pixel data format specifies that
the R, G, B components are given by ten bits each,
and the A component by the final two bits.
Thus, `R = 1111111111` (maximum value),
`G = 0000000000` (minimum value),
`B = 1111111111` (maximum value),
`A = 11` (maximum value).
These are converted to `(1.0, 0.0, 1.0, 1.0)`,
purple.

### From buffer

The example shows how to read pixel data from a staging buffer
instead of client memory.

```cpp
GLuint buffer;
glGenBuffers(1, &buffer);
glBindBuffer(GL_PIXEL_UNPACK_BUFFER, buffer);

// For testing purposes, fill `buffer` with some data...
std::vector<GLfloat> pixel = {
  /* Pixel data. Take offset into account when allocating memory.
   * Otherwise you'll most likely end up with a black screen. 
   */
};
glBufferData(GL_PIXEL_UNPACK_BUFFER,
             sizeof(GLfloat) * (width * height + offset),
             pixel.data(),
             GL_STATIC_DRAW);

GLuint tex;
glGenTextures(1, &tex);
glBindTexture(GL_TEXTURE_2D, tex);
glTexImage2D(GL_TEXTURE_2D, 0,
             GL_RGBA,
             width, height, 0,
             GL_RGBA,
             GL_FLOAT,
             reinterpret_cast<void*>(offset * sizeof(GLfloat)));
// ...
```

If `offset  = 4*k`, then the first `k` pixels will be skipped.
