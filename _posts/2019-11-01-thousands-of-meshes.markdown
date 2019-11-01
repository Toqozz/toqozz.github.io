---
layout: post
title: "Drawing thousands of meshes with DrawMeshInstanced/Indirect in Unity"
date: 2019-11-01
categories:
---

# Drawing (hundreds of) thousands of meshes in Unity
![Demo gif](/assets/2019_mesh_spiral.gif)

GPU instancing is a technique offered by Unity to draw lots of the same mesh and material quickly.

In the right circumstances, GPU instancing can allow you to feasibly draw even millions of meshes.  Unity tries to make this work automatically for you if it can.  If all your meshes use the same material, 'GPU Instancing' is ticked, your shader supports instancing, lighting and shadows play nicely, you're not using a skinned mesh renderer, etc, Unity will automatically batch meshes into a single draw call.
> A low number of draw calls is usually a sign of a well-performing game.  For each new draw call, the GPU has to do a context switch, which is *expensive*.

There's a lot of ifs and buts here, which is a pain in the ass if you just want to write performant code, and even if you do get GPU instancing to work, the overhead of `GameObject`s and `Transform`s alone is *huge*.

---

# [`DrawMeshInstanced`](https://docs.unity3d.com/ScriptReference/Graphics.DrawMeshInstanced.html)
You can use `Graphics.DrawMeshInstanced` to get around a lot of these conditions.  This will draw a number of meshes (up to 1023 in a single batch) for a single frame.

This is a particularly nice solution for when you want to draw a lot of objects that don't move very much, or only move in the shader (trees, grass).  It allows you to easily shove meshes to the GPU, customize them with `MaterialPropertyBlock`s, and avoid the fat overhead of `GameObject`s.  Additionally, Unity has to do a little less work to figure out if it can instance the objects or not, and will throw an error rather than silently nerfing performance.  The main downside here is that moving these objects usually results in a huge `for` loop, which kills performance.

> If you *must* move meshes while using `DrawMeshInstanced`, consider using a sleep/wake model to reduce the size of these loops as much as possible.

### Example
```cs
public class DrawMeshInstancedDemo : MonoBehaviour {
    // How many meshes to draw.
    public int population;
    // Range to draw meshes within.
    public float range;

    // Material to use for drawing the meshes.
    public Material material;
  
    private Matrix4x4[] matrices;
    private MaterialPropertyBlock block;

    private Mesh mesh;

    private void Setup() {
        Mesh mesh = CreateQuad();
        this.mesh = mesh;

        matrices = new Matrix4x4[population];
        Vector4[] colors = new Vector4[population];

        block = new MaterialPropertyBlock();

        for (int i = 0; i < population; i++) {
            // Build matrix.
            Vector3 position = new Vector3(Random.Range(-range, range), Random.Range(-range, range), Random.Range(-range, range));
            Quaternion rotation = Quaternion.Euler(Random.Range(-180, 180), Random.Range(-180, 180), Random.Range(-180, 180));
            Vector3 scale = Vector3.one;

            mat = Matrix4x4.TRS(position, rotation, scale);

            matrices[i] = mat;

            colors[i] = Color.Lerp(Color.red, Color.blue, Random.value);
        }

        // Custom shader needed to read these!!
        block.SetVectorArray("_Colors", colors);
    }

    private Mesh CreateQuad(float width = 1f, float height = 1f) {
        // Create a quad mesh.
        // See source for implementation.
    }

    private void Start() {
        Setup();
    }

    private void Update() {
        // Draw a bunch of meshes each frame.
        Graphics.DrawMeshInstanced(mesh, 0, material, matrices, population, block);
    }
}
```

Essentially, we fill a big array with matrices representing object transforms (position, rotation, scale) and then pass that array to `Graphics.DrawMeshInstanced`, which will assign those matrices in the shader automagically (assuming the shader supports instancing).

You can still grab the instanceID to do per-mesh customizations via `MaterialPropertyBlock`s (see color array), but you'll probably need to use a custom shader:

> *A custom shader is only required if you're customizing per-mesh properties;*

```c
Shader "Custom/InstancedColor" {
    SubShader {
        Tags { "RenderType" = "Opaque" }

        Pass {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_instancing
            
            #include "UnityCG.cginc"
            
            struct appdata_t {
                float4 vertex   : POSITION;
                float4 color    : COLOR;
                float2 texcoord : TEXCOORD0;
                UNITY_VERTEX_INPUT_INSTANCE_ID
            };

            struct v2f {
                float4 vertex   : SV_POSITION;
                fixed4 color    : COLOR;
                float2 texcoord : TEXCOORD0;
            }; 

            float4 _Colors[1023];   // Max instanced batch size.

            v2f vert(appdata_t i, uint instanceID: SV_InstanceID) {
                // Allow instancing.
                UNITY_SETUP_INSTANCE_ID(i);

                v2f o;
                o.vertex = UnityObjectToClipPos(i.vertex);
                o.color = float4(1, 1, 1, 1);

                // If instancing on (it should be) assign per-instance color.
                #ifdef UNITY_INSTANCING_ENABLED
                    o.color = _Colors[instanceID];
                #endif
                
                return o;
            }
            
            fixed4 frag(v2f i) : SV_Target {
                return i.color;
            }
            
            ENDCG
        }
    }
}
```

![1022 meshes with standard shader](/assets/2019_1022_uncolored.png)
*1022 meshes with the standard shader.*
![1022 meshes with standard shader](/assets/2019_1022_colored.png)
*1022 meshes with a custom shader, and per-mesh colors*

> Note that shadows are missing on the colored version.  This is because we're using a custom shader to apply the colors which, doesn't have a shadow pass.  Have a look at [this](https://docs.unity3d.com/Manual/SL-VertexFragmentShaderExamples.html) for some examples of adding shadow casting/receiving to a custom shader.

> If setting up a random array in the shader feels awkward, that's because it is.  There doesn't seem to be a way to get Unity to set up an array for you and index the color automatically.  If we were using individual game objects, we could do something like [this](https://docs.unity3d.com/540/Documentation/Manual/GPUInstancing.html).  You could probably get this to work by digging into the shader source and having a look at what names Unity uses for the arrays, but that's pretty convoluted for no good reason.


# [`DrawMeshInstancedIndirect`](https://docs.unity3d.com/ScriptReference/Graphics.DrawMeshInstancedIndirect.html)
`DrawMeshInstanced` turns out to be a sort of wrapper around `DrawMeshInstancedIndirect`.  You can achieve everything in the latter that you can with the former (and vice versa, with complications).  `DrawMeshInstanced` is mainly a friendly way to draw meshes without touching the GPU.

Naturally, some nice things get lost in the abstraction.  First of all, the `Indirect` variant allows you to bypass the 1023 mesh limit and draw as many meshes as you like in a single batch (the 1023 mesh limit seems to actually inherited from [`MaterialPropertyBlock`](https://docs.unity3d.com/ScriptReference/MaterialPropertyBlock.CopyProbeOcclusionArrayFrom.html)).  The primary benefit, however, is that you can offload the entirety of the work onto the GPU.  With `DrawMeshInstanced`, Unity has to upload the array of mesh matrices to the GPU each frame, whereas `Indirect` creates and stores data on the GPU indefinitely.  This also means using GPU-based structures to store data, mainly [`ComputeBuffer`s](https://docs.unity3d.com/ScriptReference/ComputeBuffer.html), which can be scary up front but is actually more convenient and opens the door to some easy mass parallelisation via compute shaders.

### Example
Here's the same program as above but using `DrawMeshInstancedIndirect` instead:
```cs
public class DrawMeshInstancedIndirectDemo : MonoBehaviour {
    public int population;
    public float range;

    public Material material;

    private ComputeBuffer meshPropertiesBuffer;
    private ComputeBuffer argsBuffer;

    private Mesh mesh;
    private Bounds bounds;

    // Mesh Properties struct to be read from the GPU.
    // Size() is a convenience funciton which returns the stride of the struct.
    private struct MeshProperties {
        public Matrix4x4 mat;
        public Vector4 color;

        public static int Size() {
            return
                sizeof(float) * 4 * 4 + // matrix;
                sizeof(float) * 4;      // color;
        }
    }

    private void Setup() {
        Mesh mesh = CreateQuad();
        this.mesh = mesh;

        // Boundary surrounding the meshes we will be drawing.  Used for occlusion.
        bounds = new Bounds(transform.position, Vector3.one * (range + 1));

        InitializeBuffers();
    }

    private void InitializeBuffers() {
        // Argument buffer used by DrawMeshInstancedIndirect.
        uint[] args = new uint[5] { 0, 0, 0, 0, 0 };
        // Arguments for drawing mesh.
        // 0 == number of triangle indices, 1 == population, others are only relevant if drawing submeshes.
        args[0] = (uint)mesh.GetIndexCount(0);
        args[1] = (uint)population;
        args[2] = (uint)mesh.GetIndexStart(0);
        args[3] = (uint)mesh.GetBaseVertex(0);
        argsBuffer = new ComputeBuffer(1, args.Length * sizeof(uint), ComputeBufferType.IndirectArguments);
        argsBuffer.SetData(args);

        // Initialize buffer with the given population.
        MeshProperties[] properties = new MeshProperties[population];
        for (int i = 0; i < population; i++) {
            MeshProperties props = new MeshProperties();
            Vector3 position = new Vector3(Random.Range(-range, range), Random.Range(-range, range), Random.Range(-range, range));
            Quaternion rotation = Quaternion.Euler(Random.Range(-180, 180), Random.Range(-180, 180), Random.Range(-180, 180));
            Vector3 scale = Vector3.one;

            props.mat = Matrix4x4.TRS(position, rotation, scale);
            props.color = Color.Lerp(Color.red, Color.blue, Random.value);

            properties[i] = props;
        }

        meshPropertiesBuffer = new ComputeBuffer(population, MeshProperties.Size());
        meshPropertiesBuffer.SetData(properties);
        material.SetBuffer("_Properties", meshPropertiesBuffer);
    }

    private Mesh CreateQuad(float width = 1f, float height = 1f) {
        ...
    }

    private void Start() {
        Setup();
    }

    private void Update() {
        Graphics.DrawMeshInstancedIndirect(mesh, 0, material, bounds, argsBuffer);
    }

    private void OnDisable() {
        // Release gracefully.
        if (meshPropertiesBuffer != null) {
            meshPropertiesBuffer.Release();
        }
        meshPropertiesBuffer = null;

        if (argsBuffer != null) {
            argsBuffer.Release();
        }
        argsBuffer = null;
    }
}
```
> The `bounds` parameter of `DrawMeshInstancedIndirect` is used for determining whether the mesh is in view and culling it.  The documentation states  `Meshes are not further culled by the view frustum or baked occluders ...` which gave me the impression that culling was simply disabled for all meshes drawn with DrawMeshInstanced/Indirect, but this is not so.  Unity will cull all the instanced meshes if the provided mesh bounds are not in view.  It will not, however, cull individual instanced meshes -- you'll have to calculate this yourself if you find it necessary.

And the associated shader:
```c
Shader "Custom/InstancedIndirectColor" {
    SubShader {
        Tags { "RenderType" = "Opaque" }

        Pass {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            #include "UnityCG.cginc"
            
            struct appdata_t {
                float4 vertex   : POSITION;
                float4 color    : COLOR;
            };

            struct v2f {
                float4 vertex   : SV_POSITION;
                fixed4 color    : COLOR;
            }; 

            struct MeshProperties {
                float4x4 mat;
                float4 color;
            };

            StructuredBuffer<MeshProperties> _Properties;

            v2f vert(appdata_t i, uint instanceID: SV_InstanceID) {
                v2f o;

                float4 pos = mul(_Properties[instanceID].mat, i.vertex);
                o.vertex = UnityObjectToClipPos(pos);
                o.color = _Properties[instanceID].color;

                return o;
            }
            
            fixed4 frag(v2f i) : SV_Target {
                return i.color;
            }
            
            ENDCG
        }
    }
}
```
> The struct for your `StructuredBuffer` **must** be byte-wise identical throughout shader/compute shader/script or you'll see bugginess.  These structures are only matched up between script and shader by reading bytes.

![1022 meshes using DrawMeshInstancedIndirect](/assets/2019_1022_indirect_colored.png)
*1022 meshes drawn with `DrawMeshInstancedIndirect`*

A key difference here is that with `DrawMeshInstanced`, we were giving Unity an array of matrices and having it automagically figure out vertex positions before we got to the shader.  Here, we're being much more direct in that we're pushing the matrices to the GPU and applying the transformation ourselves.  The shader instancing code has been cut down significantly, which is nice, and we've gained about a millisecond in rendering.
Note, however, that this number is heavily influenced by the rest of the game.  In our basic scene with nothing happening, bandwidth and CPU time is abundant, so the benefits of `Indirect` are less prevalent than in the real world.

The 1023 mesh limit has also disappeared.  Pushing the population up to even 100, 000 has little affect on my system (Ryzen 1700, 1080Ti):

![100k meshes using DrawMeshInstancedIndirect](/assets/2019_100k_indirect_colored.png)
*100k meshes drawn with `DrawMeshInstancedIndirect`*

---

# Adding movement with a compute shader
Now that we're using `DrawMeshInstancedIndirect`, we can add some mesh movement without crushing performance.

[Compute Shaders](https://docs.unity3d.com/Manual/class-ComputeShader.html) are special programs that run on the GPU and allow you to utilize the massive parallel power of graphics devices for non-graphics code.  I'm mostly looking to demonstrate how you can use a compute shader with the other tools shown so far, so I won't explain compute shaders too in-depth; [here](http://kylehalladay.com/blog/tutorial/2014/06/27/Compute-Shaders-Are-Nifty.html) is a good blog post talking about them, which covers everything better than I could.

Here's a compute shader which moves our meshes based on some "pusher":
```c
#pragma kernel CSMain

struct MeshProperties {
    float4x4 mat;
    float4 color;
};

RWStructuredBuffer<MeshProperties> _Properties;
float3 _PusherPosition;

// For the sake of simplicity, only using 1, 1, 1 threads.
[numthreads(1,1,1)]
void CSMain (uint3 id : SV_DispatchThreadID) {
    float4x4 mat = _Properties[id.x].mat;
    // In a transform matrix, the position (translation) vector is the last column.
    float3 position = float3(mat[0][3], mat[1][3], mat[2][3]);

    float dist = distance(position, _PusherPosition);
    // Scale and reverse distance so that we get a value which fades as it gets further away.
    // Max distance is 5.0.
    dist = 5.0 - clamp(0.0, 5.0, dist);

    // Get the vector from the pusher to the position, and scale it.
    float3 push = normalize(position - _PusherPosition) * dist;
    // Create a new translation matrix which represents a move in a direction.
    float4x4 translation = float4x4(
        1, 0, 0, push.x,
        0, 1, 0, push.y,
        0, 0, 1, push.z,
        0, 0, 0, 1
    );

    // Apply translation to existing matrix, which will be read in the shader.
    _Properties[id.x].mat = mul(translation, mat);
}
```

Take note that the `MeshProperties` struct is byte-wise identical to the structs in the other files.  I mentioned this earlier, but it's paramount that you keep these structs in sync or your program won't work (without error!).

You might also be wondering whether you could accomplish this same thing in the regular shader as well, by changing the `StructuredBuffer` to a `RWStructuredBuffer` and doing a similar computation.  The problem this introduces is that regular shaders are designed to work on a per-vertex or per-fragment basis, and so to get the same effect (moving an individual *mesh*) you'd have to either set some flag to mark the mesh as "moved" (horrible, absolutely kills parallelisation), or change the per-mesh approach to a per-vertex approach, probably with a different result.
While I'm sure it's possible to achieve the same thing without a compute shader, my point is that compute shaders let you specify granularity on your own terms.

Of course, we need to make a couple additions to our script to actually run our compute shader.  Where `compute` is our compute shader and `pusher` is the object we want to use to push:
```cs
public class DrawMeshInstancedIndirectDemo : MonoBehaviour {
    public int population;
    public float range;

    public Material material;
++  public ComputeShader compute;
++  public Transform pusher;

    private ComputeBuffer meshPropertiesBuffer;
    private ComputeBuffer argsBuffer;

    private Mesh mesh;
    private Bounds bounds;

    ...

    private void InitializeBuffers() {
++      int kernel = compute.FindKernel("CSMain");

        // Argument buffer used by DrawMeshInstancedIndirect.
        uint[] args = new uint[5] { 0, 0, 0, 0, 0 };
        // Arguments for drawing mesh.
        // 0 == number of triangle indices, 1 == population, others are only relevant if drawing submeshes.
        args[0] = (uint)mesh.GetIndexCount(0);
        ...

        meshPropertiesBuffer = new ComputeBuffer(population, MeshProperties.Size());
        meshPropertiesBuffer.SetData(properties);
++      compute.SetBuffer(kernel, "_Properties", meshPropertiesBuffer);
        material.SetBuffer("_Properties", meshPropertiesBuffer);
    }

...

    private void Update() {
++      int kernel = compute.FindKernel("CSMain");

++      compute.SetVector("_PusherPosition", pusher.position);
++      compute.Dispatch(kernel, population, 1, 1);
        Graphics.DrawMeshInstancedIndirect(mesh, 0, material, bounds, argsBuffer);
    }
}
```

![Pushing a lot of meshes around](/assets/2019_compute_movement.gif)

And that's all, really.  Now you know how to move and draw (hundreds of) thousands of meshes efficiently.
