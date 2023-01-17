From vertex shader to fragment shader: 
```shell
...
out vec3 ourPosition;
...
    ourPosition = aPos;
```

```shell
...
in vec3 ourPosition;
...
    FragColor = vec4(ourPosition, 1.0);    // note how the position value is linearly interpolated to get all the different colors
```
