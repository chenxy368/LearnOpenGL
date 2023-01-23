# Anti Aliasing
Jagged edges appear is due to how the rasterizer transforms the vertex data into actual fragments behind the scene.  
![image](https://user-images.githubusercontent.com/98029669/213954227-948e0ff4-5424-4a70-b2de-2b9ddd79d8f0.png)  
This effect, of clearly seeing the pixel formations an edge is composed of, is called __aliasing__. 
There are quite a few techniques out there called __anti-aliasing techniques__ that fight this aliasing behavior by producing __smoother edges__.

P.S.Super sample anti-aliasing (SSAA) that temporarily __uses a much higher resolution render buffer to render__ the scene in (super sampling). 
Then when the full scene is rendered, the __resolution is downsampled back__ to the normal resolution. 
While it did provide us with a solution to the aliasing problem, it came with __a major performance drawback__. 
This technique therefore only had a short glory moment.

## Multisampling
How OpenGL rasterizes triangles?  
![image](https://user-images.githubusercontent.com/98029669/213955089-4fd52ef8-e0e9-4d5f-8890-d84e72c1f27d.png)
![image](https://user-images.githubusercontent.com/98029669/213955098-8dac383d-afb7-4969-999a-ad6d67abc0da.png)
![image](https://user-images.githubusercontent.com/98029669/213955113-ed0aae35-455a-46d2-a6bd-5b1758b48d4b.png)


From  
![image](https://user-images.githubusercontent.com/98029669/213955160-7fd6f176-8a19-493a-988e-7e25338735bd.png)  
To  
![image](https://user-images.githubusercontent.com/98029669/213955178-27ab1e13-0437-4fc6-9749-e4f8931fa73f.png)

Due to the limited amount of screen pixels, some pixels will be rendered along an edge and some won't. 
The result is that we're rendering primitives with non-smooth edges giving rise to the jagged edges we've seen before.

What multisampling does, is not use a single sampling point for determining coverage of the triangle, but multiple sample points (guess where it got its name from).  
![image](https://user-images.githubusercontent.com/98029669/213955780-770e60f6-5149-4cb2-b90a-be9637eb18b9.png)   
![image](https://user-images.githubusercontent.com/98029669/213957034-616fd360-c0c9-4f9f-b5d3-1b4900c5cbff.png)

__Depth and stencil values are stored per subsample__ and, even though we __only run the fragment shader once__, __color values are stored per subsample__
as well for the case of __multiple triangles overlapping__ a single pixel. 
For depth testing the vertex's depth value is __interpolated to each subsample before__ running the depth test,
and for stencil testing we store the stencil values __per subsample__. 
This does mean that __the size of the buffers are now increased__ by the amount of subsamples per pixel.

From  
![image](https://user-images.githubusercontent.com/98029669/213957748-e523df13-b398-43e7-8299-f32dcb124493.png)  
To  
![image](https://user-images.githubusercontent.com/98029669/213957761-57451ebd-ea70-43f4-85aa-4b29eaaaa10e.png)

## MSAA in OpenGL
 We need a new type of buffer that can store a given amount of multisamples and this is called a multisample buffer.  
 To set MSAA sample amount
 ```C++
 glfwWindowHint(GLFW_SAMPLES, 4);
 ```
 To enable MSAA
 ```C++
 glEnable(GL_MULTISAMPLE);
 ```
 
