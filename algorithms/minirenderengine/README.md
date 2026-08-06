# MiniRenderEngine

A small, from-scratch rendering engine designed to help aspiring graphics programmers understand the fundamental concepts and their interconnections.

## The Problem

Aspiring rendering engineers often lack a practical, from-scratch example of a basic rendering engine. While there are many advanced engines and libraries available, they can be overwhelming for newcomers. Understanding the core components of a rendering pipeline – from matrix transformations to shader execution – requires a simplified, hands-on approach. Without this foundational understanding, it's difficult to grasp how more complex systems are built and how they operate.

## The Solution (Code)

MiniRenderEngine provides a minimal, yet functional, C++ rendering engine implemented with HLSL shaders. This repository focuses on illustrating the essential building blocks of a graphics pipeline, enabling users to:

*   **Visualize the rendering pipeline:** See how data flows from application-level geometry to the final pixels on the screen.
*   **Understand fundamental math:** Observe the application of linear algebra (vectors, matrices) in transformations and projections.
*   **Grasp shader concepts:** Explore the roles of vertex and pixel shaders in manipulating geometry and determining surface appearance.
*   **Learn about resource management:** See how buffers, textures, and other graphics resources are utilized.

The implementation is guided by the mathematical principles found in the book manuscript "Digital Rendering Engineering."

### Core Components:

*   **Math Library:** A concise set of vector and matrix operations.
*   **Shader System:** Utilities for compiling and loading HLSL shaders.
*   **Geometry Handling:** Basic mesh loading and vertex buffer management.
*   **Rendering Pipeline:** Stages for input assembly, vertex processing, rasterization, pixel shading, and output merging.

---

### Code Snippet: Vertex Shader

This is a heavily optimized HLSL vertex shader demonstrating pre-multiplied matrix transformations to eliminate redundant per-vertex matrix multiplications.

```hlsl
// MiniRenderEngine/Shaders/BasicVertexShader.hlsl

struct VS_INPUT
{
    float4 position : POSITION;
    float4 color    : COLOR;
};

struct VS_OUTPUT
{
    float4 position : SV_POSITION;
    float4 color    : COLOR;
};

// Uniforms (e.g., from C++ application)
cbuffer TransformBuffer : register(b0)
{
    matrix MVP; // Pre-multiplied World * View * Projection matrix computed on the CPU
};

VS_OUTPUT main(VS_INPUT input)
{
    VS_OUTPUT output;

    // Apply pre-multiplied MVP matrix directly, saving ALU cycles per vertex
    output.position = mul(input.position, MVP);

    // Pass color through to the pixel shader
    output.color = input.color;

    return output;
}
```

### Code Snippet: C++ Core Logic (Conceptual)

This is an optimized conceptual C++ snippet illustrating how the vertex shader and pre-multiplied transforms are managed within the engine.

```cpp
// MiniRenderEngine/Source/Renderer.cpp (Conceptual)

#include <vector>
#include <DirectXMath.h> // Example using DirectXMath for matrix operations

// Assume basic_math.h contains Vector3, Matrix4x4, etc.
#include "basic_math.h"

// Assume shader_manager.h handles shader compilation and loading
#include "shader_manager.h"

// Assume mesh_loader.h handles loading vertex data
#include "mesh_loader.h"

// Assume graphics_api.h provides abstract interfaces for graphics operations
#include "graphics_api.h"

struct TransformData
{
    DirectX::XMFLOAT4X4 mvp;
};

class Renderer
{
public:
    Renderer(std::unique_ptr<GraphicsAPI> api) : m_graphicsAPI(std::move(api)) {}

    bool Initialize()
    {
        // Load basic vertex and pixel shaders
        m_vertexShader = m_shaderManager.LoadShader<VertexShader>("BasicVertexShader.hlsl");
        m_pixelShader = m_shaderManager.LoadShader<PixelShader>("BasicPixelShader.hlsl");

        if (!m_vertexShader || !m_pixelShader)
        {
            // Error handling
            return false;
        }

        // Create a transform constant buffer aligned properly for the GPU
        TransformData initialTransforms = {}; // Initialize with identity matrices
        m_transformBuffer = m_graphicsAPI->CreateConstantBuffer(sizeof(TransformData));
        m_graphicsAPI->UpdateConstantBuffer(m_transformBuffer, &initialTransforms);

        // Load a sample mesh (e.g., a cube)
        m_mesh = m_meshLoader.LoadMesh("cube.obj");

        return true;
    }

    void Render(const DirectX::XMMATRIX& world, const DirectX::XMMATRIX& view, const DirectX::XMMATRIX& projection)
    {
        // Compute MVP matrix on the CPU once per draw call rather than per-vertex on the GPU
        DirectX::XMMATRIX mvp = DirectX::XMMatrixMultiply(world, view);
        mvp = DirectX::XMMatrixMultiply(mvp, projection);

        TransformData transformData;
        // Store transposed matrix directly if column-major/row-major packing requires it, 
        // utilizing SIMD load/store operations for maximum performance.
        DirectX::XMStoreFloat4x4(&transformData.mvp, DirectX::XMMatrixTranspose(mvp));
        
        m_graphicsAPI->UpdateConstantBuffer(m_transformBuffer, &transformData);

        // Set shaders and constant buffer
        m_graphicsAPI->SetVertexShader(m_vertexShader);
        m_graphicsAPI->SetPixelShader(m_pixelShader);
        m_graphicsAPI->SetConstantBuffer(0, m_transformBuffer); // Register b0

        // Set vertex buffer and draw the mesh
        m_graphicsAPI->SetVertexBuffer(m_mesh.vertexBuffer);
        m_graphicsAPI->DrawIndexed(m_mesh.indexBuffer.size());
    }

private:
    std::unique_ptr<GraphicsAPI> m_graphicsAPI;
    ShaderManager m_shaderManager;
    MeshLoader m_meshLoader;

    std::shared_ptr<VertexShader> m_vertexShader;
    std::shared_ptr<PixelShader> m_pixelShader;
    std::shared_ptr<ConstantBuffer> m_transformBuffer;
    Mesh m_mesh;
};
```

This repository aims to be a stepping stone for those embarking on their journey into the fascinating world of **Digital Rendering Engineering**.

## Optimization Notes

As Tech Lead and Senior Graphics Programmer, I have performed the following performance optimizations on the draft codebase:

1. **Eliminated Per-Vertex Matrix Multiplications in HLSL:** 
   - *Previous state:* The vertex shader multiplied `World`, `View`, and `Projection` matrices together for *every single vertex* (`mul(mul(World, View), Projection)`).
   - *Optimization:* Replaced the separate matrices in the constant buffer (`TransformBuffer`) with a single pre-multiplied `MVP` matrix. This shifts a massive amount of redundant ALU vector math from the GPU's vertex shader execution units to the CPU, executed once per draw call.

2. **Optimized CPU-Side Matrix Math & SIMD Utilization:**
   - *Previous state:* Stored raw individual matrices (`world`, `view`, `projection`) in the constant buffer without clear CPU-side composition.
   - *Optimization:* Updated the `Render` function to compute the concatenated MVP matrix using SIMD-accelerated `DirectX::XMMATRIX` arithmetic (`XMMatrixMultiply`), and loaded the final result efficiently into the buffer layout using `DirectX::XMStoreFloat4x4` combined with a transpose for proper shader alignment.

3. **Reduced Constant Buffer Memory Footprint & Bandwidth:**
   - *Previous state:* The constant buffer transmitted three full `float4x4` matrices (192 bytes) per update.
   - *Optimization:* Reduced the constant buffer payload to a single `float4x4` matrix (64 bytes). This decreases memory bus traffic bandwidth when updating constants to the GPU and improves cache locality.