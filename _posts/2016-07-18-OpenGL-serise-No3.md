---
layout:     post
title:      "《OpenGL ES 应用开发实践指南》读书笔记 No.3"
subtitle:   "Android OpenGL ES 从入门到奔溃"
date: 2016-07-19 11:00:01 +0800
author:     "Roger"
header-img: "img/opengl-bg.jpg"
tags:
    - OpenGL ES
---
Android OpenGL ES 第三章 - 编译着色器及在屏幕上绘图
---

本系列所有源码地址：[https://github.com/Rogero0o/OpenGL_Demo](https://github.com/Rogero0o/OpenGL_Demo)

请大家务必对照源码阅读本文，否则有如盲人摸象。

上一章我们学习了顶点和着色器的基本知识，现在我们看看如何编译着色器，本项目源码还是 AirHockey1 。

### 加载、编译着色器

上一章中我们将着色器代码定义在 raw 文件夹中，现在我们写一个工具类将其读取出来，代码在 util 包中的 TextResourceReader 类中。

    public class TextResourceReader {
      public static String readTextFileFromResource(Context context, int resourceId) {
        StringBuilder body = new StringBuilder();

        try {
          InputStream inputStream = context.getResources().openRawResource(resourceId);
          InputStreamReader inputStreamReader = new InputStreamReader(inputStream);
          BufferedReader bufferedReader = new BufferedReader(inputStreamReader);
          String nextLine;
          while ((nextLine = bufferedReader.readLine()) != null) {
            body.append(nextLine);
            body.append('\n');
          }
        } catch (IOException e) {
          throw new RuntimeException("Could not open resource:" + resourceId, e);
        } catch (Resources.NotFoundException nfe) {
          throw new RuntimeException("Resource not found :" + resourceId, nfe);
        }
        return body.toString();
      }
    }

我们将使用 TextResourceReader.readTextFileFromResource 这个方法来读取着色器代码。切换到 AirHockeyRender.java 中的 onSurfaceCreated 末尾加入如下代码：

    String vertexShaderSource = TextResourceReader.readTextFileFromResource(context, R.raw.simple_vertex_shader);//读取顶点着色器
    String fragmentShaderSource = TextResourceReader.readTextFileFromResource(context, R.raw.simple_fragment_shader);//读取片段着色器

下一步我们将编译着色器，在 util 中创建一个 ShaderHelper 的工具类，我先把完整的类贴出来，然后再对其中的细节进行拆解：

    public class ShaderHelper {
        private static final String TAG = "ShaderHelper";

        /**
         * Loads and compiles a vertex shader, returning the OpenGL object ID.
         */
        public static int compileVertexShader(String shaderCode) {
            return compileShader(GL_VERTEX_SHADER, shaderCode);
        }

        /**
         * Loads and compiles a fragment shader, returning the OpenGL object ID.
         */
        public static int compileFragmentShader(String shaderCode) {
            return compileShader(GL_FRAGMENT_SHADER, shaderCode);
        }

        /**
         * Compiles a shader, returning the OpenGL object ID.
         */
        private static int compileShader(int type, String shaderCode) {

            // Create a new shader object.
            final int shaderObjectId = glCreateShader(type);

            if (shaderObjectId == 0) {
                if (LoggerConfig.ON) {
                    Log.w(TAG, "Could not create new shader.");
                }

                return 0;
            }

            // Pass in the shader source.
            glShaderSource(shaderObjectId, shaderCode);

            // Compile the shader.
            glCompileShader(shaderObjectId);

            // Get the compilation status.
            final int[] compileStatus = new int[1];
            glGetShaderiv(shaderObjectId, GL_COMPILE_STATUS, compileStatus, 0);

            if (LoggerConfig.ON) {
                // Print the shader info log to the Android log output.
                Log.v(TAG, "Results of compiling source:" + "\n" + shaderCode + "\n:"
                    + glGetShaderInfoLog(shaderObjectId));
            }

            // Verify the compile status.
            if (compileStatus[0] == 0) {
                // If it failed, delete the shader object.
                glDeleteShader(shaderObjectId);

                if (LoggerConfig.ON) {
                    Log.w(TAG, "Compilation of shader failed.");
                }

                return 0;
            }

            // Return the shader object ID.
            return shaderObjectId;
        }
        /**
         * Links a vertex shader and a fragment shader together into an OpenGL
         * program. Returns the OpenGL program object ID, or 0 if linking failed.
         */
        public static int linkProgram(int vertexShaderId, int fragmentShaderId) {

            // Create a new program object.
            final int programObjectId = glCreateProgram();

            if (programObjectId == 0) {
                if (LoggerConfig.ON) {
                    Log.w(TAG, "Could not create new program");
                }

                return 0;
            }

            // Attach the vertex shader to the program.
            glAttachShader(programObjectId, vertexShaderId);
            // Attach the fragment shader to the program.
            glAttachShader(programObjectId, fragmentShaderId);

            // Link the two shaders together into a program.
            glLinkProgram(programObjectId);

            // Get the link status.
            final int[] linkStatus = new int[1];
            glGetProgramiv(programObjectId, GL_LINK_STATUS, linkStatus, 0);

            if (LoggerConfig.ON) {
                // Print the program info log to the Android log output.
                Log.v(TAG, "Results of linking program:\n"
                    + glGetProgramInfoLog(programObjectId));			
            }

            // Verify the link status.
            if (linkStatus[0] == 0) {
                // If it failed, delete the program object.
                glDeleteProgram(programObjectId);
                if (LoggerConfig.ON) {
                    Log.w(TAG, "Linking of program failed.");
                }
                return 0;
            }

            // Return the program object ID.
            return programObjectId;
        }
        /**
         * Validates an OpenGL program. Should only be called when developing the
         * application.
         */
        public static boolean validateProgram(int programObjectId) {
            glValidateProgram(programObjectId);

            final int[] validateStatus = new int[1];
            glGetProgramiv(programObjectId, GL_VALIDATE_STATUS, validateStatus, 0);
            Log.v(TAG, "Results of validating program: " + validateStatus[0]
                + "\nLog:" + glGetProgramInfoLog(programObjectId));

            return validateStatus[0] != 0;
        }
    }

*compileVertexShader* 和 *compileFragmentShader* 分别为创建顶点着色器和片段着色器。他们都调用了 *compileShader* 这个方法，在该方法中的大概逻辑如下：

1. *glCreateShader()* 创建一个新的着色器对象，该方法的唯一参数为一个 TYEP 值，可以为 GL_VERTEX_SHADER 和 GL_FRAGMENT_SHADER ，分别代表编译顶点着色器和片段做色漆，该方法会返回一个 int 值，这个值可以理解为这个着色器对象的指针，也就是该着色器对象的应用，如果返回0则表示创建失败。
2. *glShaderSource()* 这个方法将我们刚刚读取的着色器代码关联到到第一步中创建的着色器对象中
3. *glCompileShader()* 这个方法即编译着色器的源码

编译过后，我们需要取出编译状态以及在错误时取出信息日志，如下代码完成上述功能：

    // Get the compilation status.
    final int[] compileStatus = new int[1];
    glGetShaderiv(shaderObjectId, GL_COMPILE_STATUS, compileStatus, 0);

    if (LoggerConfig.ON) {
        // Print the shader info log to the Android log output.
        Log.v(TAG, "Results of compiling source:" + "\n" + shaderCode + "\n:"
            + glGetShaderInfoLog(shaderObjectId));
    }

    // Verify the compile status.
    if (compileStatus[0] == 0) {
        // If it failed, delete the shader object.
        glDeleteShader(shaderObjectId);

        if (LoggerConfig.ON) {
            Log.w(TAG, "Compilation of shader failed.");
        }

        return 0;
    }

接下来在 AirHockeyRender.java 中的 onSurfaceCreated 的结尾处加入如下代码，完成对着色器编译的调用：

    int vertexShader = ShaderHelper.compileVertexShader(vertexShaderSource);
    int fragmentShader = ShaderHelper.compileFragmentShader(fragmentShaderSource);

现在我们已经加载并编译了一个顶点着色器和一个片段着色器，下一步就是将他们绑定到一个单一的程序中。这个功能由 linkProgram 方法实现，大概流程如下。

1. 通过 *glCreateProgram()* 创建一个程序对象，同样返回值是一个代表该对象指针的 int
2. 使用 *glAttachShader()* 方法将刚刚创建的顶点着色器和片段着色器都附加到程序对象上
3. 在完成附加动作后，我们还需要调用 *glLinkProgram()* 来将着色器和对象链接起来，只有链接成功后才能在代码中使用该对象
4. 通过 *glGetProgramiv()* 方法判断链接状态，判断方法如源码所示。

在完成这些后，我们就可以在 AirHockeyRenderer 中的 onSurfaceCreated 的结尾处加入方法的调用：

    program = ShaderHelper.linkProgram(vertexShader, fragmentShader);

在使用程序对象前，我们需要验证该对象是否可用，具体方法如 *validateProgram()* 方法中所示。

接下来就可以调用 *glUseProgram()* 来告诉 OpenGL 在绘制任何东西在屏幕上的时候要使用这里定义的程序。

下一步，我们将对着色器中的变量进行赋值，还记得上一章的重点吗，顶点着色器必须对 a_Position 赋值，片段着色器必须对 u_Color 赋值。所以我们使用 *glGetUniformLocation(program, U_COLOR);* 和 *glGetAttribLocation(program, A_POSITION);* 来获得 a_Position 以及 u_Color 着色器对象的指针，有了这个指针，就能调用以下代码，将数据传入着色器中：

    vertexData.position(0);
    glVertexAttribPointer(aPositionLocation, POSITION_COMPONENT_COUNT, GL_FLOAT,
        false, 0, vertexData);

*vertexData.position(0);* 在这里， *vertexData* 是本章开始时创建的缓冲区，调用 *position(0)* 将位置设在数据的开头处。

*void glVertexAttribPointer( GLuint index, GLint size, GLenum type, GLboolean normalized, GLsizei stride,const GLvoid * pointer);* 这个方法十分的重要，它指定了渲染时索引值为 index 的顶点属性数组的数据格式和位置。具体参数如下：

    参数：
    index
    指定要修改的顶点属性的索引值
    size
    指定每个顶点属性的组件数量。必须为1、2、3或者4。初始值为4。（如position是由3个（x,y,z）组成，而颜色是4个（r,g,b,a））
    type
    指定数组中每个组件的数据类型。可用的符号常量有GL_BYTE, GL_UNSIGNED_BYTE, GL_SHORT,GL_UNSIGNED_SHORT, GL_FIXED, 和 GL_FLOAT，初始值为GL_FLOAT。
    normalized
    指定当被访问时，固定点数据值是否应该被归一化（GL_TRUE）或者直接转换为固定点值（GL_FALSE）。
    stride
    指定连续顶点属性之间的偏移量。如果为0，那么顶点属性会被理解为：它们是紧密排列在一起的。初始值为0。
    pointer
    指定第一个组件在数组的第一个顶点属性中的偏移量。该数组与GL_ARRAY_BUFFER绑定，储存于缓冲区中。初始值为0；

通过调用该方法， OpenGL 就知道在哪里读取属性 a_Position 的数据了。顶点着色器的参数在此定义完成，片段着色器的参数会在后面进行定义。下一步需要使能顶点数组，通过下面的方法：

    glEnableVertexAttribArray(aPositionLocation);

通过这个方法，OpenGL 才知道去哪里寻找它需要的参数。

### 在屏幕上绘制

接下来我们完成绘制的工作，在 onDrawFrame() 的结尾处，在 glClear() 调用之后加入如下代码：

    // Draw the table.
    glUniform4f(uColorLocation, 1.0f, 1.0f, 1.0f, 1.0f);		
    glDrawArrays(GL_TRIANGLES, 0, 6);

*glUniform4f* 的作用是为当前程序对象指定Uniform变量的值。这里即设置了片段着色器中的 u_Color 的值，后四个参数分别为红色、绿色和蓝色还有透明度的值，一旦指定了颜色，接下来我们就可以调用 *glDrawArrays* 来绘制桌子了，第一个参数 *GL_TRIANGLES* 说明我们想绘制三角形，第二个参数表示告诉 OpenGL 从顶点数组的开头处开始读顶点，第三个参数是告诉 OpenGL 读入六个顶点。因为每个三角形有三个顶点，所以这里将绘制出两个三角形。他们的坐标是：

    // Triangle 1
    0f,  0f,
    9f, 14f,
    0f, 14f,

    // Triangle 2
    0f,  0f,
    9f,  0f,							
    9f, 14f

下一步是绘制桌子中间的分割线：

    // Draw the center dividing line.
    glUniform4f(uColorLocation, 1.0f, 0.0f, 0.0f, 1.0f);		
    glDrawArrays(GL_LINES, 6, 2);

可以看到这里的线将会是红色，并且 *glDrawArrays* 的第一个参数是 *GL_LINES* ，告诉 OpenGL 将要绘制的是直线。坐标是：

    // Line 1
    0f,  7f,
    9f,  7f,

最后绘制两个点来代表木槌：

    // Draw the first mallet blue.        
    glUniform4f(uColorLocation, 0.0f, 0.0f, 1.0f, 1.0f);		
    glDrawArrays(GL_POINTS, 8, 1);

    // Draw the second mallet red.
    glUniform4f(uColorLocation, 1.0f, 0.0f, 0.0f, 1.0f);		
    glDrawArrays(GL_POINTS, 9, 1);

在完成这些之后，就可以将项目运行起来了。但是你将会发现这完全不是你想要的，没关系，那是因为我们还没将 OpenGL 的坐标映射到屏幕。在 OpenGL 中，无论是x还是y OpenGL 都会将屏幕映射到 [-1,1] 的范围内。所以我们要将之前定义的坐标做一些改变。

    float[] tableVerticesWithTriangles = {
        // Triangle 1
        -0.5f, -0.5f,
         0.5f,  0.5f,
        -0.5f,  0.5f,

        // Triangle 2
        -0.5f, -0.5f,
         0.5f, -0.5f,
         0.5f,  0.5f,

        // Line 1
        -0.5f, 0f,
         0.5f, 0f,

        // Mallets
        0f, -0.25f,
        0f,  0.25f
    };

再次运行这个程序，你将看到一个桌子，上边被一条横线分割开，还有两个代表木槌的点，那么你已经成功了。
