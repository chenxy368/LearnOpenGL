# Camera
## Review
Local->World: Usually start from (0, 0, 0), find the desired position in the world space  
World->View: ?? In this section  
View->Clip: Perspective matrix, clipping outside element  
## Camera Space
![camera_axes](https://user-images.githubusercontent.com/98029669/213585444-e3a4812c-6aa0-4089-b0bc-92fb71fe0ba5.png)  
__1.position__  
 The camera position is a vector in world space that points to the camera's position. 
```C++
glm::vec3 cameraPos = glm::vec3(0.0f, 0.0f, 3.0f);
```  
Don't forget that the positive z-axis is going through your screen towards you so if we want the camera to move backwards, we move along the positive z-axis.  
__2.direction__  
The next vector required is the camera's direction e.g. at what direction it is pointing at. For now we let the camera point to the origin of our scene: (0,0,0).
Subtracting the camera position vector from the scene's origin vector thus results in the direction vector we want.   
```C++
glm::vec3 cameraTarget = glm::vec3(0.0f, 0.0f, 0.0f);
glm::vec3 cameraDirection = glm::normalize(cameraPos - cameraTarget);
```  
The name direction vector is not the best chosen name, since it is actually pointing in the reverse direction of what it is targeting.  
__3.right axis__  
The next vector that we need is a right vector that represents the positive x-axis of the camera space. 
To get the right vector we use a little trick by first specifying an up vector that points upwards (in world space). 
Then we do a cross product on the up vector and the direction vector from step 2. 
```C++
glm::vec3 up = glm::vec3(0.0f, 1.0f, 0.0f); 
glm::vec3 cameraRight = glm::normalize(glm::cross(up, cameraDirection));
```
__4.up axis__  
Cross product finds orthogonal vector.
```C++
glm::vec3 cameraUp = glm::cross(cameraDirection, cameraRight);
```
(Gram–Schmidt process: find orthogonal basis in a space)  
## Look At  
Transform any vector to that coordinate space by multiplying it with this matrix
![lookat](https://user-images.githubusercontent.com/98029669/213589342-f09cca2f-93d2-4fdb-9f6d-7eb558a9a3e1.png)  
Where R is the right vector, U is the up vector, D is the direction vector and P is the camera's position vector.  
```C++
glm::mat4 view;
view = glm::lookAt(glm::vec3(0.0f, 0.0f, 3.0f), glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f, 1.0f, 0.0f));
```
## Rotate Around
```C++
const float radius = 10.0f;
float camX = sin(glfwGetTime()) * radius;
float camZ = cos(glfwGetTime()) * radius;
glm::mat4 view;
view = glm::lookAt(glm::vec3(camX, 0.0, camZ), glm::vec3(0.0, 0.0, 0.0), glm::vec3(0.0, 1.0, 0.0));  
```
