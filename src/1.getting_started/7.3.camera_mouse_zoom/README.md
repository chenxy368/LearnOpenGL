# Camera
## Eular Angle
![camera_pitch_yaw_roll](https://user-images.githubusercontent.com/98029669/213593660-726b4d57-ed32-41be-b009-eb8888e8161f.png)  
Note: No Roll for Camera  
![camera_pitch](https://user-images.githubusercontent.com/98029669/213593833-2661be41-9a74-4fae-9af8-b4924e02ad52.png)
```C++
direction.y = sin(glm::radians(pitch));
```
![camera_yaw](https://user-images.githubusercontent.com/98029669/213593914-252e62b2-89dd-4b17-a02d-96c8875658ed.png)
```C++
direction.x = cos(glm::radians(pitch));
direction.z = cos(glm::radians(pitch));
```
![getfront](https://user-images.githubusercontent.com/98029669/213595006-95acc2ca-a60b-407a-9b83-707fe143937a.jpg)

```C++
front.x = cos(glm::radians(pitch)) * cos(glm::radians(yaw));
front.y = sin(glm::radians(pitch));
front.z = cos(glm::radians(pitch)) * sin(glm::radians(yaw));
```
## Mouse Input to Control Front Vector
Hide and capture cursor
```C++
glfwSetInputMode(window, GLFW_CURSOR, GLFW_CURSOR_DISABLED);
...
// Regist Callback
glfwSetCursorPosCallback(window, mouse_callback);
```
Mouse Event CallBack
```C++
// Save cursor position at last frame
float lastX = 400, lastY = 300;
...
void mouse_callback(GLFWwindow* window, double xpos, double ypos)
{
    // Save if first
    if(firstMouse)
    {
        lastX = xpos;
        lastY = ypos;
        firstMouse = false;
    }

    // Compute offset
    float xoffset = xpos - lastX;
    float yoffset = lastY - ypos; 
    lastX = xpos;
    lastY = ypos;

    // Times sensitivity to avoid to fast rotate
    float sensitivity = 0.05;
    xoffset *= sensitivity;
    yoffset *= sensitivity;

    // Add offset to Eular angles
    yaw   += xoffset;
    pitch += yoffset;

    // Boundary check, do not move out
    if(pitch > 89.0f)
        pitch = 89.0f;
    if(pitch < -89.0f)
        pitch = -89.0f;

    // Compute front
    glm::vec3 front;
    front.x = cos(glm::radians(yaw)) * cos(glm::radians(pitch));
    front.y = sin(glm::radians(pitch));
    front.z = sin(glm::radians(yaw)) * cos(glm::radians(pitch));
    cameraFront = glm::normalize(front);
}
```
## Scroll Input to Zoom
Zoom by changing Fov
```C++
void scroll_callback(GLFWwindow* window, double xoffset, double yoffset)
{
  if(fov >= 1.0f && fov <= 45.0f)
    fov -= yoffset;
  if(fov <= 1.0f)
    fov = 1.0f;
  if(fov >= 45.0f)
    fov = 45.0f;
}
```
```C++
projection = glm::perspective(glm::radians(fov), 800.0f / 600.0f, 0.1f, 100.0f);
// Regist Callback
glfwSetScrollCallback(window, scroll_callback);
```
