# Shadow Mapping
![image](https://user-images.githubusercontent.com/98029669/214085676-f354880a-0201-47ad-ae89-66105edee65d.png)
## Shadow Mapping
We render the scene from the light's point of view and everything we see from the light's perspective is lit and everything we can't see must be in shadow.
(If camera at the light's position, what cannot be seen should be in shadow.)  
![image](https://user-images.githubusercontent.com/98029669/214086029-28d7b15c-19a2-471f-8258-a8c79a20eda1.png)  
Here blue line are lit and black line are in shadow.

Naively, we want to get the point on the ray where it first hit an object and compare this closest point to other points on this ray. 
We then do a basic test to see if a test point's ray position is further down the ray than the closest point and if so, the test point must be in shadow. __(Too computational expensive)__

__!!!If camera at the light's position, what cannot be seen should be in shadow.!!!__ Why not use the depth buffer when we put the camera at the light's position to construct shadow.

You may remember from the depth testing chapter that a value in the depth buffer corresponds to the depth of a fragment clamped to [0,1] from the camera's point of view. 
After all, the depth values show the first fragment visible from the light's perspective. 
We store __all these depth values in a texture__ that we call a __depth map or shadow map__.


