# SSAO 
One type of indirect lighting approximation is called ambient occlusion that tries to approximate indirect lighting by darkening creases, holes, and surfaces 
that are close to each other. These areas are largely occluded by surrounding geometry and thus light rays have fewer places to escape to, hence the areas appear darker. 

![image](https://user-images.githubusercontent.com/98029669/214497045-9a1c930b-a326-471b-b1ca-f4d5eb03cc30.png)

It is clear the quality and precision of the effect directly relates to the number of surrounding samples we take. 
We can reduce the amount of samples we have to test by introducing some randomness into the sample kernel. 
By randomly rotating the sample kernel each fragment we can get high quality results with a much smaller amount of samples.
This does come at a price as the randomness introduces a noticeable noise pattern that we'll have to fix by blurring the results. 

![image](https://user-images.githubusercontent.com/98029669/214497440-bad7c633-ab30-4e23-acae-53c4a541b962.png)

Because the sample kernel used was a sphere, it caused flat walls to look gray as half of the kernel samples end up being in the surrounding geometry. 

![image](https://user-images.githubusercontent.com/98029669/214497673-8e75ec46-1644-465d-886d-62461b8c52ed.png)

__In fact, the essential idea of SSAO is the same as all convolution kernel method in image processing. Just sample around the pixel and change the pixel value with the sample result.
Here, we also need some trick to have a better sample quality.__

## G-buffer
For each fragment, we're going to need the following data:

1. A per-fragment position vector.  
2. A per-fragment normal vector.  
3. A per-fragment albedo color.  
4. A sample kernel.  
5. A per-fragment random rotation vector used to rotate the sample kernel.  

![image](https://user-images.githubusercontent.com/98029669/214498211-c3ef3d6f-02c0-4ce4-b4e5-bddd85dc4fae.png)

__For sampling, we only need depth buffer which is a 2D texture so no 3D information is needed here.__
