---
layout:     post
title:      "《OpenGL ES 应用开发实践指南》读书笔记 No.7"
subtitle:   "Android OpenGL ES 从入门到奔溃"
date: 2016-07-22 11:00:01 +0800
author:     "Roger"
header-img: "img/opengl-bg.jpg"
tags:
    - OpenGL ES
---
Android OpenGL ES 第七章 - 用纹理增加细节
---

本系列所有源码地址：[https://github.com/Rogero0o/OpenGL_Demo](https://github.com/Rogero0o/OpenGL_Demo)

请大家务必对照源码阅读本文，否则有如盲人摸象。

上一章我们学习了如何完成三维的桌子，这一章我们将学习如何使用纹理增加细节。这一章的项目名为 AirHockeyTextured 。

### 理解纹理

OpenGL 中的纹理可以用来表示图像、照片、甚至由一个数学算法生成的分形数据。每个二维的纹理都由许多小的纹理元素组成。要使用纹理，最常用的方式是直接从一个图像文件加载数据。其中需要注意的是 **OpenGL要求纹理的高度和宽度都必须是2的n次方大小，只有满足这个条件，这个纹理图片才是有效的。**

### 加载纹理

我们的第一个任务就是把一个图像文件的数据加载到一个 OpenGL 的纹理中。在 util 包里新建一个名为 “ TextureHelper ” 的新类，这个方法的功能是将 Android 上下文和资源 ID 作为输入参数，并返回加载图像后的 OpenGL 纹理的ID。代码如下：

    private final static String TAG = "TextureHelper";

    public static int loadTexture(Context context,int resourceId){
      final int[] textureObjectIds = new int[1];
      GLES20.glGenTextures(1,textureObjectIds,0);
      if(textureObjectIds[0] == 0 ){
        if(LoggerConfig.ON){
          Log.w(TAG,"Could not generate a new OpenGL texture object.");
        }
        return 0;
      }
      final BitmapFactory.Options options = new BitmapFactory.Options();
      options.inScaled = false;

      final Bitmap bitmap = BitmapFactory.decodeResource(context.getResources(),resourceId,options);

      if(bitmap==null){
        if(LoggerConfig.ON){
          Log.w(TAG,"Resource ID "+resourceId+" could not be decoded");
        }
        GLES20.glDeleteTextures(1,textureObjectIds,0);
        return 0;
      }
      GLES20.glBindTexture(GLES20.GL_TEXTURE_2D,textureObjectIds[0]);
      GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D,GLES20.GL_TEXTURE_MIN_FILTER,GLES20.GL_LINEAR_MIPMAP_LINEAR);
      GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D,GLES20.GL_TEXTURE_MAG_FILTER,GLES20.GL_LINEAR);
      GLUtils.texImage2D(GLES20.GL_TEXTURE_2D,0,bitmap,0);
      GLES20.glGenerateMipmap(GLES20.GL_TEXTURE_2D);

      bitmap.recycle();

      GLES20.glBindTexture(GLES20.GL_TEXTURE_2D,0);
      return textureObjectIds[0];
    }

通过传递 1 作为第一个参数调用 *glGenTextures(1,textureObjectIds,0)* 我们就生成了一个纹理对象。 OpenGL 会把那个生成的 ID 存储在 textureObjectIds 中。我们也检查了该方法是否调用成功，如果返回结果不等于 0 则继续。

下一步是使用 Android 的 API 读入图像文件的数据。首先创建一个 *BitmapFactory.Options* 的实例，命名为 “ options ”，并且设置 inScaled 为 false。这个告诉 Android 我们想要原始数据，而不是这个图像的缩放版本。

接下来调用 *BitmapFactory.decodeResource()* 做实际的解码工作，这个调用会返回一个解码后的 bitmap。在可以使用这个新生成的纹理对象做任何其他事之前，我们需要告诉 OpenGL 后面的纹理调用应该应用于这个纹理对象。我们为此使用一个 *glBindTexture()* 调用。

### 理解纹理过滤

当纹理大小被扩大或者缩小时，我们需要使用纹理过滤明确说明发生了什么。当我们在渲染表面上绘制一个纹理时，那个纹理的纹理元素可能无法精确的映射到 OpenGL 生成的片段上。有两种情况，放大或缩小。针对每一种情况，我们都需要制定一个纹理过滤器。其中有几种过滤模式，具体请参考链接: [Link](http://blog.csdn.net/pizi0475/article/details/49740879)

显示在代码中，设置纹理过滤的代码为：

    GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D,GLES20.GL_TEXTURE_MIN_FILTER,GLES20.GL_LINEAR_MIPMAP_LINEAR);
    GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D,GLES20.GL_TEXTURE_MAG_FILTER,GLES20.GL_LINEAR);

我们使用 *glTexParameteri* 来设置每个过滤器，*GL_TEXTURE_MIN_FILTER* 代表缩小的情况，*GL_TEXTURE_MAG_FILTER* 代表放大的情况。对于缩小的情况，我们选择 *GL_LINEAR_MIPMAP_LINEAR* ，告诉 OpenGL 使用三线性过滤，对于放大的情况，我们设置放大器为 *GL_LINEAR* ，告诉 OpenGL 使用双线性过滤。

接下来我们调用 *GLUtils.texImage2D(GLES20.GL_TEXTURE_2D,0,bitmap,0);* 来加载位图到 OpenGL 中，并把它复制到当前绑定的纹理对象。下一步调用 *GLES20.glGenerateMipmap(GLES20.GL_TEXTURE_2D);* 生成 MIP 贴图。完成这些动作后，我们需要解除与这个纹理的绑定，以免用其他方法以外的改变这个纹理，调用方法  *GLES20.glBindTexture(GLES20.GL_TEXTURE_2D,0);* 。

### 创建新的着色器集合

完成这些之后，我们需要创建一套新的着色器，它们可以接受纹理，并把它应用到要绘制的片段上。在 res 的 raw 文件夹中，我们新建一个 *texture_vertex_shader.glsl* 文件：

    uniform mat4 u_Matrix;

    attribute vec4 a_Position;
    attribute vec2 a_TextureCoordinates;

    varying vec2 v_TextureCoordinates;

    void main()
    {
      v_TextureCoordinates = a_TextureCoordinates;
      gl_Position = u_Matrix*a_Position;
    }

在这里，我们新加了一个纹理坐标的属性，它叫 “ a_TextureCoordinates ”。下一步是一个新的片段着色器，新建一个文件 *texture_fragment_shader.glsl* ：

    precision mediump float;

    uniform sampler2D u_TextureUnit;
    varying vec2 v_TextureCoordinates;

    void main()
    {
      gl_FragColor = texture2D(u_TextureUnit,v_TextureCoordinates);
    }

为了把一个纹理绘制到物体上，OpenGL 会为每一个片段都调用片段着色器，并且每个调用都会接受 v_TextureCoordinates 的纹理坐标。

### 新的类结构

由于我们现在已经有各种不同的对象和着色器，所以需要对之前项目的类结构做相应的调整。大概来说就是将木槌和桌子分为两个类，包含各自的数据和绘制方法。为颜色着色器和纹理着色器各创建一个类，其中过程不予赘述，请参考源码，下面对纹理着色器类进行一下说明，它首先继承于父类 ShaderProgram :

    protected static final String U_MATRIX = "u_Matrix";
    protected static final String U_TEXTURE_UNIT = "u_TextureUnit";

    protected static final String A_POSITION = "a_Position";
    protected static final String A_COLOR = "a_Color";
    protected static final String A_TEXTURE_COORDINATES = "a_TextureCoordinates";

    protected final int program;

    protected ShaderProgram(Context context, int vertexShaderResourceId,
        int fragmentShaderResourceId) {
      program = ShaderHelper.buildProgram(
          TextResourceReader.readTextFileFromResource(context, vertexShaderResourceId),
          TextResourceReader.readTextFileFromResource(context, fragmentShaderResourceId));
    }

    public void useProgram() {
      GLES20.glUseProgram(program);
    }

可以看到这个父类的方法主要是读入顶点着色器和片段着色器的代码，还有使用程序对象的两个方法。再来看纹理着色器类：

    public class TextureShaderProgram extends ShaderProgram {

      private final int uMatrixLocation;
      private final int uTextureUnitLocation;

      private final int aPositionLocation;
      private final int aTextureCoordnatesLocation;

      public TextureShaderProgram(Context context) {
        super(context, R.raw.texture_vertex_shader, R.raw.texture_fragment_shader);
        uMatrixLocation = GLES20.glGetUniformLocation(program, U_MATRIX);
        uTextureUnitLocation = GLES20.glGetUniformLocation(program, U_TEXTURE_UNIT);

        aPositionLocation = GLES20.glGetAttribLocation(program, A_POSITION);
        aTextureCoordnatesLocation = GLES20.glGetAttribLocation(program, A_TEXTURE_COORDINATES);
      }

      @Override public void useProgram() {
        super.useProgram();
      }

      public void setUniforms(float[] matrix, int textureId) {
        GLES20.glUniformMatrix4fv(uMatrixLocation, 1, false, matrix, 0);
        GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureId);
        GLES20.glUniform1i(uTextureUnitLocation, 0);
      }

      public int getPositionAttributeLocation() {
        return aPositionLocation;
      }

      public int getTextureCoordinatesAttributeLocation() {
        return aTextureCoordnatesLocation;
      }
    }

在构造函数中调用了 *super(context, R.raw.texture_vertex_shader, R.raw.texture_fragment_shader);* 首先将顶点着色器和片段着色器的代码读入进来。接下来就是读入各种 uniform 和属性的位置。在方法 *setUniforms* 中，第一步是传递矩阵给它的 uniform ，下一步调用 *GLES20.glActiveTexture(GLES20.GL_TEXTURE0);*，这个方法把活动的纹理单元设置为纹理单元 0 ，然后通过调用 *glBindTexture(GLES20.GL_TEXTURE_2D, textureId);* 把纹理绑定到这个单元，接着通过调用 *glUniform1i(uTextureUnitLocation, 0);* 把被选定的纹理单元传递给片段着色器中的 u_TextureUnit 。

### 使用纹理进行绘制

在新的 AirHockeyRenderer 类中，由于其他类结构的变换导致该类也需要做相应调整，具体不在赘述，请打开源文件参考，我们重点来看下在 *onDrawFrame()* 中是如何用纹理来绘制的：

    @Override public void onDrawFrame(GL10 gl10) {

      GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);

      textureProgram.useProgram();
      textureProgram.setUniforms(projectionMatrix,texture);
      table.bindData(textureProgram);
      table.draw();

      colorProgram.useProgram();
      colorProgram.setUniforms(projectionMatrix);
      mallet.bindData(colorProgram);
      mallet.draw();

    }

清空渲染表面后，我们做的第一件事是绘制桌子，首先调用 *textureProgram.useProgram();* 告诉 OpenGL 使用这个程序，然后通过调用 *textureProgram.setUniforms(projectionMatrix,texture);* 把那些 uniform 传递进去，下一步通过调用 *table.bindData(textureProgram);* 把顶点数组数据和着色器程序绑定起来。最后调用 *table.draw();* 绘制桌子。

完成这些后，我们就能用纹理来装饰那个桌子了。复习一下这一章，我们调整纹理来适应它们将要被绘制的形状，既可以通过调整纹理坐标，也可以通过拉伸或是压缩纹理本身来实现。纹理不会被直接绘制，它们要被绑定到纹理单元，然后把这些纹理单元传递给着色器。通过在纹理单元中把纹理切来切去，我们还可以在场景中绘制不同的纹理，但是过分的切换可能使性能下降。我们也可以同时使用多个纹理单元绘制几个纹理。

下一章我们将学习如何构建简单的物体。


[《OpenGL ES 应用开发实践指南》读书笔记 No.1](http://www.rogerblog.cn/2016/07/18/OpenGL-serise-No1/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.2](http://www.rogerblog.cn/2016/07/18/OpenGL-serise-No2/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.3](http://www.rogerblog.cn/2016/07/19/OpenGL-serise-No3/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.4](http://www.rogerblog.cn/2016/07/20/OpenGL-serise-No4/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.5](http://www.rogerblog.cn/2016/07/20/OpenGL-serise-No5/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.6](http://www.rogerblog.cn/2016/07/21/OpenGL-serise-No6/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.7](http://www.rogerblog.cn/2016/07/22/OpenGL-serise-No7/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.8](http://www.rogerblog.cn/2016/07/24/OpenGL-serise-No8/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.9](http://www.rogerblog.cn/2016/07/26/OpenGL-serise-No9/)
