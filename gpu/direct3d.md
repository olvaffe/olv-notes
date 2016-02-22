Direct3D
========

## Links

* direct3d Pipeline stages
  * `http://msdn.microsoft.com/en-us/library/bb205123(v=vs.85).aspx`
* state objects used by the stages
  * `http://msdn.microsoft.com/en-us/library/bb205071(v=VS.85).aspx`
  * performance considerations

## Input-Assembler Stage

* responsible for supplying data to the pipeline
* assembles vertices into primitives and attaches system-generated values
  primitives
  * SV are such as primitive id, vertex id, instance id
* API reference

    IAGetIndexBuffer  
    IAGetInputLayout    
    IAGetPrimitiveTopology  
    IAGetVertexBuffers  

    IASetIndexBuffer    
    IASetInputLayout    
    IASetPrimitiveTopology  
    IASetVertexBuffers  

## Vertex-Shader Stage

* processes a vertex and output a vertex at a time
  * for transformations, skinning, morphing, and lighting
* see vertex stages
* API reference

    VSGetConstantBuffers        
    VSGetSamplers   
    VSGetShader     
    VSGetShaderResources    

    VSSetConstantBuffers    
    VSSetSamplers   
    VSSetShader     
    VSSetShaderResources    

## Geometry-Shader Stage

* processes a primitive and output zero, one, or more primitives at a time
  * that is, the vertices together with the adjacent vertices of a primitive
* see vertex stages
* API reference: s/VS/GS/ above

## Stream-Output Stage

* streams the primitive data to a buffer.  The rasterization could be enabled or
  disabled while SO is on
* API reference

    SOGetTargets    
    SOSetTargets    

## Rasterizer Stage

* clips the primitives and produces fragments
* is optional (as we may be SO only)
* clipping and divide by w
  * clipping makes sure x, y, z is in `[-1, 1]` after dividing by w
    * for direct3d, z is actually clipped to `[0, 1]`
  * this and the viewport is called post-vs in gallium
* viewport
  * there can be multiple viewports.  but only one of them, which is
    selectable from the gs, can be active
  * viewport transforms clipped coordinates to screen coordinates
* scissoring
  * there can be multiple scissor rectangles.  but only one of them, which is
    selectable from the gs, can be active
  * the scissor rectangle restricts the fragments that are generated
* fill modes and cull modes
  * solid or wireframe
  * cull none, front, or back
* API reference

    RSGetScissorRects   
    RSGetState  
    RSGetViewports  

    RSSetScissorRects   
    RSSetState  
    RSSetViewports  

## Pixel-Shader Stage

* processes a fragment at a time
* see vertex stages
* API reference: s/VS/PS/ above

## Output-Merger Stage

* depth test
* stencil test
* blending
* mrt
  * there are at least 8 rt, all of the same type (buffer, 1d tex, ...), and
    size
* color mask
  * mask out a channel
* sample mask
  * mask out a sample in multisample rendering
* there is no alpha test
  * use the pixel shader for this
* API reference

    OMGetBlendState 
    OMGetDepthStencilState  
    OMGetRenderTargets  

    OMSetBlendState 
    OMSetDepthStencilState  
    OMSetRenderTargets  

## Shader Stages

* vertex shader
  * always active
  * each vertex can have 16 ins and 16 outs
  * there can be 2 system generated values: VertexID and InstanceID
