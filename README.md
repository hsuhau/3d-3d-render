### How to create your own simple 3D render engine in pure Java



[August 03, 2015](http://blog.rogach.org/2015/08/how-to-create-your-own-simple-3d-render.html)

[![img](http://1.bp.blogspot.com/-qbvbVmdthJ8/VbvrOwfKjvI/AAAAAAAAADs/dNUpUnK4xyM/s400/header.png)](http://1.bp.blogspot.com/-qbvbVmdthJ8/VbvrOwfKjvI/AAAAAAAAADs/dNUpUnK4xyM/s1600/header.png)

3D render engines that are nowdays used in games and multimedia production are breathtaking in complexity of mathematics and programming used. Results they produce are correspondingly stunning.

Many developers may think that building even the simplest 3D application from scratch requires inhuman knowledge and effort, but thankfully that isn't always the case. Here I'd like to share with you how you can build your very own 3D render engine, fully capable of producing nice-looking 3D images.

Why would you want to build a 3D engine? At the very least, it will really help understanding how real modern engines do their black magic. Also it is sometimes useful to add 3D rendering capabilities to your application without calling to huge external dependencies. In case of Java, that means that you can build 3D viewer app with zero dependencies (apart from Java APIs) that will run almost anywhere - and fit into 50kb!

Of course, if you want to build big 3D applications with fluid graphics, you'll be much better off with using OpenGL/WebGL. Still, once you will have a basic understanding of 3D engine internals, more complex engines will seem much more approachable.

In this post, I will be covering basic 3d rendering with orthographic projection, simple triangle rasterization, z-buffering and flat shading. I will not be focusing on heavy performance optimizations and more complex topics like textures or different lighting setups - if you need that, consider using better suited tools for that, like OpenGL (there are lots of libraries that allow you to work with OpenGL even from Java).

Code examples will be in Java, but the ideas explained here can be applied to any language of your choice. For your convenience, I will be following along with small interactive JavaScript demos right here in the post.

Enough talk - let's begin!

#### GUI wrapper

First of all, we want to put at least *something* on screen. For that I will use very simple application with our rendered image and two sliders to adjust the rotation.

```
import javax.swing.*;
import java.awt.*;

public class DemoViewer {

    public static void main(String[] args) {
        JFrame frame = new JFrame();
        Container pane = frame.getContentPane();
        pane.setLayout(new BorderLayout());

        // slider to control horizontal rotation
        JSlider headingSlider = new JSlider(0, 360, 180);
        pane.add(headingSlider, BorderLayout.SOUTH);

        // slider to control vertical rotation
        JSlider pitchSlider = new JSlider(SwingConstants.VERTICAL, -90, 90, 0);
        pane.add(pitchSlider, BorderLayout.EAST);

        // panel to display render results
        JPanel renderPanel = new JPanel() {
                public void paintComponent(Graphics g) {
                    Graphics2D g2 = (Graphics2D) g;
                    g2.setColor(Color.BLACK);
                    g2.fillRect(0, 0, getWidth(), getHeight());

                    // rendering magic will happen here
                }
            };
        pane.add(renderPanel, BorderLayout.CENTER);

        frame.setSize(400, 400);
        frame.setVisible(true);
    }
}
```

The resulting window should resemble this:

[![img](http://4.bp.blogspot.com/-FPLVEWmkdfg/Vbvr4SEK5aI/AAAAAAAAAD0/4--15P21MX8/s400/wrapper-setup.png)](http://4.bp.blogspot.com/-FPLVEWmkdfg/Vbvr4SEK5aI/AAAAAAAAAD0/4--15P21MX8/s1600/wrapper-setup.png)

Now let's add some essential model classes - vertices and triangles. Vertex is simply a structure to store our three coordinates (X, Y and Z), and triangle binds together three vertices and stores its color.

```
class Vertex {
    double x;
    double y;
    double z;
    Vertex(double x, double y, double z) {
        this.x = x;
        this.y = y;
        this.z = z;
    }
}

class Triangle {
    Vertex v1;
    Vertex v2;
    Vertex v3;
    Color color;
    Triangle(Vertex v1, Vertex v2, Vertex v3, Color color) {
        this.v1 = v1;
        this.v2 = v2;
        this.v3 = v3;
        this.color = color;
    }
}
```

For this post, I'll assume that X coordinate means movement in left-right direction, Y means movement up-down on screen, and Z will be depth (so Z axis is perpendicular to your screen). Positive Z will mean "towards the observer".

As our example object, I selected tetrahedron, as it's the easiest 3d shape I could think of - only 4 triangles are needed to describe it. Here's the visualization:

[![img](http://1.bp.blogspot.com/-dKZe2K4H1c4/VbvtX-ihNeI/AAAAAAAAAEA/VTy7OhFfKjE/s320/tetrahedron.gif)](http://1.bp.blogspot.com/-dKZe2K4H1c4/VbvtX-ihNeI/AAAAAAAAAEA/VTy7OhFfKjE/s1600/tetrahedron.gif)

The code is very simple - we just create 4 triangles and add them to a list:

```
List tris = new ArrayList<>();
tris.add(new Triangle(new Vertex(100, 100, 100),
                      new Vertex(-100, -100, 100),
                      new Vertex(-100, 100, -100),
                      Color.WHITE));
tris.add(new Triangle(new Vertex(100, 100, 100),
                      new Vertex(-100, -100, 100),
                      new Vertex(100, -100, -100),
                      Color.RED));
tris.add(new Triangle(new Vertex(-100, 100, -100),
                      new Vertex(100, -100, -100),
                      new Vertex(100, 100, 100),
                      Color.GREEN));
tris.add(new Triangle(new Vertex(-100, 100, -100),
                      new Vertex(100, -100, -100),
                      new Vertex(-100, -100, 100),
                      Color.BLUE));
```

Resulting shape is centered at origin (0, 0, 0), which is quite convenient since we will be doing rotation around that point later.

Now let's put that on screen. For now, we'll ignore the rotation and will just show the wireframe. Since we are using orthographic projection, it's quite simple - just discard the Z coordinate and draw the resulting triangles.

```
g2.translate(getWidth() / 2, getHeight() / 2);
g2.setColor(Color.WHITE);
for (Triangle t : tris) {
    Path2D path = new Path2D.Double();
    path.moveTo(t.v1.x, t.v1.y);
    path.lineTo(t.v2.x, t.v2.y);
    path.lineTo(t.v3.x, t.v3.y);
    path.closePath();
    g2.draw(path);
}
```

Note how I applied translation before drawing all the triangles. That is done to put the origin (0, 0, 0) to the center of our drawing area - initially, 2d origin is located in top left corner of screen. Result should look like this:

You may not believe it yet, but that's our tetrahedron in orthographic projection, I promise!

Now we need to add rotation. To do that, I'll need to digress a little and talk about using matrices to achieve transformations on 3D points.

There are many possible ways to manipulate 3d points, but the most flexible is to use matrix multiplication. The idea is to represent your points as 3x1 vectors, and transformation is then simply multiplication by 3x3 matrix.

You take your input vector A:



*A*=[*a**x**a**y**a**z*]A=[axayaz]



and multiply it with transformation matrix T to get output vector B:



*A**T*=[*a**x**a**y**a**z*]⎡⎣⎢*t**x**x**t**y**x**t**z**x**t**x**y**t**y**y**t**z**y**t**x**z**t**y**z**t**z**z*⎤⎦⎥=[*a**x**t**x**x*+*a**y**t**y**x*+*a**z**t**z**x**a**x**t**x**y*+*a**y**t**y**y*+*a**z**t**z**y**a**x**t**x**z*+*a**y**t**y**z*+*a**z**t**z**z*]=[*b**x**b**y**b**z*]AT=[axayaz][txxtxytxztyxtyytyztzxtzytzz]=[axtxx+aytyx+aztzxaxtxy+aytyy+aztzyaxtxz+aytyz+aztzz]=[bxbybz]



For example, here's how you would scale a point by 2:



[123]⎡⎣⎢200020002⎤⎦⎥=[1×22×23×2]=[246][123][200020002]=[1×22×23×2]=[246]



You can't describe all possible transformations using 3x3 matrices - for example, translation is off-limits. You can achieve it with 4x4 matrices, effectively doing skew in 4D space, but that is beyond the scope of this tutorial.

Most useful transformations that we will need in this tutorial are scaling and rotating.

Any rotation in 3D space can be expressed as combination of 3 primitive rotations: rotation in XY plane, rotation in YZ plane and rotation in XZ plane. We can write out transformation matrices for each of those rotations as follows:

XY rotation matrix:



⎡⎣⎢*c**o**s**θ**s**i**n**θ*0−*s**i**n**θ**c**o**s**θ*0001⎤⎦⎥[cosθ−sinθ0sinθcosθ0001]



YZ rotation matrix:



⎡⎣⎢1000*c**o**s**θ*−*s**i**n**θ*0*s**i**n**θ**c**o**s**θ*⎤⎦⎥[1000cosθsinθ0−sinθcosθ]



XZ rotation matrix:





⎡⎣⎢*c**o**s**θ*0*s**i**n**θ*010−*s**i**n**θ*0*c**o**s**θ*⎤⎦⎥[cosθ0−sinθ010sinθ0cosθ]



Here comes the magic: if you need to first rotate a point in XY plane using transformation matrix *T*1T1, and then rotate it in YZ plane using transfromation matrix *T*2T2, you can simply multiply *T*1T1 with *T*2T2 and get a single matrix to describe the whole rotation:



(*A**T*1)*T*2=*A*(*T*1*T*2)(AT1)T2=A(T1T2)



This is a very useful optimization - instead of recomputing multiple rotations on each point, you precompute the matrix once and then use it in your pipeline.

Enough of the scary math stuff, let's get back to code. We will create utility class Matrix3 that will handle matrix-matrix and vector-matrix multiplication:

```
class Matrix3 {
    double[] values;
    Matrix3(double[] values) {
        this.values = values;
    }
    Matrix3 multiply(Matrix3 other) {
        double[] result = new double[9];
        for (int row = 0; row < 3; row++) {
            for (int col = 0; col < 3; col++) {
                for (int i = 0; i < 3; i++) {
                    result[row * 3 + col] +=
                        this.values[row * 3 + i] * other.values[i * 3 + col];
                }
            }
        }
        return new Matrix3(result);
    }
    Vertex transform(Vertex in) {
        return new Vertex(
            in.x * values[0] + in.y * values[3] + in.z * values[6],
            in.x * values[1] + in.y * values[4] + in.z * values[7],
            in.x * values[2] + in.y * values[5] + in.z * values[8]
        );
    }
}
```

Now we can bring to life our rotation sliders. The horizontal slider would control "heading" - in our case, rotation in XZ direction (left-right), and vertical slider will control "pitch" - rotation in YZ direction (up-down).

Let's create our rotation matrix and add it into our pipeline:

```
double heading = Math.toRadians(headingSlider.getValue());
Matrix3 transform = new Matrix3(new double[] {
        Math.cos(heading), 0, -Math.sin(heading),
        0, 1, 0,
        Math.sin(heading), 0, Math.cos(heading)
    });

g2.translate(getWidth() / 2, getHeight() / 2);
g2.setColor(Color.WHITE);
for (Triangle t : tris) {
    Vertex v1 = transform.transform(t.v1);
    Vertex v2 = transform.transform(t.v2);
    Vertex v3 = transform.transform(t.v3);
    Path2D path = new Path2D.Double();
    path.moveTo(v1.x, v1.y);
    path.lineTo(v2.x, v2.y);
    path.lineTo(v3.x, v3.y);
    path.closePath();
    g2.draw(path);
}
```

You'll also need to add a listeners on heading and pitch sliders to force redraw when you drag the handle:

```
headingSlider.addChangeListener(e -> renderPanel.repaint());
pitchSlider.addChangeListener(e -> renderPanel.repaint());
```

Here's what you should get working (this example is interactive - try dragging the handles!):

 

As you may have noticed, up-down rotation doesn't work yet. Let's add next transform:

```
Matrix3 headingTransform = new Matrix3(new double[] {
        Math.cos(heading), 0, Math.sin(heading),
        0, 1, 0,
        -Math.sin(heading), 0, Math.cos(heading)
    });
double pitch = Math.toRadians(pitchSlider.getValue());
Matrix3 pitchTransform = new Matrix3(new double[] {
        1, 0, 0,
        0, Math.cos(pitch), Math.sin(pitch),
        0, -Math.sin(pitch), Math.cos(pitch)
    });
Matrix3 transform = headingTransform.multiply(pitchTransform);
```

Observe that both rotations now work and combine together nicely:

 

Up to this point, we were only drawing the wireframe of our shape. Now we need to start filling up those triangles with some substance. To do this, we first need to "rasterize" the triangle - convert it to list of pixels on screen that it occupies.

I'll use relatively simple, but inefficient method - rasterization via [barycentric coordinates](https://en.wikipedia.org/wiki/Barycentric_coordinate_system). Real 3d engines use hardware rasterization, which is very fast and efficient, but we can't use the graphic card and so will be doing it manually in our code.

The idea is to compute barycentric coordinate for each pixel that could possibly lie inside the triangle and discard those that are outside. The following snippet implements the algorithm. Note how we started using direct access to image pixels.

```
BufferedImage img = 
    new BufferedImage(getWidth(), getHeight(), BufferedImage.TYPE_INT_ARGB);

for (Triangle t : tris) {
    Vertex v1 = transform.transform(t.v1);
    Vertex v2 = transform.transform(t.v2);
    Vertex v3 = transform.transform(t.v3);

    // since we are not using Graphics2D anymore,
    // we have to do translation manually
    v1.x += getWidth() / 2;
    v1.y += getHeight() / 2;
    v2.x += getWidth() / 2;
    v2.y += getHeight() / 2;
    v3.x += getWidth() / 2;
    v3.y += getHeight() / 2;

    // compute rectangular bounds for triangle
    int minX = (int) Math.max(0, Math.ceil(Math.min(v1.x, Math.min(v2.x, v3.x))));
    int maxX = (int) Math.min(img.getWidth() - 1, 
                              Math.floor(Math.max(v1.x, Math.max(v2.x, v3.x))));
    int minY = (int) Math.max(0, Math.ceil(Math.min(v1.y, Math.min(v2.y, v3.y))));
    int maxY = (int) Math.min(img.getHeight() - 1,
                              Math.floor(Math.max(v1.y, Math.max(v2.y, v3.y))));

    double triangleArea =
       (v1.y - v3.y) * (v2.x - v3.x) + (v2.y - v3.y) * (v3.x - v1.x);

    for (int y = minY; y <= maxY; y++) {
        for (int x = minX; x <= maxX; x++) {
            double b1 = 
              ((y - v3.y) * (v2.x - v3.x) + (v2.y - v3.y) * (v3.x - x)) / triangleArea;
            double b2 =
              ((y - v1.y) * (v3.x - v1.x) + (v3.y - v1.y) * (v1.x - x)) / triangleArea;
            double b3 =
              ((y - v2.y) * (v1.x - v2.x) + (v1.y - v2.y) * (v2.x - x)) / triangleArea;
            if (b1 >= 0 && b1 <= 1 && b2 >= 0 && b2 <= 1 && b3 >= 0 && b3 <= 1) {
                img.setRGB(x, y, t.color.getRGB());
            }
        }
    }

}

g2.drawImage(img, 0, 0, null);
```

Quite a lot of code, but now we have colored tetrahedron on our displays:

 

If you play around with the demo, you'll notice that not all is well - for example, blue triangle is always above others. It happens becase we are currently painting the triangles one after another, and blue triangle is last - thus it is painted over all others.

To fix this I will introduce the concept of z-buffer (or depth buffer). The idea is to build an intermediate array during rasterization that will store depth of last seen element at any given pixel. When rasterizing triangles, we will be checking that pixel depth is less than previously seen, and only color the pixel if it is above others.

```
double[] zBuffer = new double[img.getWidth() * img.getHeight()];
// initialize array with extremely far away depths
for (int q = 0; q < zBuffer.length; q++) {
    zBuffer[q] = Double.NEGATIVE_INFINITY;
}

for (Triangle t : tris) {
    // handle rasterization...
    // for each rasterized pixel:
    double depth = b1 * v1.z + b2 * v2.z + b3 * v3.z;
    int zIndex = y * img.getWidth() + x;
    if (zBuffer[zIndex] < depth) {
        img.setRGB(x, y, t.color.getRGB());
        zBuffer[zIndex] = depth;
    }
}
```

Now you can see that our tetrahedron actually has one white side:

 

We now have a functioning rendering pipeline!

But we are not finished here. In real life, perceived color of the surface varies with light source positions - if only a small amount of light is incident to the surface, we perceive that surface as being darker.

In computer graphics, we can achieve similar effect by using so-called "shading" - altering the color of the surface based on its angle and distance to lights.

Simplest form of shading is flat shading. It takes into account only the angle between surface normal and direction of the light source. You just need to find cosine of angle between those two vectors and multiply the color by the resulting value. Such approach is very simple and cheap, so it is often used for high-speed rendering when more advanced shading technologies are too computationally expensive.

First, we need to compute normal vector for our triangle. If we have triangle ABC, we can compute its normal vector by calculating cross product of vectors AB and AC and then dividing resulting vector by its length.

Cross product is a binary operation on two vectors that is defined in 3d space as follows:



*u*×*v*=[*u**x**u**y**u**z*]×[*v**x**v**y**v**z*]=[*u**y*×*v**z*−*u**z*×*v**y**u**z*×*v**x*−*u**x*×*v**z**u**x*×*v**y*−*u**y*×*v**x*]u×v=[uxuyuz]×[vxvyvz]=[uy×vz−uz×vyuz×vx−ux×vzux×vy−uy×vx]



Here's the visual explanation of what cross product does:

[![img](http://3.bp.blogspot.com/-cx5-0n6dbQ0/Vbvu74ppU_I/AAAAAAAAAEM/8FCk16u8I2Y/s400/cross_product_vector.png)](http://3.bp.blogspot.com/-cx5-0n6dbQ0/Vbvu74ppU_I/AAAAAAAAAEM/8FCk16u8I2Y/s1600/cross_product_vector.png)

```
for (Triangle t : tris) {
    // transform vertices before calculating normal...

    Vertex norm = new Vertex(
         ab.y * ac.z - ab.z * ac.y,
         ab.z * ac.x - ab.x * ac.z,
         ab.x * ac.y - ab.y * ac.x
    );
    double normalLength =
        Math.sqrt(norm.x * norm.x + norm.y * norm.y + norm.z * norm.z);
    norm.x /= normalLength;
    norm.y /= normalLength;
    norm.z /= normalLength;
}
```

Now we need to calculate cosine between triangle normal and light direction. For simplicity, we will assume that our light is positioned directly behind the camera at some infinite distance (such configuration is called "directional light") - so our light source direction will be [001][001].

Cosine of angle between vectors can be calculated using this formula:



*c**o**s**θ*=*A*⋅*B*||*A*||×||*B*||cosθ=A⋅B||A||×||B||



where ||*A*||||A|| is length of a vector, and *A*⋅*B*A⋅B is dot product of vectors:



*A*⋅*B*=[*a**x**a**y**a**z*]⋅[*b**x**b**y**b**z*]=*a**x*×*b**x*+*a**y*×*b**y*+*a**z*×*b**z*A⋅B=[axayaz]⋅[bxbybz]=ax×bx+ay×by+az×bz



Notice that length of our light direction vector ([001][001]) is 1, as well as the length of triangle normal (we already have normalized it). Thus the formula simply becomes:



*c**o**s**θ*=*A*⋅*B*=[*a**x**a**y**a**z*]⋅[*b**x**b**y**b**z*]cosθ=A⋅B=[axayaz]⋅[bxbybz]



Also observe that only Z component of light direction vector is non-zero, so we can simplify further:



*c**o**s**θ*=*A*⋅*B*=[*a**x**a**y**a**z*]⋅[001]=*a**z*cosθ=A⋅B=[axayaz]⋅[001]=az



The code is now trivial:

```
double angleCos = Math.abs(norm.z);
```

We drop the sign from the result because for our simple purposes we don't care which triangle side is facing the camera. In real application, you will need to keep track of that and apply shading accordingly.

Now that we have our shade coefficient, we can apply it to triangle color. Naive version may look as follows:

```
public static Color getShade(Color color, double shade) {
    int red = (int) (color.getRed() * shade);
    int green = (int) (color.getGreen() * shade);
    int blue = (int) (color.getBlue() * shade);
    return new Color(red, green, blue);
}
```

While it will give us some shading effect, it will have much quicker falloff than we need. That happens because Java uses sRGB color space, which is already scaled to match our logarithmic color perception.

So we need to convert each color from scaled to linear format, apply shade, and then convert back to scaled format. Real conversion from sRGB to linear RGB is [quite involved](https://en.wikipedia.org/wiki/SRGB#Specification_of_the_transformation), so I won't implement the full spec here - just the basic approximation.

```java
public static Color getShade(Color color, double shade) {
    double redLinear = Math.pow(color.getRed(), 2.4) * shade;
    double greenLinear = Math.pow(color.getGreen(), 2.4) * shade;
    double blueLinear = Math.pow(color.getBlue(), 2.4) * shade;

    int red = (int) Math.pow(redLinear, 1/2.4);
    int green = (int) Math.pow(greenLinear, 1/2.4);
    int blue = (int) Math.pow(blueLinear, 1/2.4);

    return new Color(red, green, blue);
}
```

Observe how our tetrahedron comes to life:

 

Now we have a working 3d render engine, with colors, lighting and shading, and it took us about 200 lines of code - not bad!

Here's one bonus for you - we can quickly create a sphere approximation from this tetrahedron. It can be done by repeatedly subdividing each triangle into four smaller ones and "inflating":

```java
public static List inflate(List tris) {
    List result = new ArrayList<>();
    for (Triangle t : tris) {
        Vertex m1 =
            new Vertex((t.v1.x + t.v2.x)/2, (t.v1.y + t.v2.y)/2, (t.v1.z + t.v2.z)/2);
        Vertex m2 =
            new Vertex((t.v2.x + t.v3.x)/2, (t.v2.y + t.v3.y)/2, (t.v2.z + t.v3.z)/2);
        Vertex m3 =
            new Vertex((t.v1.x + t.v3.x)/2, (t.v1.y + t.v3.y)/2, (t.v1.z + t.v3.z)/2);
        result.add(new Triangle(t.v1, m1, m3, t.color));
        result.add(new Triangle(t.v2, m1, m2, t.color));
        result.add(new Triangle(t.v3, m2, m3, t.color));
        result.add(new Triangle(m1, m2, m3, t.color));
    }
    for (Triangle t : result) {
        for (Vertex v : new Vertex[] { t.v1, t.v2, t.v3 }) {
            double l = Math.sqrt(v.x * v.x + v.y * v.y + v.z * v.z) / Math.sqrt(30000);
            v.x /= l;
            v.y /= l;
            v.z /= l;
        }
    }
    return result;
}
```

Here's what you will see:

 

You can find full source code for this app [here](https://gist.github.com/Rogach/f3dfd457d7ddb5fcfd99/4f2aaf20a468867dc195cdc08a02e5705c2cc95c). It's only 220 lines and has no dependencies - you can just compile and start it!

I will finish this article by recommending one awesome book: [3D Math Primer for Graphics and Game Development](http://www.amazon.com/Math-Primer-Graphics-Development-Edition/dp/1568817231). It explains all the details of rendering pipelines and math involved - definitely a worthy read if you are interested in rendering engines.

Hope this article was useful! If you found some parts to be confusing, please leave a comment, and I will do my best to provide better and more detailed explanations.