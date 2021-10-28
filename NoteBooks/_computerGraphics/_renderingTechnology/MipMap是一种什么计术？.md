## 是什么？

Mipmap 又称多级渐远纹理，是一种解决单一纹理在屏幕上进行缩小、放大变化时，出现的采样率低、颜色信息丢失、走样、抖动闪烁现象的纹理优化技术。

## 技术原理

1. Mipmap 技术，是将纹理按照 2 的倍数，进行缩放，直到 1 * 1 的大小，生成一系列的图，然后把这些图储存起来。
2. 图形渲染管线会根据当前映射区域的大小与纹理坐标对应的纹理区域大小的不同，在这一系列 Mipmapped 状态的图中，选择最合适的纹理图，上传到显存。

## OpenGL 的实现

### 手动设置

用户可通过 `gl.texImage2D` api 为每一个级别设置纹理图数据。用户手动设置，要求提前准备好这一些列的纹理图。

```js
// WebGL2:
void gl.texImage2D(target, level, internalformat, width, height, border, format, type, GLintptr offset);
void gl.texImage2D(target, level, internalformat, width, height, border, format, type, HTMLCanvasElement source);
void gl.texImage2D(target, level, internalformat, width, height, border, format, type, HTMLImageElement source);
void gl.texImage2D(target, level, internalformat, width, height, border, format, type, HTMLVideoElement source);
void gl.texImage2D(target, level, internalformat, width, height, border, format, type, ImageBitmap source);
void gl.texImage2D(target, level, internalformat, width, height, border, format, type, ImageData source);
void gl.texImage2D(target, level, internalformat, width, height, border, format, type, ArrayBufferView srcData, srcOffset)
```

### 自动生成

用户只需要设置基础层(Base-level)纹理，之后调用 `gl.generateMipmap` 自动生成所有需要的多级渐远纹理。

```js
gl.texImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
gl.generateMipmap(GL_TEXTURE_2D);
```

### 整个过程

```c++
// 生成纹理对象，并绑定
unsigned int texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
// 为当前绑定的纹理对象设置环绕、过滤方式
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);   
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
// 加载并生成纹理
int width, height, nrChannels;
unsigned char *data = stbi_load("container.jpg", &width, &height, &nrChannels, 0);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
// 生成 Mipmap
glGenerateMipmap(GL_TEXTURE_2D);

// 生成了纹理和相应的多级渐远纹理后，释放图像的内存是一个很好的习惯。
stbi_image_free(data);
```

> **注意：**
>
> 在渲染中切换多级渐远纹理级别(Level)时，OpenGL在两个不同级别的多级渐远纹理层之间会产生不真实的生硬边界。就像普通的纹理过滤一样，切换多级渐远纹理级别时你也可以在两个不同多级渐远纹理级别之间使用NEAREST和LINEAR过滤。为了指定不同多级渐远纹理级别之间的过滤方式，你可以使用下面四个选项中的一个代替原有的过滤方式：
>
> | 过滤方式                  | 描述                                                         |
> | :------------------------ | :----------------------------------------------------------- |
> | GL_NEAREST_MIPMAP_NEAREST | 使用最邻近的多级渐远纹理来匹配像素大小，并使用邻近插值进行纹理采样 |
> | GL_LINEAR_MIPMAP_NEAREST  | 使用最邻近的多级渐远纹理级别，并使用线性插值进行采样         |
> | GL_NEAREST_MIPMAP_LINEAR  | 在两个最匹配像素大小的多级渐远纹理之间进行线性插值，使用邻近插值进行采样 |
> | GL_LINEAR_MIPMAP_LINEAR   | 在两个邻近的多级渐远纹理之间使用线性插值，并使用线性插值进行采样 |
>
> 就像纹理过滤一样，我们可以使用glTexParameteri将过滤方式设置为前面四种提到的方法之一：
>
> ```c++
> glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
> glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
> ```
>
> 一个常见的错误是，将放大过滤的选项设置为多级渐远纹理过滤选项之一。这样没有任何效果，因为多级渐远纹理主要是使用在纹理被缩小的情况下的：纹理放大不会使用多级渐远纹理，为放大过滤设置多级渐远纹理的选项会产生一个GL_INVALID_ENUM错误代码。

## Mipmap 的功能应用

### 一、加快渲染速度和减少图像锯齿，提高采样效率

因为从一张小纹理进行采样肯定会比从一张大纹理中采样要速度快，效果说不定还会比大纹理采样的结果好，因为生成mipmap的时候，我们可以通过glHins选择GL_NICEST的方式，这样小纹理不会特别的失真，（下面我们会再介绍glHints这个API）。然而如果使用大纹理再进行GL_NEAREST的话会更容易失真。当然有人可能说那生成mipmap是需要耗费资源的，其实大部分mipmap都是在离线生成好，然后在游戏不忙的时候将资源传入，生成纹理的，再不济也是会选择在游戏不忙的时候使用glGeneratemipmap生成，虽然一定程度上影响了应用程序的性能，但是减少了对GPU内存和带宽占用，且只需要generate一次，尽量不会让生成mipmap成为游戏性能的瓶颈。

### 二、显示小图片的时候其实是使用不一样的内容

这样的话就需要mipmap是通过glTexImage2D等API传入的，而非glGeneratemipmap生成的，这样的话开发者就可以控制texture的第i层mipmap的内容，比如可以让mipmap第0层显示人物的全身照片，而第1层显示人物的大头照。这样，把纹理绘制到绘制buffer上的时候，根据纹理坐标大小，导致同一张纹理绘制出来的结果不同。

## 结论

综上所述，Mipmap 是非常有用的，它只有一点不好，那就是如果离线生成 Mipmap 内容的话，会导致包体要稍微大一些，每张纹理大概会比原来多占用1/3的空间。



[LearnOpenGL - 纹理]: https://learnopengl-cn.github.io/01%20Getting%20started/06%20Textures/
[OpenGL ES2.0 知识串讲（9）-- OpenGL ES 详解 III（纹理）]: http://geekfaner.com/shineengine/blog10_OpenGLESv2_9.html#glTexParameter
[知乎：mipmap的优点具体是什么？]: https://www.zhihu.com/question/66993945
[OpenGL 之纹理映射 Mipmap]: https://www.jianshu.com/p/65376d181323
[贴图 filtering 与 MIP map 简介]: https://www.lihuasoft.net/article/show.php?id=4509

