---
layout:     post
title:      "《OpenGL ES 应用开发实践指南》读书笔记 No.9"
subtitle:   "Android OpenGL ES 从入门到奔溃"
date: 2016-07-26 11:00:01 +0800
author:     "Roger"
header-img: "img/opengl-bg.jpg"
tags:
    - OpenGL ES
---
Android OpenGL ES 第九章 - 增加触控反馈，与空气曲棍球游戏交互
---

本系列所有源码地址：[https://github.com/Rogero0o/OpenGL_Demo](https://github.com/Rogero0o/OpenGL_Demo)

请大家务必对照源码阅读本文，否则有如盲人摸象。

上一章我们学习了如何使用三角形构建物体，这一章我们将学习如何为项目添加触控的功能。这一章的项目名为 AirHockeyTouch 。

### 为 Activity 增加触控支持

首先我们需要保存渲染器的一个引用，因此打开 *AirHockeyActivity* ，并按如下代码修改 *setRenderer()* :

    final AirHockeyRenderer airHockeyRenderer = new AirHockeyRenderer(this);

    if (supportsEs2) {
           // ...
           glSurfaceView.setRenderer(airHockeyRenderer);

下一步就可以开始写触控处理程序了，在调用 *setContentView()* 之前加入如下代码：

    glSurfaceView.setOnTouchListener(new OnTouchListener() {
                @Override
                public boolean onTouch(View v, MotionEvent event) {
                    if (event != null) {           
                        // Convert touch coordinates into normalized device
                        // coordinates, keeping in mind that Android's Y
                        // coordinates are inverted.
                        final float normalizedX =
                            (event.getX() / (float) v.getWidth()) * 2 - 1;
                        final float normalizedY =
                            -((event.getY() / (float) v.getHeight()) * 2 - 1);

                        if (event.getAction() == MotionEvent.ACTION_DOWN) {
                            glSurfaceView.queueEvent(new Runnable() {
                                @Override
                                public void run() {
                                    airHockeyRenderer.handleTouchPress(
                                        normalizedX, normalizedY);
                                }
                            });
                        } else if (event.getAction() == MotionEvent.ACTION_MOVE) {
                            glSurfaceView.queueEvent(new Runnable() {
                                @Override
                                public void run() {
                                    airHockeyRenderer.handleTouchDrag(
                                        normalizedX, normalizedY);
                                }
                            });
                        }                    

                        return true;                    
                    } else {
                        return false;
                    }
                }
            });

我们在着色器中需要使用归一化设备坐标，因此我们需要把触控时间坐标转换回归一化设备坐标，这需要把 y 轴反转，并把每个坐标按比例映射到范围 [-1,1] 内。下一步分情况处理按下状态和拖拽状态，**由于 Android 的 UI 运行在主线程，而 OpenGL 的 GLSurfaceView 运行在一个单独的线程中** ，因此我们需要使用线程安全的技术在两个线程间通信。 我们使用 *queueEvent()* 给 OpenGL 线程分发调用。


### 增加相交测试

现在我们已经得到了屏幕被触碰的区域，就需要判断这个区域是否涵盖木槌。我们需要执行相交测试，这是开发三维游戏的一个非常重要的操作，大概分为如下两部：
1. 首先，我们要把二维屏幕坐标转换到三维坐标中，并看看我们触碰到了什么。要做到这点，我们要把被触碰的点投射到一条射线上，这条射线从我们的视点跨越的那个三维场景。
2. 然后，我们需要检查看看这条射线是否与木槌相交。为了使事情简单一些，我们假定那个木槌实际上时一个差不多同样大小的包围球，然后测试那个球。

让我们先创建两个新的成员变量，在 *AirHockeyRenderer* 中：

    private boolean malletPressed = false;
    private Point blueMalletPosition;   

它需要被初始化为一个默认值，因此给 *onSurfaceCreated()* 添加如下代码：

  blueMalletPosition = new Point(0f, mallet.height / 2f, 0.4f);

接下来更新 *handleTGouchPress()* :

    public void handleTouchPress(float normalizedX, float normalizedY) {

        Ray ray = convertNormalized2DPointToRay(normalizedX, normalizedY);

        // Now test if this ray intersects with the mallet by creating a
        // bounding sphere that wraps the mallet.
        Sphere malletBoundingSphere = new Sphere(new Point(
                blueMalletPosition.x,
                blueMalletPosition.y,
                blueMalletPosition.z),
            mallet.height / 2f);

        // If the ray intersects (if the user touched a part of the screen that
        // intersects the mallet's bounding sphere), then set malletPressed =
        // true.
        malletPressed = Geometry.intersects(malletBoundingSphere, ray);
    }

要计算被触碰的点是否与木槌相交，我们首先要把被触碰的点投射到一条射线上，用一个包围球封装木槌，然后测试一下看看那条射线是否与球相交。

通常，当我们把一个三维场景投递到二维屏幕的时候，我们使用透视投影和透视除法把顶点坐标变换为归一化设备坐标。现在我们想按相反的方向变换：我们有被触碰点的归一化设备坐标，我们要计算处在三维世界里那个被触碰的点与哪里相对应。为了把被触碰的点转换为一个三维射线，实质上我们需要取消透视投影和透视除法。

我们当前有被触碰点的 x 和 y 坐标，但我们还不知道它应该在多远或多近的地方。要解决这个模糊性，我们把被触碰的点映射到三维空间的一条直线：直线的近端映射到我们在投影矩阵中定义的视锥体的近平面，直线的远端映射到视锥体的远平面。要实现这个转换，我们需要一个反转矩阵：

在 *AirHockeyRenderer* 中：

    private final float[] invertedViewProjectionMatrix = new float[16];

在 *onDrawFrame()* 里，在调用 *multiplyMM()* 之后加入如下一行代码：

    invertM(invertedViewProjectionMatrix, 0, viewProjectionMatrix, 0);

这个调用会创建一个反转的矩阵，我们可以用它把那个二维被触碰的点转换为两个三维坐标。现在我们来看看 *convertNormalized2DPointToRay()* 这个方法：

    private Ray convertNormalized2DPointToRay(
        float normalizedX, float normalizedY) {
        // We'll convert these normalized device coordinates into world-space
        // coordinates. We'll pick a point on the near and far planes, and draw a
        // line between them. To do this transform, we need to first multiply by
        // the inverse matrix, and then we need to undo the perspective divide.
        final float[] nearPointNdc = {normalizedX, normalizedY, -1, 1};
        final float[] farPointNdc =  {normalizedX, normalizedY,  1, 1};

        final float[] nearPointWorld = new float[4];
        final float[] farPointWorld = new float[4];

        multiplyMV(
            nearPointWorld, 0, invertedViewProjectionMatrix, 0, nearPointNdc, 0);
        multiplyMV(
            farPointWorld, 0, invertedViewProjectionMatrix, 0, farPointNdc, 0);

        // Why are we dividing by W? We multiplied our vector by an inverse
        // matrix, so the W value that we end up is actually the *inverse* of
        // what the projection matrix would create. By dividing all 3 components
        // by W, we effectively undo the hardware perspective divide.
        divideByW(nearPointWorld);
        divideByW(farPointWorld);

        // We don't care about the W value anymore, because our points are now
        // in world coordinates.
        Point nearPointRay =
            new Point(nearPointWorld[0], nearPointWorld[1], nearPointWorld[2]);

        Point farPointRay =
            new Point(farPointWorld[0], farPointWorld[1], farPointWorld[2]);

        return new Ray(nearPointRay,
                       Geometry.vectorBetween(nearPointRay, farPointRay));
    }

为了把被触碰的点映射到一条直线，我们在归一化设备坐标里设置了两个点：其中一个点是 z 值为 -1 的点，另一个点是 z 值为 1 的点。分别存储在 *nearPoingNdc* 和 *farPointNdc* 。 因为我们不知道 w 分量应该被设为多大，暂且把它们的 w 分量设为 1 。 接下来，我们把每个点都与 *invertedViewProjectionMatrix* 相乘，得到世界空间中的坐标。我们也需要撤销透视除法的影响。反转的视图投影矩阵有一个有趣的属性：把顶点和反转的视图投影矩阵相乘后， *nearPoitWorld* 和 *farPointWorld* 实际上就含有了反转的 w 值。这是因为，通常情况下，投影矩阵的主要意义就是创建不同的 w 值，以便透视除法可以施加它的魔法，因此，如果我们使用一个反转的投影矩阵，我们就会得到一个反转的 w 。所以我们需要做的就是把 x , y , z 除以这些反转的 w， 这样就撤销了透视除法的影响。在代码中体现就是 *divideByW* 这个方法：

    private void divideByW(float[] vector) {
        vector[0] /= vector[3];
        vector[1] /= vector[3];
        vector[2] /= vector[3];
    }

 现在我们已经成功的把一个被触碰的点转换为世界空间中的两个点了。我们现在可以用这两个点定义一个跨越那个三维场景的射线了。一下代码完成这个逻辑：

     Point nearPointRay =
         new Point(nearPointWorld[0], nearPointWorld[1], nearPointWorld[2]);

     Point farPointRay =
         new Point(farPointWorld[0], farPointWorld[1], farPointWorld[2]);

     return new Ray(nearPointRay,
                    Geometry.vectorBetween(nearPointRay, farPointRay));

我们还需要添加几个其中用到的方法，在 *Geometry* 中，加入如下方法：

    public static class Ray {
        public final Point point;
        public final Vector vector;

        public Ray(Point point, Vector vector) {
            this.point = point;
            this.vector = vector;
        }        
    }

    public static class Vector  {
        public final float x, y, z;

        public Vector(float x, float y, float z) {
            this.x = x;
            this.y = y;
            this.z = z;
        }
    }

    public static Vector vectorBetween(Point from, Point to) {
        return new Vector(
            to.x - from.x,
            to.y - from.y,
            to.z - from.z);
    }

现在我们已经完成了第一部分：把一个被触碰的点转换为一条三维射线。现在我们需要加入相交测试了。

之前提到过，我们需要假定木槌是一个球体，现在我们来定义一个与木槌大小相当的包围球：

    Sphere malletBoundingSphere = new Sphere(new Point(
            blueMalletPosition.x,
            blueMalletPosition.y,
            blueMalletPosition.z),
        mallet.height / 2f);

需要在 *Geometry* 中加入如下代码：

    public static class Sphere {
        public final Point center;
        public final float radius;

        public Sphere(Point center, float radius) {
            this.center = center;
            this.radius = radius;
        }
    }

我们还需要加入相交测试：

    malletPressed = Geometry.intersects(malletBoundingSphere, ray);

跟进这个方法，来到 *Geometry* 类中：

    public static boolean intersects(Sphere sphere, Ray ray) {
        return distanceBetween(sphere.center, ray) < sphere.radius;
    }

    public static float distanceBetween(Point point, Ray ray) {
        Vector p1ToPoint = vectorBetween(ray.point, point);
        Vector p2ToPoint = vectorBetween(ray.point.translate(ray.vector), point);

        // The length of the cross product gives the area of an imaginary
        // parallelogram having the two vectors as sides. A parallelogram can be
        // thought of as consisting of two triangles, so this is the same as
        // twice the area of the triangle defined by the two vectors.
        // http://en.wikipedia.org/wiki/Cross_product#Geometric_meaning
        float areaOfTriangleTimesTwo = p1ToPoint.crossProduct(p2ToPoint).length();
        float lengthOfBase = ray.vector.length();

        // The area of a triangle is also equal to (base * height) / 2. In
        // other words, the height is equal to (area * 2) / base. The height
        // of this triangle is the distance from the point to the ray.
        float distanceFromPointToRay = areaOfTriangleTimesTwo / lengthOfBase;
        return distanceFromPointToRay;
    }

这个方法使用一个三角形来计算射线和球体的距离，然后和球体的半径对比即可检测二者是否相交。让我们完善 *Geometry* 类，添加以下几个方法：

    public Point translate(Vector vector) {
        return new Point(
            x + vector.x,
            y + vector.y,
            z + vector.z);
    }

    //在 *Vector* 类中加入如下两个方法

    public float length() {
        return FloatMath.sqrt(
            x * x
          + y * y
          + z * z);
    }

    public Vector crossProduct(Vector other) {
        return new Vector(
            (y * other.z) - (z * other.y),
            (z * other.x) - (x * other.z),
            (x * other.y) - (y * other.x));
    }

现在我们已经完成了按下的逻辑，下一步是通过拖动移动物体的逻辑。

让我们完成 *handleTouchDrag()* 的定义：

    public void handleTouchDrag(float normalizedX, float normalizedY) {

        if (malletPressed) {
            Ray ray = convertNormalized2DPointToRay(normalizedX, normalizedY);
            // Define a plane representing our air hockey table.
            Plane plane = new Plane(new Point(0, 0, 0), new Vector(0, 1, 0));
            // Find out where the touched point intersects the plane
            // representing our table. We'll move the mallet along this plane.
            Point touchedPoint = Geometry.intersectionPoint(ray, plane);
            // Clamp to bounds                        

            previousBlueMalletPosition = blueMalletPosition;            
            /*
            blueMalletPosition =
                new Point(touchedPoint.x, mallet.height / 2f, touchedPoint.z);
            */
            // Clamp to bounds            
            blueMalletPosition = new Point(
                clamp(touchedPoint.x,
                      leftBound + mallet.radius,
                      rightBound - mallet.radius),
                mallet.height / 2f,
                clamp(touchedPoint.z,
                      0f + mallet.radius,
                      nearBound - mallet.radius));            

            // Now test if mallet has struck the puck.
            float distance =
                Geometry.vectorBetween(blueMalletPosition, puckPosition).length();

            if (distance < (puck.radius + mallet.radius)) {
                // The mallet has struck the puck. Now send the puck flying
                // based on the mallet velocity.
                puckVector = Geometry.vectorBetween(
                    previousBlueMalletPosition, blueMalletPosition);                
            }
        }
    }

在判断木槌被按下之后，我们就做射线转换，它与我们在 *handlerTouchPress()* 中的转换是一样的。一旦我们有了表示被触碰点的射线，我们就要找出这条射线与表示空气曲棍球桌子的平面在哪里相交了，然后，把木槌移动到那个点。完善 *Geomitry* 中 *Plane* 的定义：

    public static class Plane {                
        public final Point point;
        public final Vector normal;

        public Plane(Point point, Vector normal) {                        
            this.point = point;
            this.normal = normal;
        }
    }

平面的定义非常简单：包含一个法向向量和平面上的一个点。下面让我们加入如下代码来计算射线和平面相交的点：

    public static Point intersectionPoint(Ray ray, Plane plane) {        
        Vector rayToPlaneVector = vectorBetween(ray.point, plane.point);

        float scaleFactor = rayToPlaneVector.dotProduct(plane.normal)
                          / ray.vector.dotProduct(plane.normal);

        Point intersectionPoint = ray.point.translate(ray.vector.scale(scaleFactor));
        return intersectionPoint;
    }

要计算这个交点，我们需要计算出射线的向量要缩放多少才能刚好与平面相接触；这个因子就是缩放因子。接下来我们用这个被缩放的向量平移射线的点来找出这个相交点。

要计算这个缩放因子，我们首先创建一个向量，它在射线的七点和平面上的一个点之间。然后计算那个向量与平面的法向向量之间的点积。这两个向量的点积与他们之间的余弦直接相关。我们可以用射线到平面的向量与平面法向量之间的点积除以射线向量与平面法向量的点积。这就得到了我们需要的缩放因子。

给 *Vector* 类加入如下代码来填充余下的空白：

    public float dotProduct(Vector other) {
        return x * other.x
             + y * other.y
             + z * other.z;
    }

    public Vector scale(float f) {
        return new Vector(
            x * f,
            y * f,
            z * f);
    }

最后，让我们更新 *onDrawFrame()* ，并把第二个 *positionObjectInScene()* 调用更新为如下代码：

    positionObjectInScene(blueMalletPosition.x, blueMalletPosition.y,
               blueMalletPosition.z);

现在我们可以运行一下程序，你应该能在屏幕上来回拖动那个木槌了！

接下来我们增加一下碰撞检测，以便让木槌待在它应该在的地方。首先在 *AirHockeyRenderer* 加入如下定义：

    private final float leftBound = -0.5f;
    private final float rightBound = 0.5f;
    private final float farBound = -0.8f;
    private final float nearBound = 0.8f;

更新 *handleTouchDrag()* 并用下面的代码替换 *blueMalletPosition* 的值：

    blueMalletPosition =
        new Point(touchedPoint.x, mallet.height / 2f, touchedPoint.z);

再为 *clamp()* 加入定义：

    private float clamp(float value, float min, float max) {
        return Math.min(max, Math.max(value, min));
    }

下一步我们添加一些代码来击打冰球了，在 *AirHockeyRenderer* 中加入几个新的成员变量：

    private Point previousBlueMalletPosition;
    private Point puckPosition;
    private Vector puckVector;

在 *handleTouchDrag()* 中加入如下代码：

    previousBlueMalletPosition = blueMalletPosition;

在 *onSurfaceCreated()* 中加入：

    puckPosition = new Point(0f, puck.height / 2f, 0f);
    puckVector = new Vector(0f, 0f, 0f);

现在我们可以在 *handleTouchDrag()* 结尾处加入如下的碰撞检测代码：

    float distance =
        Geometry.vectorBetween(blueMalletPosition, puckPosition).length();

    if (distance < (puck.radius + mallet.radius)) {
        // The mallet has struck the puck. Now send the puck flying
        // based on the mallet velocity.
        puckVector = Geometry.vectorBetween(
            previousBlueMalletPosition, blueMalletPosition);                
    }

这段代码首先检测蓝色木槌和冰球的距离，然后，它会看那个距离是否小于它们的半径之和。如果是，就创建一个方向向量，木槌移动得越快，那个向量就越大，冰球也会移动得越快。

我们需要更新 *onDrawFrame()* ，在开始处加入：

    puckPosition = puckPosition.translate(puckVector);

最后一步，绘制冰球前，我们需要更新 *positionObjectInScene()* :

    positionObjectInScene(puckPosition.x, puckPosition.y, puckPosition.z);

接下来我们为冰球加入边界反射，在 *onDrawFrame()* 中，我们可以在 *puckPosition.translate()* 调用之后加入如下代码：

    if (puckPosition.x < leftBound + puck.radius
     || puckPosition.x > rightBound - puck.radius) {
        puckVector = new Vector(-puckVector.x, puckVector.y, puckVector.z);
        puckVector = puckVector.scale(0.9f);
    }        
    if (puckPosition.z < farBound + puck.radius
     || puckPosition.z > nearBound - puck.radius) {
        puckVector = new Vector(puckVector.x, puckVector.y, -puckVector.z);
        puckVector = puckVector.scale(0.9f);
    }

    puckPosition = new Point(
        clamp(puckPosition.x, leftBound + puck.radius, rightBound - puck.radius),
        puckPosition.y,
        clamp(puckPosition.z, farBound + puck.radius, nearBound - puck.radius)
    );

现在，冰球就能在桌子内来回弹了。下一步我们增加摩擦，在 *onDrawFrame()* 中，冰球相关的代码结尾处加入如下代码：

    puckVector = puckVector.scale(0.99f);

通过给桌子的反弹加入额外的阻尼，我们可以使这个游戏更真实。一旦落入每次反弹检查的条件里面，加入两遍下面的代码：

    puckVector = puckVector.scale(0.9f);

现在，我们已经完成了这个游戏，或许现在它看起来还是有些简陋，但是在构造的过程中我们学习了许多 OpenGL 最基本的知识：

1. 首先我们搞清楚了着色器是如何工作的
2. 学习通过颜色、矩阵和纹理来构建事物
3. 我们学习了如何构建简单的物体
4. 完善游戏细节，加入碰撞检测和阻尼等功能

至此，我们已经完成了这本书大部分的内容，后边部分为粒子系统、天空盒、地形、灯光和 Android 的动态壁纸，有兴趣的同学可以购买本书进行学习。

希望通过这 9 片博文，能让大家初步了解 OpenGL 。如果有问题或疑问，请留言。 Thanks : ) 。

[《OpenGL ES 应用开发实践指南》读书笔记 No.1](http://www.rogerblog.cn/2016/07/18/OpenGL-serise-No1/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.2](http://www.rogerblog.cn/2016/07/18/OpenGL-serise-No2/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.3](http://www.rogerblog.cn/2016/07/19/OpenGL-serise-No3/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.4](http://www.rogerblog.cn/2016/07/20/OpenGL-serise-No4/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.5](http://www.rogerblog.cn/2016/07/20/OpenGL-serise-No5/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.6](http://www.rogerblog.cn/2016/07/21/OpenGL-serise-No6/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.7](http://www.rogerblog.cn/2016/07/22/OpenGL-serise-No7/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.8](http://www.rogerblog.cn/2016/07/24/OpenGL-serise-No8/)

[《OpenGL ES 应用开发实践指南》读书笔记 No.9](http://www.rogerblog.cn/2016/07/26/OpenGL-serise-No9/)
