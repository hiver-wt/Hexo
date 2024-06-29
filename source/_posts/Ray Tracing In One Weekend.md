---
title: Ray Tracing In One Weekend
date: 2024-06-28 18:45:41
tags: CG
categories: CG
description: Ray Tracing In One Weekend
swiper_index: 3 #置顶轮播图顺序，非负整数，数字越大越靠前
---
# 2 Output an Image
## 2.1 The PPM Image Format

<img src="/RTimage/Pasted image 20240201173749.png" alt="示例图片" style="zoom:50%;" />

Output such a thing
```c++
#include <iostream>
int main() 
{
    // Image
    int image_width = 256;
    int image_height = 256;
    
    // Render
    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";
    for (int j = 0; j < image_height; ++j)
     {
        for (int i = 0; i < image_width; ++i) 
        {
            auto r = double(i) / (image_width-1);
            auto g = double(j) / (image_height-1);
            auto b = 0;

            int ir = static_cast<int>(255.999 * r);
            int ig = static_cast<int>(255.999 * g);
            int ib = static_cast<int>(255.999 * b);

            std::cout << ir << ' ' << ig << ' ' << ib << '\n';
        }
    }
}
```
Some NOTE
1. The pixels are wirtten out in rows.
2. Every row of pixels is written out left to right.
3. These rows are written out from top to bottom.
4. By convention, each of the red/green/blue components are represented internally by real-valued variables that range from 0.0 to 1.0. These must be scaled to integer values between 0 and 255 before we print them out.
5. When we calculate the color value later, we will use a dynamic range, which is not 0 to 1. 
6.  Red goes from fully off (black) to fully on (bright red) from left to right, and green goes from fully off at the top to black at the bottom. Adding red and green light together make yellow so we should expect the bottom right corner to be yellow.
## 2.2 Creating an Image File

Now we need to write cout's output stream to a file. Fortunately we have the command line operator `>` to direct the output stream. In the Windows operating system, it looks like this:

<img src="/blog/_posts/RTimage/Pasted image 20240202131111.png" alt="示例图片" style="zoom:50%;" />

First PPM image

<img src="/blog/_posts/RTimage/Pasted image 20240202131653.png" alt="示例图片" style="zoom:50%;" />

open it a text editor and see
```c++
P3
256 256
255
0 0 0
1 0 0
2 0 0
3 0 0
4 0 0
5 0 0
...
```
## 2.3 Adding a Progress Indicator
A Progress Indicator：This is a handy way to track the progress of a long render, and also to possibly identify a run that's stalled out due to an infinite loop or other problem.

Our program outputs the image to the standard output stream (`std::cout`), so leave that alone and instead write to the logging output stream (`std::clog`):
PS：the 3.0v use `std::cerr`
- `std::cerr` uses an unbuffered stream, and each output operation is output immediately; 
- `std::clog` uses a buffered stream, which is line buffering by default, and will flush the buffer when a new line arrives or the buffer is full, thereby outputting the data in the buffer. 
Therefore, cerr is used to output real-time information, and clog is used to output cached log information
```c++
for (int j = 0; j < image_height; ++j)
{        
	std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;       
	for (int i = 0; i < image_width; ++i) 
	{
		auto r = double(i) / (image_width-1);
		auto g = double(j) / (image_height-1);
		auto b = 0;

		int ir = static_cast<int>(255.999 * r);
		int ig = static_cast<int>(255.999 * g);
		int ib = static_cast<int>(255.999 * b);

		std::cout << ir << ' ' << ig << ' ' << ib << '\n';
    }
}
std::clog << "\rDone.                 \n";
```
# 3 The Vec3 Class
Almost all graphics programs use 4D vectors(3D position plus a homogeneous coordinate for geometry, or RGB plus an alpha transparency component for colors). For our purposes, three coordinates suffice.
we declare two aliases for `vec3`: `point3` and `color`. Since these two types are just aliases for `vec3`, you won't get warnings if you pass a `color` to a function expecting a `point3`, and nothing is stopping you from adding a `point3` to a `color`, but it makes the code a little bit easier to read and to understand.

```c++
#pragma once
#include<cmath>
#include<iostream>
class vec3
{
public:
    double e[3];

    vec3(): e{0,0,0} {}
    vec3(double e0, double e1, double e2): e{e0, e1, e2} {}

    // 返回向量的 x、y、z 坐标的成员函数
    double x() const { return e[0]; }
    double y() const { return e[1]; }
    double z() const { return e[2]; }

    // 重载一元负运算符，返回当前向量的负向量
    vec3 operator-() const { return vec3(-e[0], -e[1], -e[2]); }

    // 重载下标运算符，允许通过下标访问向量的各个坐标分量
    // 返回索引i处元素的值，但不允许修改vec3对象。
    double operator[](int i) const { return e[i]; }
    // 可以通过下标修改向量的元素
    double& operator[](int i) { return e[i]; }

    vec3& operator+=(const vec3& v)
    {
        e[0] += v.e[0];
        e[1] += v.e[1];
        e[2] += v.e[2];
        return *this;
    }

    vec3& operator*=(const double t)
    {
        e[0] *= t;
        e[1] *= t;
        e[2] *= t;
        return *this;
    }

    vec3& operator/=(const double t)
    {
        return *this *= 1 / t;
    }

    double length() const
    {
        return sqrt(length_squared());
    }

    double length_squared() const
    {
        return e[0] * e[0] + e[1] * e[1] + e[2] * e[2];
    }

    void write_color(std::ostream& out)
    {
        // Write the translated [0,255] value of each color component.
        out << static_cast<int>(255.999 * e[0]) << ' '
            << static_cast<int>(255.999 * e[1]) << ' '
            << static_cast<int>(255.999 * e[2]) << '\n';
    }
};
// point3 is just an alias for vec3, but useful for geometric clarity in the code.
using point3 = vec3;

inline std::ostream& operator<<(std::ostream& out, const vec3& v)
{
    return out << v.e[0] << ' ' << v.e[1] << ' ' << v.e[2];
}

inline vec3 operator+(const vec3& u, const vec3& v)
{
    return vec3(u.e[0] + v.e[0], u.e[1] + v.e[1], u.e[2] + v.e[2]);
}

inline vec3 operator-(const vec3& u, const vec3& v)
{
    return vec3(u.e[0] - v.e[0], u.e[1] - v.e[1], u.e[2] - v.e[2]);
}

inline vec3 operator*(const vec3& u, const vec3& v)
{
    return vec3(u.e[0] * v.e[0], u.e[1] * v.e[1], u.e[2] * v.e[2]);
}

inline vec3 operator*(double t, const vec3& v)
{
    return vec3(t * v.e[0], t * v.e[1], t * v.e[2]);
}

inline vec3 operator*(const vec3& v, double t)
{
    return t * v;
}

inline vec3 operator/(vec3 v, double t)
{
    return (1 / t) * v;
}

inline double dot(const vec3& u, const vec3& v)
{
    return u.e[0] * v.e[0]
        + u.e[1] * v.e[1]
        + u.e[2] * v.e[2];
}

inline vec3 cross(const vec3& u, const vec3& v)
{
    return vec3(u.e[1] * v.e[2] - u.e[2] * v.e[1],
                u.e[2] * v.e[0] - u.e[0] * v.e[2],
                u.e[0] * v.e[1] - u.e[1] * v.e[0]);
}

inline vec3 unit_vector(vec3 v)
{
    return v / v.length();
}
```
## 3.1 Color Utility Functions
Using our new `vec3` class, we'll create a new `color.h` header file and define a utility function that writes a single pixel's color out to the standard output stream.
```c++
#pragma once
#include "vec3.h"
#include<iostream>

using color = vec3;

void writeColor(std::ostream& out, color pixel_color)
{
    // Write the translated [0,255] value of each color component.
    out << static_cast<int>(255.999 * pixel_color.x()) << ' '
        << static_cast<int>(255.999 * pixel_color.y()) << ' '
        << static_cast<int>(255.999 * pixel_color.z()) << '\n';
}
```

Now we can change our main to use both of these:
```c++
#include "color.h"
#include "vec3.h"
#include <iostream>

int main() 
{
    // Image
    int image_width = 256;
    int image_height = 256;
    // Render
    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";
    for (int j = 0; j < image_height; ++j) 
    {
        std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
        for (int i = 0; i < image_width; ++i) 
        {            
	        auto pixel_color = color(double(i)/(image_width-1), double(j)/(image_height-1), 0);
            writeColor(std::cout, pixel_color);        
        }
    }
    std::clog << "\rDone.                 \n";
}
```
And you should get the exact same picture as before.

# 4 Rays, a Simple Camera, and Background
## 4.1 The ray Class
The one thing that all ray tracers have is a ray class and a computation of what color is seen along a ray. Let’s think of a ray as a function $P(t)=A+t\vec b$ Here $P$ is a 3D position along a line in 3D. $A$ is the ray origin and $\vec b$ is the ray direction.
The ray parameter $t$ is a real number (`double` in the code).Plug in a different $t$ and $P(t)$ moves the point along the ray.
Add in negative $t$ values and you can go anywhere on the 3D line. For positive $t$, you get only the parts in front of $A$, and this is what is often called a half-line or a ray.
![[Pasted image 20240202155603.png]]
We can represent the idea of a ray as a class, and represent the function $P(t)$ as a function that we'll call `ray::at(t)`:
```c++
#ifndef RAY_H
#define RAY_H

#include "vec3.h"

class ray
{
public:
    ray() {}
    ray(const point3& origin, const vec3& direction)
        : orig(origin), dir(direction)
    {
    }

    point3 origin() const { return orig; }
    vec3 direction() const { return dir; }

    point3 at(double t) const
    {
        return orig + t * dir;
    }

public:
    point3 orig;
    vec3 dir;
};
#endif

```
## 4.2 Sending Rays Into the Scene
Now we are ready to turn the corner and make a ray tracer. At its core, a ray tracer sends rays through pixels and computes the color seen in the direction of those rays. The involved steps are

1. Calculate the ray from the “eye” through the pixel,
2. Determine which objects the ray intersects, and
3. Compute a color for the closest intersection point.

When first developing a ray tracer, I always do a simple camera for getting the code up and running.

The image's aspect ratio can be determined from the ratio of its height to its width. However, since we have a given aspect ratio in mind, it's easier to set the image's width and the aspect ratio, and then using this to calculate for its height. This way, we can scale up or down the image by changing the image width, and it won't throw off our desired aspect ratio. We do have to make sure that when we solve for the image height the resulting height is at least 1.

In addition to setting up the pixel dimensions for the rendered image, we also need to set up a virtual _viewport_ through which to pass our scene rays. The viewport is a virtual rectangle in the 3D world that contains the grid of image pixel locations. If pixels are spaced the same distance horizontally as they are vertically, the viewport that bounds them will have the same aspect ratio as the rendered image. The distance between two adjacent pixels is called the pixel spacing, and square pixels is the standard.

To start things off, we'll choose an arbitrary viewport height of 2.0, and scale the viewport width to give us the desired aspect ratio.
```c++
auto aspect_ratio = 16.0 / 9.0;
int image_width = 400;

// Calculate the image height, and ensure that it's at least 1.
int image_height = static_cast<int>(image_width / aspect_ratio);
image_height = (image_height < 1) ? 1 : image_height;

// Viewport widths less than one are ok since they are real valued ( double type ).
auto viewport_height = 2.0;
auto viewport_width = viewport_height * (static_cast<double>(image_width)/image_height);
```
Why we don't just use `aspect_ratio` when computing `viewport_width`：
- because the value set to `aspect_ratio` is the ideal ratio, it may not be the _actual_ ratio between `image_width` and `image_height`. If `image_height` was allowed to be real valued—rather than just an integer—then it would fine to use `aspect_ratio`. But the _actual_ ratio between `image_width` and `image_height` can vary based on two parts of the code. First, `integer_height` is rounded down to the nearest integer, which can increase the ratio. Second, we don't allow `integer_height` to be less than one, which can also change the actual aspect ratio.
- Note that `aspect_ratio` is an ideal ratio, which we approximate as best as possible with the integer-based ratio of image width over image height. In order for our viewport proportions to exactly match our image proportions, we use the calculated image aspect ratio to determine our final viewport width.

Next we will define the camera center: a point in 3D space from which all scene rays will originate. The vector from the camera center to the viewport center will be orthogonal（_vertical_） to the viewport. We'll initially set the distance between the viewport and the camera center point to be one unit. This distance is often referred to as the _focal length_.

For simplicity we'll start with the camera center at **(0,0,0)**. We'll also have the y-axis go up, the x-axis to the right, and the negative z-axis pointing in the viewing direction. (This is commonly referred to as _right-handed coordinates_.)
![[Pasted image 20240202163250.png]]

As we scan our image, we will start at the upper left pixel (pixel 0,0), scan left-to-right across each row, and then scan row-by-row, top-to-bottom. To help navigate the pixel grid, we'll use a vector from the left edge to the right edge ($V_u$), and a vector from the upper edge to the lower edge ($V_v$).

Our pixel grid will be inset from the viewport edges by half the pixel-to-pixel distance.
_(By indenting the pixel grid by half a pixel, you ensure that the center of each pixel is in the center of the area occupied by the pixel)_
This way, our viewport area is evenly divided into width × height identical regions. Here's what our viewport and pixel grid look like:
![[Pasted image 20240202165123.png]]
Drawing from all of this, here's the code that implements the camera. We'll stub in a function `RayColor(const ray& r)` that returns the color for a given scene ray — which we'll set to always return black for now.
```c++
#include "color.h"
#include "vec3.h"
#include"ray.h"

#include <iostream>
color RayColor(const ray& r)
{
    return color(0, 0, 0);
}
int main()
{
    // Image
    auto aspect_ratio = 16.0 / 9.0;
    int image_width = 400;

    // Calculate the image height, and ensure that it's at least 1.
    int image_height = static_cast<int>(image_width / aspect_ratio);
    image_height = (image_height < 1) ? 1 : image_height;

    // Camera
    auto focal_length = 1.0;
    auto viewport_height = 2.0;
    auto viewport_width = viewport_height * (static_cast<double>(image_width) / image_height);
    auto camera_center = point3(0, 0, 0);

    // Calculate the vectors across the horizontal and down the vertical viewport edges.
    auto viewport_u = vec3(viewport_width, 0, 0);
    auto viewport_v = vec3(0, -viewport_height, 0);

    // Calculate the horizontal and vertical delta vectors from pixel to pixel.
    auto pixel_delta_u = viewport_u / image_width;
    auto pixel_delta_v = viewport_v / image_height;

    // Calculate the location of the upper left pixel.
    auto viewport_upper_left = camera_center
        - vec3(0, 0, focal_length) - viewport_u / 2 - viewport_v / 2;
    auto pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);

    // Render
    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";
    for (int j = 0; j < image_height; ++j)
    {
        std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
        for (int i = 0; i < image_width; ++i)
        {
            auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);
            auto ray_direction = pixel_center - camera_center;
            ray r(camera_center, ray_direction);

            color pixel_color = RayColor(r);
            writeColor(std::cout, pixel_color);
        }
    }
    std::clog << "\rDone.                 \n";
}
```
Notice：
I didn't make `ray_direction` a unit vector, because I think not doing that makes for simpler and slightly faster code.

Now we'll fill in the `ray_color(ray)` function to implement a simple gradient. This function will linearly blend white and blue depending on the height of the $y$ coordinate _after_ scaling the ray direction to unit length (so $−1.0<y<1.0$). Because we're looking at the $y$ height after normalizing the vector, you'll notice a horizontal gradient to the color in addition to the vertical gradient.

I'll use a standard graphics trick to linearly scale $-1.0 ≤ a ≤ 1.0$，When $a = 1.0$，I want blue. When $a = 0.0$ , I want white. In between, I want a blend. This forms a “linear blend”, or “linear interpolation”. This is commonly referred to as a _lerp_ between two values. A lerp is always of the form
$$blendedValue=(1−a)⋅startValue+a⋅endValue,$$
with $a$ going from zero to one.

Putting all this together, here's what we get:
```c++
color ray_color(const ray& r) 
{    
	vec3 unit_direction = unit_vector(r.direction());
	auto a = 0.5*(unit_direction.y() + 1.0);
	return (1.0-a)*color(1.0, 1.0, 1.0) + a*color(0.5, 0.7, 1.0);
}
```
![[Pasted image 20240202190821.png]]

# 5 Adding a Sphere
## 5.1 Ray-Sphere Intersection
The equation for a sphere of radius $r$ that is centered at the origin is an important mathematical equation:
$$x^2+y^2+z^2=r^2$$
If a given point $(x,y,z)$ is _inside_ the sphere, then  $x^2+y^2+z^2<r^2$ 
If a given point $(x,y,z)$ is _outside_ the sphere, then $x^2+y^2+z^2>r^2$.

If we want to allow the sphere center to be at an arbitrary point $(C_x, C_y,C_z)$, then the equation becomes a lot less nice:
$$(x−C_x)^2+(y−C_y)^2+(z−C_z)^2=r^2$$

You might note that the vector from center $C=(Cx,Cy,Cz)$ to point $P=(x,y,z)$ is $(P−C).$ If we use the definition of the dot product:
$$(P−C)⋅(P−C)=(x−C_x)^2+(y−C_y)^2+(z−C_z)^2$$

Then we can rewrite the equation of the sphere in vector form as:
$$(P−C)⋅(P−C)=r^2$$
We can read this as “any point P that satisfies this equation is on the sphere”. We want to know if our ray $P(t)=A+t\vec b$ ever hits the sphere anywhere. If it does hit the sphere, there is some $t$ for which $P(t)$ satisfies the sphere equation. So we are looking for any $t$ where this is true:
$$(P(t)−C)⋅(P(t)−C)=r^2$$

which can be found by replacing $P(t)$ with its expanded form:
$$((A+t\vec b)−C)⋅((A+t\vec b)−C)=r^2$$
we want to solve for $t$, so we'll separate the terms based on whether there is a $t$ or not:
$$(t\vec b+(A−C))⋅(t\vec b+(A−C))=r^2$$
And now we follow the rules of vector algebra to distribute the dot product and move the square of the radius over to the left hand side:
$$t^2\vec b⋅\vec b+2t\vec b⋅\vec{(A−C)}+\vec{(A−C)}⋅\vec{(A−C)}−r^2=0$$

The only unknown is $t$, and we have a $t^2$, which means that this equation is quadratic. You can solve for a quadratic equation by using the quadratic formula:
![[Pasted image 20240203131027.png]]

Use the root formula to determine the number of intersections. If it is positive, there will be two intersections. If it is negative, there will be no intersections. If it is 0, there will be one intersection
![[Pasted image 20240203131114.png]]

## 5.2 Creating Our First Raytraced Image
We can test our code by placing a small sphere at −1 on the z-axis and then coloring red any pixel that intersects it.
```c++
bool hitSphere(const point3& center, double radius, const ray& r)
{
    vec3 oc = r.origin() - center;
    auto a = dot(r.direction(), r.direction());
    auto b = 2.0 * dot(oc, r.direction());
    auto c = dot(oc, oc) - radius * radius;
    auto discriminant = b * b - 4 * a * c;
    return (discriminant >= 0);
}
color RayColor(const ray& r)
{
    if (hitSphere(point3(0, 0, -1), 0.5, r))
    {
        return color(0.5, 0.9, 1.0);
    }
    vec3 unit_direction = unit_vector(r.direction());
    auto a = 0.5 * (unit_direction.y() + 1.0);
    return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}
```

![[Pasted image 20240203133741.png]]

One thing to be aware of is that we are testing to see if a ray intersects with the sphere by solving the quadratic equation and seeing if a solution exists, but solutions with negative values of $t$ work just fine. If you change your sphere center to $z = 1$ you will get exactly the same picture because this solution doesn't distinguish between objects _in front of the camera_ and objects _behind the camera_. This is not a feature! We’ll fix those issues next.

# 6 Surface Normals and Multiple Objects
## 6.1 Shading with Surface Normals
First, let’s get ourselves a surface normal so we can shade. This is a vector that is perpendicular(_vertical_) to the surface at the point of intersection.

For a sphere, the outward normal is in the direction of the hit point minus the center:
(_sphere normals can be made unit length simply by dividing by the sphere radius_)
![[Pasted image 20240203140927.png]]

We don’t have any lights or anything yet, so let’s just visualize the normals with a color map. A common trick used for visualizing normals(_because it’s easy and somewhat intuitive to assume $\vec n$ is a unit length vector — so each component is between −1 and 1_)is to map each component to the interval from 0 to 1, and then map $(x,y,z)$to $(red,green,blue)$

For the normal, we need the hit point, not just whether we hit or not. We only have one sphere in the scene, and it's directly in front of the camera, so we won't worry about negative values of $t$ yet. We'll just assume the closest hit point (smallest $t$) is the one that we want. These changes in the code let us compute and visualize $\vec n$:
```c++
double hitSphere(const point3& center, double radius, const ray& r)
{
    vec3 oc = r.origin() - center;
    auto a = dot(r.direction(), r.direction());
    auto b = 2.0 * dot(oc, r.direction());
    auto c = dot(oc, oc) - radius * radius;
    auto discriminant = b * b - 4 * a * c;
    if (discriminant < 0)
        return -1.0;
    else
        return (-b - sqrt(discriminant)) / (2.0 * a); //smallest
}
color RayColor(const ray& r)
{
    auto t = hitSphere(point3(0, 0, -1), 0.5, r);
    if (t > 0.0)
    {
        vec3 N = unit_vector(r.at(t) - vec3(0, 0, -1));
        return 0.5 * color(N.x() + 1, N.y() + 1, N.z() + 1);
    }
    vec3 unit_direction = unit_vector(r.direction());
    auto a = 0.5 * (unit_direction.y() + 1.0);
    return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}
```
## 6.2 Simplifying the Ray-Sphere Intersection Code

Let’s revisit the ray-sphere function:
```c++
double hitSphere(const point3& center, double radius, const ray& r)
{
    vec3 oc = r.origin() - center;
    auto a = dot(r.direction(), r.direction());
    auto b = 2.0 * dot(oc, r.direction());
    auto c = dot(oc, oc) - radius * radius;
    auto discriminant = b * b - 4 * a * c;
    if (discriminant < 0)
    {
        return -1.0;
    }
    else
    {
        return (-b - sqrt(discriminant)) / (2.0 * a); //smallest
    }
}
```
First, recall that a vector dotted with itself is equal to the squared length of that vector.

Second, notice how the equation for `b` has a factor of two in it. Consider what happens to the quadratic equation if $b=2h$:
![[Pasted image 20240203143401.png]]
Using these observations, we can now simplify the sphere-intersection code to this:
(_Personally, I don't feel it's necessary? The original code is better to understand_)
```c++
double hit_sphere(const point3& center, double radius, const ray& r) 
{
    vec3 oc = r.origin() - center;    auto a = r.direction().length_squared();
    auto half_b = dot(oc, r.direction());
    auto c = oc.length_squared() - radius*radius;
    auto discriminant = half_b*half_b - a*c;
    if (discriminant < 0) 
        return -1.0;
    else
	     return (-half_b - sqrt(discriminant) ) / a;    
}
```
## 6.3 An Abstraction for Hittable Objects
Now, how about more than one sphere? While it is tempting to have an array of spheres, a very clean solution is to make an “abstract class” for anything a ray might hit, and make both a sphere and a list of spheres just something that can be hit.

This `hittable` abstract class will have a `hit` function that takes in a ray. Most ray tracers have found it convenient to add a valid interval for hits $t_{min}$ to $t_{max}$, so the hit only “counts” if $t_{min} <t<t_{max}$ 

For the initial rays this is positive $t$, but as we will see, it can simplify our code to have an interval $t_{min}$ to $t_{max}$ 

We might end up hitting something closer as we do our search, and we will only need the normal of the closest thing.I will go with the simple solution and compute a bundle of stuff I will store in some structure. Here’s the abstract class:
_hittable.h_
```c++
#pragma once
#include"ray.h"
class hit_record
{
public:
    point3 p;
    vec3 normal;
    double t;
};
class hittable
{
public:
    //  虚析构函数:通常用于确保在继承体系中正确地释放资源
    virtual ~hittable() = default;
    //  纯虚函数，在基类中声明但没有提供实现，需要在派生类中实现
    virtual bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const = 0;
};

```
And here’s the sphere:
_sphere.h_
```c++
#pragma once
#include "hittable.h"
#include "vec3.h"
class sphere:public hittable
{
public:
    sphere(point3 _center, double _radius): center(_center), radius(_radius) {}
    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const override
    {
        vec3 oc = r.origin() - center;
        auto a = r.direction().length_squared();
        auto half_b = dot(oc, r.direction());
        auto c = oc.length_squared() - radius * radius;

        auto discriminant = half_b * half_b - a * c;
        if (discriminant < 0) return false;
        auto sqrtd = sqrt(discriminant);

        // Find the nearest root that lies in the acceptable range.
        auto root = (-half_b - sqrtd) / a;
        if (root <= ray_tmin || ray_tmax <= root)
        {
            root = (-half_b + sqrtd) / a;
            if (root <= ray_tmin || ray_tmax <= root)
                return false;
        }

        rec.t = root;
        rec.p = r.at(rec.t);//光线与物体相交时的交点位置
        rec.normal = (rec.p - center) / radius;

        return true;
    }
private:
    point3 center;
    double radius;
};

```
## 6.4 Front Faces Versus Back Faces
The second design decision for normals is whether they should always point out.
Alternatively, we can have the normal always point against the ray. If the ray is outside the sphere, the normal will point outward, but if the ray is inside the sphere, the normal will point inward.
![[Pasted image 20240203152734.png]]

We need to choose one of these possibilities because we will eventually want to determine which side of the surface that the ray is coming from. This is important for objects that are rendered differently on each side, like the text on a two-sided sheet of paper, or for objects that have an inside and an outside, like glass balls.

We can figure this out by comparing the ray with the normal. If the ray and the normal face in the same direction, the ray is inside the object, if the ray and the normal face in the opposite direction, then the ray is outside the object. 
This can be determined by taking the dot product of the two vectors, where if their dot is positive, the ray is inside the sphere.
_Comparing the ray and the normal_
```c++
if (dot(ray_direction, outward_normal) > 0.0) //cos(θ)>90°
{
    // ray is inside the sphere
    ...
} 
else 
{
    // ray is outside the sphere
    ...
}
```

If we decide to have the normals always point against the ray, we won't be able to use the dot product to determine which side of the surface the ray is on. Instead, we would need to store that information:
_Remembering the side of the surface_
```c++
bool front_face;
if (dot(ray_direction, outward_normal) > 0.0) 
{
    // ray is inside the sphere
    normal = -outward_normal;
    front_face = false;
} 
else 
{
    // ray is outside the sphere
    normal = outward_normal;
    front_face = true;
}
```
We can set things up so that normals always point “outward” from the surface, or always point against the incident ray. This decision is determined by whether you want to determine the side of the surface at the time of geometry intersection or at the time of coloring.

In this book we have more material types than we have geometry types, so we'll go for less work and put the determination at geometry time. This is simply a matter of preference, and you'll see both implementations in the literature.

We add the `front_face` bool to the `hit_record` class. We'll also add a function to solve this calculation for us: `set_face_normal()`. For convenience we will assume that the vector passed to the new `set_face_normal()` function is of unit length. 

We could always normalize the parameter explicitly, but it's more efficient if the geometry code does this, as it's usually easier when you know more about the specific geometry.
_hittable.h_ 
```c++
class hit_record
{
public:
    point3 p;
    vec3 normal;
    double t;

    bool front_face;
    void setFaceNormal(const ray& r, const vec3& outward_normal)
    {
        // Sets the hit record normal vector.
        // NOTE: the parameter `outward_normal` is assumed to have unit length.
        front_face = dot(r.direction(), outward_normal) < 0;
        //根据front_face的值选择。如果是正面击中，法向量就是 `outward_normal`
        normal = front_face ? outward_normal : -outward_normal;
    }
};
```
And then we add the surface side determination to the class:
```c++
class sphere : public hittable 
{
  public:
    ...
    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const 
    {
        ...
        rec.t = root;
        rec.p = r.at(rec.t);        
        vec3 outward_normal = (rec.p - center) / radius;
        rec.setFaceNormal(r, outward_normal);
        return true;
    }
    ...
};
```
## 6.5 A List of Hittable Objects
We have a generic object called a `hittable` that the ray can intersect with. We now add a class that stores a list of `hittables`:
_hittable_list.h_ 
```c++
#pragma once
#include"hittable.h"
#include<memory>
#include<vector>

using std::shared_ptr;
using std::make_shared;

class hittable_list: public hittable
{
public:
    std::vector<shared_ptr<hittable>> objects;

    hittable_list() {}
    hittable_list(shared_ptr<hittable> object) { add(object); }

    void clear() { objects.clear(); }

    void add(shared_ptr<hittable> object)
    {
        objects.push_back(object);
    }

    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const override
    {
        hit_record temp_rec;
        bool hit_anything = false;
        auto closest_so_far = ray_tmax;

        for (const auto& object : objects)
        {
            if (object->hit(r, ray_tmin, closest_so_far, temp_rec))
            {
                hit_anything = true;
                closest_so_far = temp_rec.t;
                rec = temp_rec;
            }
        }
        return hit_anything;
    }
};
```
## 6.6 Some New C++ Features
[[C++#share_ptr]]
[[C++#std vector]]

## 6.7 Common Constants and Utility Functions
```c++
#pragma once
#include <cmath>
#include <limits>
#include <memory>

// Usings

using std::shared_ptr;
using std::make_shared;
using std::sqrt;

// Constants
const double infinity = std::numeric_limits<double>::infinity();//  正无穷大
const double pi = 3.1415926535897932385;

// Utility Functions

inline double degrees_to_radians(double degrees)
{
    return degrees * pi / 180.0;
}

// Common Headers

#include "ray.h"
#include "vec3.h"
```
The new main.cpp
```c++
#include "rtweekend.h"

#include "color.h"
#include "hittable.h"
#include "hittable_list.h"
#include "sphere.h"

#include <iostream>
color RayColor(const ray& r, const hittable& world)
{
    hit_record rec;
    if (world.hit(r, 0, infinity, rec))
    {
        return 0.5 * (rec.normal + color(1, 1, 1));
    }
    vec3 unit_direction = unit_vector(r.direction());
    auto a = 0.5 * (unit_direction.y() + 1.0);
    return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}
int main()
{
    // Image
    auto aspect_ratio = 16.0 / 9.0;
    int image_width = 400;

    // Calculate the image height, and ensure that it's at least 1.
    int image_height = static_cast<int>(image_width / aspect_ratio);
    image_height = (image_height < 1) ? 1 : image_height;

    // World

    hittable_list world;

    world.add(make_shared<sphere>(point3(0, 0, -1), 0.5));
    world.add(make_shared<sphere>(point3(0, -100.5, -1), 100));

    // Camera
    auto focal_length = 1.0;
    auto viewport_height = 2.0;
    auto viewport_width = viewport_height * (static_cast<double>(image_width) / image_height);
    auto camera_center = point3(0, 0, 0);

    // Calculate the vectors across the horizontal and down the vertical viewport edges.
    auto viewport_u = vec3(viewport_width, 0, 0);
    auto viewport_v = vec3(0, -viewport_height, 0);

    // Calculate the horizontal and vertical delta vectors from pixel to pixel.
    auto pixel_delta_u = viewport_u / image_width;
    auto pixel_delta_v = viewport_v / image_height;

    // Calculate the location of the upper left pixel.
    auto viewport_upper_left = camera_center
        - vec3(0, 0, focal_length) - viewport_u / 2 - viewport_v / 2;
    auto pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);

    // Render
    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

    for (int j = 0; j < image_height; ++j)
    {
        std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
        for (int i = 0; i < image_width; ++i)
        {
            auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);
            auto ray_direction = pixel_center - camera_center;
            ray r(camera_center, ray_direction);

            color pixel_color = RayColor(r, world);
            writeColor(std::cout, pixel_color);
        }
    }
    std::clog << "\rDone.                 \n";
}
```
_注意：world调用add之后传入objects的是sphere类型，所有object调用的是sphere的hit_

## 6.8 An Interval Class
Before we continue, we'll implement an interval class to manage real-valued intervals with a minimum and a maximum. We'll end up using this class quite often as we proceed.
_interval.h_
```c++
#pragma once

class interval
{
public:
    double min, max;
    interval():min(+infinity), max(-infinity) {}
    interval(double _min, double _max): min(_min), max(_max) {}
    bool contains(double x) const
    {
        return min <= x && x <= max;
    }

    bool surrounds(double x) const
    {
        return min < x&& x < max;
    }

    static const interval empty, universe;
};

const static interval empty(+infinity, -infinity);
const static interval universe(-infinity, +infinity);
```
_rtweekend.h_
```c++
// Common Headers
#include "interval.h"         //双向引用
#include "ray.h"
#include "vec3.h"
```
_hittable.h_
![[Pasted image 20240204130929.png]]
_hittable_list.h_
![[Pasted image 20240204130953.png]]
_sphere.h_ 
![[Pasted image 20240204131007.png]]
_main.cpp_
![[Pasted image 20240204131049.png]]

# 7 Moving Camera Code Into Its Own Class
The camera class will be responsible for two important jobs:

1. Construct and dispatch rays into the world.
2. Use the results of these rays to construct the rendered image.

The skeleton of our new `camera` class:
_camera.h_
```c++
#pragma once
#include "rtweekend.h"
#include "color.h"
#include "hittable.h"
class camera
{
public:
    /* Public Camera Parameters Here */
    void render(const hittable& world) { }
private:
    /* Private Camera Variables Here */
    void initialize( ) { }
    color ray_color(const ray& r, const hittable& world) const { }
};
```
To begin with, let's fill in the `ray_color()` function from `main.cpp`:
```c++
color ray_color(const ray& r, const hittable& world) const
{
	hit_record rec;
	if (world.hit(r, interval(0, infinity), rec))
	{
		return 0.5 * (rec.normal + color(1, 1, 1));
	}
	vec3 unit_direction = unit_vector(r.direction());
	auto a = 0.5 * (unit_direction.y() + 1.0);
	return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}
```
Now we move almost everything from the `main()` function into our new camera class. The only thing remaining in the `main()` function is the world construction. Here's the camera class with newly migrated code:
_camera.h_
```c++
#pragma once
#include "rtweekend.h"
#include "color.h"
#include "hittable.h"
class camera
{
public:
    double aspect_ratio = 1.0;
    int image_width = 400;
    void render(const hittable& world)
    {
        initialize();
        std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";
        for (int j = 0; j < image_height; ++j)
        {
            std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
            for (int i = 0; i < image_width; ++i)
            {
                auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);
                auto ray_direction = pixel_center - camera_center;
                ray r(camera_center, ray_direction);

                color pixel_color = ray_color(r, world);
                writeColor(std::cout, pixel_color);
            }
        }
    }
private:
    int    image_height;   // Rendered image height
    point3 camera_center;         // Camera center
    point3 pixel00_loc;    // Location of pixel 0, 0
    vec3   pixel_delta_u;  // Offset to pixel to the right
    vec3   pixel_delta_v;  // Offset to pixel below

    void initialize()
    {
        image_height = static_cast<int>(image_width / aspect_ratio);
        image_height = (image_height < 1) ? 1 : image_height;

        camera_center = point3(0, 0, 0);

        // Determine viewport dimensions.
        auto focal_length = 1.0;
        auto viewport_height = 2.0;
        auto viewport_width = viewport_height * (static_cast<double>(image_width) / image_height);

        // Calculate the vectors across the horizontal and down the vertical viewport edges.
        auto viewport_u = vec3(viewport_width, 0, 0);
        auto viewport_v = vec3(0, -viewport_height, 0);

        // Calculate the horizontal and vertical delta vectors from pixel to pixel.
        pixel_delta_u = viewport_u / image_width;
        pixel_delta_v = viewport_v / image_height;

        // Calculate the location of the upper left pixel.
        auto viewport_upper_left =
            camera_center - vec3(0, 0, focal_length) - viewport_u / 2 - viewport_v / 2;
        pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);
    }
    color ray_color(const ray& r, const hittable& world) const(){}
};
```
The new main:
```c++
#include "rtweekend.h"
#include "camera.h"
#include "hittable_list.h"
#include "sphere.h"
int main()
{
    // World
    hittable_list world;
    world.add(make_shared<sphere>(point3(0, 0, -1), 0.5));
    world.add(make_shared<sphere>(point3(0, -100.5, -1), 100));

    camera cam;
    cam.aspect_ratio = 16.0 / 9.0;
    cam.image_width = 400;
    cam.render(world);
}
```
# 8 Antialiasing
[[06 光栅化（深度测试与抗锯齿）]]
We'll adopt the simplest model: sampling the square region centered at the pixel that extends halfway to each of the four neighboring pixels. This is not the optimal approach, but it is the most straight-forward.
_Pixel samples_
![[Pasted image 20240204160903.png]]
## 8.1 Some Random Number Utilities
We need a random number generator that returns real random numbers
This function should return a canonical random number, which by convention falls in the range  $0≤n<1$ The “less than” before the 1 is important, as we will sometimes take advantage of that.

A simple approach to this is to use the `rand()` function that can be found in `<cstdlib>`, which returns a random integer in the range 0 and `RAND_MAX`.
```c++
inline double random_double()
{
    // Returns a random real in [0,1).
    return rand() / (RAND_MAX + 1.0);
}
inline double random_double(double min, double max)
{
    // Returns a random real in [min,max).
    return min + (max - min) * random_double();
}
```
_C++ did not traditionally have a standard random number generator, but newer versions of C++ have addressed this issue with the `<random>` header (if imperfectly according to some experts). If you want to use this, you can obtain a random number with the conditions we need as follows:_
```c++
#include <random>
inline double random_double() 
{
    static std::uniform_real_distribution<double> distribution(0.0, 1.0);
    static std::mt19937 generator;
    return distribution(generator);
}
```
## 8.2 Generating Pixels with Multiple Samples
For a single pixel composed of multiple samples, we'll select samples from the area surrounding the pixel and average the resulting light (color) values together.

To ensure that the color components of the final result remain within the proper [0,1] bounds, we'll add and use a small helper function: `interval::clamp(x)`.
```c++
double clamp(double x) const
{
	if (x < min)return min;
	if (x > max)return max;
}
```
And here's the updated `writeColor()` function that takes the sum total of all light for the pixel and the number of samples involved:
_color.h_
```c++
void writeColor(std::ostream& out, color pixel_color, int samples_per_pixel)
{
    auto r = pixel_color.x();
    auto g = pixel_color.y();
    auto b = pixel_color.z();

    // Divide the color by the numbers of samples
    auto scale = 1.0 / samples_per_pixel;
    r *= scale;
    g *= scale;
    b *= scale;

    // Write the translated [0,255] value of each color component.
    static const interval intensity(0.000, 0.999);
    out << static_cast<int>(256 * intensity.clamp(r)) << ' '
        << static_cast<int>(256 * intensity.clamp(g)) << ' '
        << static_cast<int>(256 * intensity.clamp(b)) << '\n';
}
```

Now let's update the camera class to define and use a new `camera::get_ray(i,j)` function, which will generate different samples for each pixel. This function will use a new helper function `pixel_sample_square()` that generates a random sample point within the unit square centered at the origin. We then transform the random sample from this ideal square back to the particular pixel we're currently sampling.

update _camera.h_ 
```c++
for (int j = 0; j < image_height; ++j)
{
	std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
	for (int i = 0; i < image_width; ++i)
	{
		color pixel_color(0, 0, 0);
		for (int sample = 0; sample < samples_per_pixel; ++sample)
		{
			ray r = get_ray(i, j);
			pixel_color += ray_color(r, world);
		}
		writeColor(std::cout, pixel_color, samples_per_pixel);
	}
}
ray get_ray(int i, int j) const
{
	// Get a randomly sampled camera ray for the pixel at location i,j.
	auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);
	auto pixel_sample = pixel_center + pixel_sample_square();

	auto ray_origin = camera_center;
	auto ray_direction = pixel_sample - ray_origin;

	return ray(ray_origin, ray_direction);
}

vec3 pixel_sample_square() const
{
	//偏移
	// Returns a random point in the square surrounding a pixel at the origin.
	auto px = -0.5 + random_double();
	auto py = -0.5 + random_double();
	return (px * pixel_delta_u) + (py * pixel_delta_v);
}
```
_总结_：
在render里面，通过循环在同一个像素处，采样多次，并且每次采样时对像素进行偏移，最后混合，达到抗锯齿（模糊）的效果
![[Pasted image 20240205130505.png]]

# 9 Diffuse Materials
We’ll start with diffuse materials (also called _matte_). One question is whether we mix and match geometry and materials (so that we can assign a material to multiple spheres, or vice versa) or if geometry and materials are tightly bound (which could be useful for procedural objects where the geometry and material are linked). We’ll go with separate — which is usual in most renderers — but do be aware that there are alternative approaches.

## 9.1 A Simple Diffuse Material
Light that reflects off a diffuse surface has its direction randomized, so, if we send three rays into a crack between two diffuse surfaces they will each have different random behavior:
![[Pasted image 20240205163425.png]]
 Let's start with the most intuitive: a surface that randomly bounces a ray equally in all directions. For this material, a ray that hits the surface has an equal probability of bouncing in any direction away from the surface.![[Pasted image 20240205164023.png]]
 **The first thing we need is the ability to generate arbitrary random vectors:**
 _vec3.h_ 
```c++
class vec3 
{
  public:
    static vec3 random()
     {
        return vec3(random_double(), random_double(), random_double());
    }
    static vec3 random(double min, double max) 
    {
        return vec3(random_double(min,max), random_double(min,max), random_double(min,max));
    }
};
```
Then we need to figure out how to manipulate a random vector so that we only get results that are on the surface of a hemisphere.

We'll use what is typically the easiest algorithm: A rejection method. 
1. Generate a random vector inside of the unit sphere
2. Normalize this vector
3. Invert the normalized vector if it falls onto the wrong hemisphere

First, we will use a rejection method to generate the random vector inside of the unit sphere. 

Pick a random point in the unit cube, where $x$, $y$, and $z$ all range from −1 to +1, and reject this point if it is outside the unit sphere.
_Two vectors were rejected before finding a good one_
![[Pasted image 20240205165026.png]]
_vec3.h_
```c++
inline vec3 random_in_unit_sphere()
{
    while (true)
    {
        auto p = vec3::random(-1, 1);
        if (p.length_squared() < 1)
            return p;
    }
}
```

_The accepted random vector is normalized to produce a unit vector_
![[Pasted image 20240205165442.png]]
```c++
inline vec3 random_unit_vector()
{
    return unit_vector(random_in_unit_sphere());
}
```
And now that we have a random vector on the surface of the unit sphere, we can determine if it is on the correct hemisphere by comparing against the surface normal:
![[Pasted image 20240205165656.png]]

We can take the dot product of the surface normal and our random vector to determine if it's in the correct hemisphere. If the dot product is positive, then the vector is in the correct hemisphere. If the dot product is negative, then we need to invert the vector.
```c++
inline vec3 random_on_hemisphere(const vec3& normal)
{
    vec3 on_unit_sphere = random_unit_vector();
    if (dot(on_unit_sphere, normal) > 0.0) // In the same hemisphere as the normal
        return on_unit_sphere;
    else
        return -on_unit_sphere;
}
```
 As a first demonstration of our new diffuse material we'll set the `ray_color` function to return 50% of the color from a bounce. We should expect to get a nice gray color.
 _camera.h_
 ```c++
 color ray_color(const ray& r, const hittable& world) const
{
	hit_record rec;
	if (world.hit(r, interval(0, infinity), rec))
	{
		vec3 direction = random_on_hemisphere(rec.normal);
		return 0.5 * ray_color(ray(rec.p, direction), world);
		//return 0.5 * (rec.normal + color(1, 1, 1));
	}
	vec3 unit_direction = unit_vector(r.direction());
	auto a = 0.5 * (unit_direction.y() + 1.0);
	return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}
```
总结：首先在球内生成随机生成向量，然后标准化，然后和法线做点乘判断该向量是否在正确的半球，如果是就返回，如果不是就将他反向再返回，以确保结果在同一半球面上。最后根据得到的向量，递归调用ray_color，达到反射的效果
![[Pasted image 20240206171733.png]]
## 9.2 Limiting the Number of Child Rays
Limit the maximum recursion depth, returning no light contribution at the maximum depth:
_camera.h_ 
```c++
class camera
{
public:
	...
    int max_depth = 10;    // Maximum number of ray bounces into scene
    ...
    for (int sample = 0; sample < samples_per_pixel; ++sample)
	{
		ray r = get_ray(i, j);
		pixel_color += ray_color(r,max_depth,world);
	}
	...
color ray_color(const ray& r, int depth, const hittable& world) const
{
	hit_record rec;
	// If we've exceeded the ray bounce limit, no more light is gathered.
	if (depth <= 0)
		return color(0, 0, 0);
	if (world.hit(r, interval(0, infinity), rec))
	{
		vec3 direction = random_on_hemisphere(rec.normal);
		return 0.5 * ray_color(ray(rec.p, direction), depth - 1, world);
	}
	vec3 unit_direction = unit_vector(r.direction());
	auto a = 0.5 * (unit_direction.y() + 1.0);
	return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}
```
_main.cpp_
```c++
int main()
{
	...
    camera cam;
	...
    cam.max_depth = 50;
```
For this very simple scene we should get basically the same result:
![[Pasted image 20240206171639.png]]

## 9.3 Fixing Shadow Acne
A ray will attempt to accurately calculate the intersection point when it intersects with a surface.But this calculation is susceptible to floating point rounding errors which can cause the intersection point to be ever so slightly off.

If the ray's origin is just below the surface then it could intersect with that surface again.

The simplest hack to address this is just to ignore hits that are very close to the calculated intersection point:
_camera.h_ 
```c++
if (world.hit(r, interval(0.001, infinity), rec))
```
This gets rid of the shadow acne problem. Yes it is really called that. Here's the result:
![[Pasted image 20240206173944.png]]
## 9.4 True Lambertian Reflection
[[07 着色（光照与基本着色模型）#Lambert漫反射模型]]
_Randomly generating a vector according to Lambertian distribution_ 
![[Pasted image 20240206194513.png]]
_camera.h_ 
```c++
if (world.hit(r, interval(0.001, infinity), rec))
{
	vec3 direction = rec.normal + random_unit_vector();
	//vec3 direction = random_on_hemisphere(rec.normal);
	return 0.5 * ray_color(ray(rec.p, direction), depth - 1, world);
}
```
![[Pasted image 20240206195150.png]]
It's hard to tell the difference between these two diffuse methods, given that our scene of two spheres is so simple, but you should be able to notice two important visual differences:

1. The shadows are more pronounced after the change
2. Both spheres are tinted blue from the sky after the change (属实没看出来)

![[Pasted image 20240206195329.png]]

Both of these changes are due to the less uniform scattering of the light rays—more rays are scattering toward the normal. This means that for diffuse objects, they will appear _darker_ because less light bounces toward the camera. For the shadows, more light bounces straight-up, so the area underneath the sphere is darker.

## 9.5 Gamma Correction
_The gamut of our renderer so far_ 
![[Pasted image 20240206195832.png]]
We need to go from linear space to gamma space, which means taking the inverse of $gamma 2$, which means an exponent of $1/gamma$, which is just the square-root.
_color.h_
```c++
inline double linear_to_gamma(double linear_component)
{
    return sqrt(linear_component);
}
void writeColor(std::ostream& out, color pixel_color, int samples_per_pixel)
{
	...
    // Apply the linear to gamma transform.
    r = linear_to_gamma(r);
    g = linear_to_gamma(g);
    b = linear_to_gamma(b);
    ...
}
```
![[Pasted image 20240206200537.png]]
# 10 Metal
## 10.1 An Abstract Class for Materials
We could have an abstract material class that encapsulates unique behavior.

For our program the material needs to do two things:
1. Produce a scattered ray (or say it absorbed the incident ray).
2. If scattered, say how much the ray should be attenuated.
_material.h_ 
```c++
#pragma once
#include"rtweekend.h"
class hit_record;
class material
{
public:
    virtual ~material() = default;
    virtual bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered) const = 0;
};
```
## 10.2 A Data Structure to Describe Ray-Object Intersections
In C++ you just tell the compiler that we're storing a pointer to the material inside `hit_record`
_hittable.h_ 
```c++
#include"rtweekend.h"
class material;
class hit_record
{
public:
    point3 p;
    vec3 normal;
    shared_ptr<material> mat;
    double t;

    bool front_face;
    void setFaceNormal(const ray& r, const vec3& outward_normal)
    {
        // Sets the hit record normal vector.
        // NOTE: the parameter `outward_normal` is assumed to have unit length.
        front_face = dot(r.direction(), outward_normal) < 0;
        normal = front_face ? outward_normal : -outward_normal;
    }
};
...
```
注意：  
在给定的代码中，`class material;`是一个前置声明或者叫做前向声明（forward declaration）。它的作用是告诉编译器在此处有一个名为`material`的类存在，但是并不提供其完整的定义。这种声明允许你在当前作用域中使用该类的指针或引用，而不需要实际的类定义。

当光线射入一个表面(比如一个球体), hit_record中的材质指针会被球体的材质指针所赋值, 而球体的材质指针是在main()函数中构造时传入的。当color()函数获取到hit_record时, 他可以找到这个材质的指针, 然后由材质的函数来决定光线是否发生散射, 怎么散射。

To achieve this, `hit_record` needs to be told the material that is assigned to the sphere.
_sphere.h_ 
```c++
class sphere: public hittable
{
public:
    sphere(point3 _center, double _radius, shared_ptr<material> _material)
        : center(_center), radius(_radius), mat(_material)   {  }
    bool hit(const ray& r, interval ray_t, hit_record& rec) const override
    {
        ...
        rec.mat = mat;
        return true;
    }

private:
    point3 center;
    double radius;
    shared_ptr<material> mat;
};
```
## 10.3 Modeling Light Scatter and Reflectance
对于我们之前写过的Lambertian(漫反射)材质来说, 这里有两种理解方法, 要么是光线永远发生散射, 每次散射衰减至R, 要么是光线并不衰减, 转而物体吸收(1-R)的光线。
_material.h_
```c++
class lambertian: public material
{
public:
    lambertian(const color& a):albedo(a){}
    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered) const override
    {
        auto scatter_dircetion = rec.normal + random_unit_vector();
        scattered = ray(rec.p, scatter_dircetion);
        attenuation = albedo;
        return true;
    }
private:
    color albedo;
};
```
注意我们也可以让光线根据一定的概率p发生散射【译注: 若判断没有散射, 光线直接消失】, 并使光线的衰减率(代码中的attenuation)为 $albedo/p$ 。

If the random unit vector we generate is exactly opposite the normal vector, the two will sum to zero, which will result in a zero scatter direction vector. 

In service of this, we'll create a new vector method — `vec3::near_zero()` — that returns true if the vector is very close to zero in all dimensions.
_vec3.h_ 
```c++
class vec3
{
    ...
    bool near_zero() const 
    {
        // Return true if the vector is close to zero in all dimensions.
        auto s = 1e-8;
        return (fabs(e[0]) < s) && (fabs(e[1]) < s) && (fabs(e[2]) < s);
    }
};
```
_material.h_
```c++
bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered) const override
{
	auto scatter_direction = rec.normal + random_unit_vector();

	// Catch degenerate scatter direction
	if (scatter_direction.near_zero())
		scatter_direction = rec.normal;

	scattered = ray(rec.p, scatter_direction);
	attenuation = albedo;
	return true;
}
```
## 10.4 Mirrored Light Reflection
![[Pasted image 20240207142146.png]]
The reflected ray direction in red is just $\vec v+2\vec b$.  In our design, $\vec n$ is a unit vector, but $\vec v$ may not be. The length of $\vec b$ should be $\vec v⋅\vec n$(v在n上的投影的长度). Because $\vec v$ points in, we will need a minus sign, yielding:
_vec3.h_ 
```c++
vec3 reflect(const vec3& v, const vec3& n)
{
    return v - 2 * dot(v, n) * n;
}
```

The metal material just reflects rays using that formula:
_material.h_ 
```c++
class metal: public material
{
public:
    metal(const color& a): albedo(a) {}
    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
        const override
    {
        vec3 reflected = reflect(unit_vector(r_in.direction()), rec.normal);
        scattered = ray(rec.p, reflected);
        attenuation = albedo;
        return true;
    }
private:
    color albedo;
};
```
We need to modify the `ray_color()` function for all of our changes:
_Ray color with scattered reflectance_
_camera.h_ 
```c++
color ray_color(const ray& r, int depth, const hittable& world) const
{
	hit_record rec;
	// If we've exceeded the ray bounce limit, no more light is gathered.
	if (depth <= 0)
		return color(0, 0, 0);
	if (world.hit(r, interval(0.001, infinity), rec))
	{
		ray scattered;
		color attenuation;
		if (rec.mat->scatter(r, rec, attenuation, scattered))
			return attenuation * ray_color(scattered, depth - 1, world);
		return color(0, 0, 0);
	}
	vec3 unit_direction = unit_vector(r.direction());
	auto a = 0.5 * (unit_direction.y() + 1.0);
	return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}
```
## 10.5 A Scene with Metal Spheres
Now let’s add some metal spheres to our scene:
_main.cpp_
```c++
#include "rtweekend.h"

#include "camera.h"
#include "color.h"
#include "hittable_list.h"
#include "material.h"
#include "sphere.h"
int main()
{
    // World

    hittable_list world;

    auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
    auto material_center = make_shared<lambertian>(color(0.7, 0.3, 0.3));
    auto material_left = make_shared<metal>(color(0.8, 0.8, 0.8));
    auto material_right = make_shared<metal>(color(0.8, 0.6, 0.2));

    world.add(make_shared<sphere>(point3(0.0, -100.5, -1.0), 100.0, material_ground));
    world.add(make_shared<sphere>(point3(0.0, 0.0, -1.0), 0.5, material_center));
    world.add(make_shared<sphere>(point3(-1.0, 0.0, -1.0), 0.5, material_left));
    world.add(make_shared<sphere>(point3(1.0, 0.0, -1.0), 0.5, material_right));
    ...
}
```
![[Pasted image 20240207150301.png]]
## 10.6 Fuzzy Reflection
We can also randomize the reflected direction by using a small sphere and choosing a new endpoint for the ray. We'll use a random point from the surface of a sphere centered on the original endpoint, scaled by the fuzz factor.
![[Pasted image 20240207150629.png]]
The bigger the sphere, the fuzzier the reflections will be. This suggests adding a fuzziness parameter that is just the radius of the sphere (so zero is no perturbation). The catch is that for big spheres or grazing rays, we may scatter below the surface. We can just have the surface absorb those.
_material.h_
```c++
class metal : public material
{
public:    
	metal(const color& a, double f) : albedo(a), fuzz(f < 1 ? f : 1) {}
    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
    const override 
    {
        vec3 reflected = reflect(unit_vector(r_in.direction()), rec.normal);
		scattered = ray(rec.p, reflected + fuzz*random_unit_vector());        
		attenuation = albedo;        
		return (dot(scattered.direction(), rec.normal) > 0);    
	}
  private:
    color albedo;    double fuzz;
};
```
We can try that out by adding fuzziness 0.3 and 1.0 to the metals:

```c++
int main() 
{
    ...
    auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
    auto material_center = make_shared<lambertian>(color(0.7, 0.3, 0.3));    
    auto material_left   = make_shared<metal>(color(0.8, 0.8, 0.8), 0.3);
    auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2), 1.0);  
      ...
}
```
![[Pasted image 20240207151028.png]]
# 11 Dielectrics
## 11.1 Refraction
 Have all the light refract if there is a refraction ray at all.
_Glass first_ 
![[Pasted image 20240207151350.png]]
The world should be flipped upside down and no weird black stuff.
## 11.2 Snell's Law
The refraction is described by Snell’s law:$$η⋅sinθ=η^′⋅sinθ^′$$
 θ与 θ′ 是入射光线与折射光线距离法相的夹角,  η与 η′是介质的折射率(规定空气为1.0, 玻璃为1.3-1.7,钻石为2.4), 如图: ![[Pasted image 20240207151806.png]]In order to determine the direction of the refracted ray, we have to solve for $sinθ^′$:
$$ sinθ^′=\fracη{η′}⋅sinθ$$
![[Pasted image 20240207153252.png]]
![[Pasted image 20240207153303.png]]
_material.h_ 
```c++
class dielectric: public material
{
public:
    dielectric(double index_of_refraction): ir(index_of_refraction) {}
    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
        const override
    {
        attenuation = color(1.0, 1.0, 1.0);
        double refraction_ratio = rec.front_face ? (1.0 / ir) : ir;

        vec3 unit_direction = unit_vector(r_in.direction());
        vec3 refracted = refract(unit_direction, rec.normal, refraction_ratio);

        scattered = ray(rec.p, refracted);
        return true;
    }
private:
    double ir; // Index of Refraction
};
```
_main.cpp_
```c++
auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
auto material_center = make_shared<dielectric>(1.5);
auto material_left   = make_shared<dielectric>(1.5);
auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2), 1.0);
```
This gives us the following result:
![[Pasted image 20240207153553.png]]
## 11.3 Total Internal Reflection
当光线从高折射律介质射入低折射率介质时, 对于上述的Snell方程可能没有实解【 sin⁡θ>1 】。这时候就不会发生折射, 所以就会出现许多小黑点。我们回头看一下snell法则的式子:
![[Pasted image 20240207153757.png]]
_material.h_ 
```c++
if (refraction_ratio * sin_theta > 1.0) 
{
    // Must Reflect
    ...
} 
else 
{
    // Can Refract
    ...
}
```
这里所有的光线都不发生折射, 转而发生了反射。因为这种情况常常在实心物体的内部发生, 所以我们称这种情况被称为"全内反射"。这也当你浸入水中时, 你发现水与空气的交界处看上去像一面镜子的原因。
![[Pasted image 20240207153915.png]]
一个在可以偏折的情况下总是偏折, 其余情况发生反射的绝缘体材质为:
_material.h_ 
```c++
class dielectric: public material
{
public:
    dielectric(double index_of_refraction): ir(index_of_refraction) {}

    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
        const override
    {
        attenuation = color(1.0, 1.0, 1.0);
        double refraction_ratio = rec.front_face ? (1.0 / ir) : ir;

        vec3 unit_direction = unit_vector(r_in.direction());
        double cos_theta = fmin(dot(-unit_direction, rec.normal), 1.0);
        double sin_theta = sqrt(1.0 - cos_theta * cos_theta);

        bool cannot_refract = refraction_ratio * sin_theta > 1.0;
        vec3 direction;

        if (cannot_refract)
            direction = reflect(unit_direction, rec.normal);
        else
            direction = refract(unit_direction, rec.normal, refraction_ratio);

        scattered = ray(rec.p, direction);
        return true;
    }

private:
    double ir; // Index of Refraction
};
```
这里的光线衰减率为1——就是不衰减, 玻璃表面不吸收光的能量。如果我们使用下面的参数:
_main.cpp_ 
```c++
auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));
auto material_left = make_shared<dielectric>(1.5);
auto material_right = make_shared<metal>(color(0.8, 0.6, 0.2), 0.0);
```
![[Pasted image 20240207154126.png]]
