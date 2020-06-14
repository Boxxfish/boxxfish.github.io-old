---
layout: post
title: How to Create a Gizmo System
date: '2020-06-06 23:13:18'
tags:
- 3d
- tutorial
---

![Header Image](/assets/pics/6_6_20/noot.png)

While working on my latest project, I had to implement my own gizmo system. For the uninitiated, gizmos are tools that lets you manipulate 3D content; in a sense, a 3D GUI. Hopefully this post will save people from a lot of the headaches I had to endure trying to figure out how to do this thing.

## Casting Rays

Before we start writing code, we need to first talk about raycasts. A raycast involves finding the point on a surface a given ray intersects. While I'm going to focus on ray-plane intersections, note that raycasts can be done on many other simple geometric shapes.

![Intersections](/assets/pics/6_6_20/intersections.png)

If you've played any shooters recently, you've seen the power of raycasts first hand. Let's say you needed to take out someone with a sniper rifle. Typically, this is what you'd see:

![Sniper 1](/assets/pics/6_6_20/sniper1.png)

When you take the shot, the game has to figure out which "box" the center of the reticle is over. It does this by executing the following series of steps:

 1. Calculate a ray in world space that the bullet will follow.
 2. For every box in the scene, check if it intersects the ray. If it does, store the point on its surface the ray goes through.
 3. Of all the boxes the ray intersects, find the one with the intersection closest to the player. This is the box that has been hit.

From another angle, it looks like this:

![Sniper 2](/assets/pics/6_6_20/sniper2.png)

Now, let's get into the math.

### Rays and Bounding Volumes

`Quick note here: when I use the term "bounding volume", I'm talking about the boxes from before and other geometric shapes you can ray cast against. This includes planes, which isn't *technically* correct because they don't represent volume, but... ignore that.`

Let's start with the equation of a ray. A ray consists of a starting point and a directional vector. Every point along a ray will follow the equation

\[ p = o + dt \]

where *p* is the point, *o* is the starting point, *d* is the direction vector, and *t* is the distance from the starting point. If we want to find the intersection of a ray and some bounding volume now, all we have to do is set \( o + dt \) equal to the equation for a point on the bounding volume, solve for *t*, then plug it back into our original ray equation.

Let's take a look at the equation of a plane. A plane consists of a normal vector orthogonal to its surface and a center point. Every point on a plane will follow the equation

\[ n * (o - p) = 0 \]

where *n* is the normal, *o* is the center point, and *p* is the point. Intuitively, this works because the difference between any 2 points on the plane, \( (o - p) \), should be orthogonal to the normal, so when the dot product is applied to *n* and *o - p*, the result is 0.

Substituting *p* from the plane equation with the ray equation, then solving for t, we get

\[ t = n * (o_plane - o_ray) / (n * d) \]

What about a sphere? A sphere consists of a center point and a radius. Every point is defined as \( (p - o)^2 = r^2 \), where *r* is the radius. Again, if you substitute *p*, you can obtain *t*, which can be plugged into the ray equation.

Next, we'll look at how to generate rays from screen space coordinates.

### Camera Rays

How does our 3D content get mapped to pixels on our screen? Put simply, it undergoes a process like this:

 1. Each vertex is multiplied by a local-to-world-space matrix. Now the vertices are in world space.

 2. Each world space vertex is multiplied by a world-to-view-space matrix, which is the inverse of the camera's world transform matrix. Now the vertices are in view space.

 3. Each view space vertex is multiplied by a projection matrix. This causes each vertex to have perspective division performed on it, so vertices that are farther from the screen are scaled down. Now the vertices are in clip space, and the z coordinate has been removed.

 4. The GPU rasterizes our transformed 3D content and shades the pixels of our screen accordingly. Now the vertices are in screen space, and there is a physical pixel on our screen for every vertex.

 ![Rasterization](/assets/pics/6_6_20/rasterization.png)

In a gizmo system, we want to do the opposite: to take the current pixel under our cursor and find a corresponding world space coordinate. Could we just do the steps above in reverse to get a world space point from a screen space point? Yes and no. Doing the steps above *does* yield a world space point, but it's projected onto a plane that faces the camera and goes through the world space origin. Fortunately, all is not lost. As long as we define some geometry in world space, we can use raycasts to select objects and project screen space points onto any shapes we want.

We'll need a way to convert our screen space mouse coordinates into world space rays. Since we know where our camera is, we can use that as our ray's starting point. By projecting our mouse coordinates onto the world space plane using the reverse of the steps above, we can obtain a point along the ray, and use the difference between this point and our camera position to get our ray's direction. From there, we just need to use our raycasting equations from the previous section to get our world space point!

![Projection](/assets/pics/6_6_20/projection.png)

Here's some Python pseudocode where we project a screen space coordinate onto some arbitrary plane. Once you define helper functions like this, raycasting becomes a breeze.

```Python
def project_onto_plane(screen_coord, plane_normal, plane_center):
  """ Projects a screen space coordinate onto a world space plane
  and returns the result.

  :param screen_coord: the screen space coordinate
  :param plane_normal: the world space plane's normal
  :param plane_center: the world space plane's center
  :return: the world space point under screen_coord that lies on the plane
  """

  # Obtain camera ray
  ray_center = cam_pos
  view_to_world_mat = inverse(world_to_view_mat)  # world_to_view_mat is the matrix from step 2 multiplied by the matrix from step 3
  ray_point = view_to_world_mat.dot(screen_coord)
  ray_dir = ray_point - ray_center

  # Perform raycast against plane
  t = plane_normal.dot(plane_center - ray_center) / (plane_normal.dot(ray_dir))
  projected_point = ray_center + t * ray_dir
  return projected_point

```

That's all the math we need! Now, let's actually implement a simple gizmo system.

## The Part We've Been Waiting For

Let's start off with a class called `GizmoSystem`. Objects of this class get passed all the data needed (camera position, relevant matrices, etc.) on initialization. When we want to create a piece of interactable geometry, we call the corresponding method from `GizmoSystem` and pass callbacks that trigger when we interact with the gizmo.

```Python
class GizmoSystem:

  def __init__(params):
    # Initialize stuff here

  def create_line(point_a, point_b, on_press, on_release, on_drag...):
    ref = Line(point_a, point_b, on_press, on_release, on_drag)
    self.geoms.append(ref)
    return ref

  def create_sphere(center, radius, on_press, on_release, on_drag...):
    ref = Sphere(center, radius, on_press, on_release, on_drag)
    self.geoms.append(ref)
    return ref

  def create_cube(center, width, height, depth, on_press, on_release, on_drag...):
    ref = Cube(center, width, height, depth, on_press, on_release, on_drag)
    self.geoms.append(ref)
    return ref

  def create_cone(center, axis, radius, on_press, on_release, on_drag...):
    ref = Cone(center, axis, radius, on_press, on_release, on_drag)
    self.geoms.append(ref)
    return ref

  # And so on...
```

What are `Line`, `Sphere`, and `Cube`? They're subclasses of `RaycastGeom`, which stores geometry properties and callbacks, and contains a method to override called `raycast` that returns the intersection between a given ray and this geometry.


```Python
class RaycastGeom:

  def __init__(callbacks):
    # Store the callbacks

  def raycast(ray_origin, ray_dir):
    return None
```

To interact with our geometry, `GizmoSystem` has event handlers that trigger when the cursor moves, a mouse button is pressed, etc.

```Python
class GizmoSystem:
  ...

  # Helper function for obtaining rays
  def get_ray(self, screen_coord):
    ray_center = self.cam_pos
    ray_point = self.view_to_world_mat.dot(screen_coord)
    ray_dir = ray_point - ray_center
    return ray_center, ray_dir

  # When the mouse moves, recalculate which geometry is under the mouse.
  # Remember our sniper example?
  def handle_mouse_moved(self, new_x, new_y):
    closest_geom = None
    closest_dist = 999
    for geom in self.geoms:

      world_point = geom.raycast(self.get_ray((new_x, new_y))))
      if world_point is not None:
        # Check if the distance between world_point and the camera is less than closest_dist

        point_dist = length(world_point - self.cam_pos)
        if point_dist < closest_dist:
          closest_geom = geom
          closest_dist = point_dist

      # Set as the target gizmo the geometry directly under the cursor
      self.target_gizmo = closest_geom

      # If the mouse button is down, trigger the on_drag callback of the target gizmo
      if self.mouse_down and self.target_gizmo is not None:
        self.target_gizmo.on_drag(new_x, new_y)

      # Store the new mouse coordinates
      self.mouse_x = new_x
      self.mouse_y = new_y

  # When the mouse button is pressed or released, trigger the callbacks on the target gizmo
  def handle_mouse_pressed(self):
    self.mouse_down = True

    if self.target_gizmo is not None:
      self.target_gizmo.on_press()

  def handle_mouse_released(self):
    self.mouse_down = False

    if self.target_gizmo is not None:
      self.target_gizmo.on_release()
```

To finish up this tutorial, we'll use this gizmo system to implement a translation gizmo. When the translation arrow is dragged, the gizmo should
move in the direction of arrow such that the same part of the arrow is always under our cursor. In other words, if I select the tip of the arrow, no matter where I move the mouse, the tip of the arrow should be under my cursor.

Our arrow will be made up of a cone and a line pointing up in the Y direction, so let's start with that.

```Python
# We'll fill these callbacks in next
def on_press():
  pass

def on_release():
  pass

def on_drag(mouse_x, mouse_y):
  pass

# Since these two geometric shapes are part of the same gizmo, we can use the same callbacks
cone_ref = gizmo_system.create_cone((0, 1, 0),
                                    (0, 1, 0),
                                    0.25,
                                    on_press,
                                    on_release,
                                    on_drag)
line_ref = gizmo_system.create_line((0, 0, 0),
                                    (0, 1, 0),
                                    on_press,
                                    on_release,
                                    on_drag)
```

To move the arrow the way we want, we need to do some more raycasting. More specifically, the following series of steps:

 1. Define a plane by using the bottom of the arrow as the plane origin and the X direction as the plane normal.

 2. When the arrow is selected, save the arrow's location. Use raycasting to get the position of the cursor projected onto the plane, and save that too.

 3. When the arrow is dragged, use raycasting to get the position of the new mouse position projected onto the plane. Find the difference between this cursor world projection and the original cursor world projection, and set the arrow's location to the original location + this difference.

```Python
orig_arrow_pos = (0, 0, 0)
orig_mouse_pos = (0, 0, 0)
def on_press():
  # Save the arrow's original location
  orig_arrow_pos = line_ref.point_a

  # Find the raycasted world position of the cursor
  orig_mouse_pos = project_onto_plane((gizmo_system.mouse_x, gizmo_system.mouse_y),
                                       (1, 0, 0),
                                       orig_arrow_pos)

def on_drag(mouse_x, mouse_y):
  # Get new raycasted world position of the cursor
  new_mouse_pos = project_onto_plane((mouse_x, mouse_y),
                                      (1, 0, 0),
                                      orig_arrow_pos)

  # Set the position of the arrow equal to the starting point + the difference
  translation = new_mouse_pos - orig_mouse_pos
  line_ref.point_a = orig_arrow_pos + translation
  line_ref.point_b = orig_arrow_pos + (0, 1, 0) + translation
  cone_ref.center = orig_arrow_pos + (0, 1, 0) + translation
```

![Gizmo](/assets/pics/6_6_20/gizmo.gif)

That wraps up this tutorial. I feel like one of the barriers to making intuitive 3D tools is understanding how things like gizmo systems work, and as a result, you get applications where you manually enter coordinates into some text box or have to use arrow keys to look around. Hopefully, this tutorial's helped you take your level editor/modelling app/CAD software up a notch.
