## Normal texture

Basic texture in Android, such as a picture, have two options for share texture with unity. One is to run directly in the rendering thread of unity and render in the rendering thread of unity. The other is to share textures by share context

## Texture for video stream RTT

This kind of texture represents the video stream data of the camera or the texture data of the screen projection of the mobile phone. There are also two ways

Take the screen projection of the mobile phone as an example

First create the textureID and the corresponding surfaceTexture and surface, and then bind it to the virtual display, so that the rendered texture is bound to the textureID

Secondly, because unity cannot directly use the OES format of openGL ES, when rendering, it is necessary to create a new texture id of texture2D type, and then bind this id through FBO, so that what is drawn under OES will be drawn Go to the texture ID of texture2D, and then give this texture ID to unity, and unity renders

Unity cannot directly use the OES format of openGL ES, but Unity can receive the OES format, as long as Unity itself handles it through shader, so the process will be simpler, because the rendering is done in Unity

First, get the texture data of the virtual display as above

Secondly, directly give this texture ID to unity, and unity binds a shader in the script, and renders through this shader
