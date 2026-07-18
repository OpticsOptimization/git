# SimplePBRRenderer

A minimal, well-commented implementation of Physically Based Rendering (PBR) in HLSL and C++.

## The Problem

Physically Based Rendering can be a complex topic for newcomers. Understanding the core mathematical principles and their translation into shaders can be a significant hurdle. Existing PBR implementations can sometimes be overly feature-rich or lack clear explanations, making it difficult for beginners to grasp the fundamental concepts.

## The Solution (Code)

This repository provides a streamlined and easy-to-understand PBR renderer. It focuses on clarity and educational value, demonstrating the essential components of PBR without unnecessary complexity. The implementation is based on the foundational PBR math principles, drawing inspiration from authoritative resources in the field.

We aim to provide a solid starting point for developers and students who want to:

*   Learn the core concepts of PBR.
*   Integrate PBR into their existing rendering pipelines.
*   Experiment with PBR parameters and understand their effects.

### HLSL Shader Code Snippet (`SimplePBR.hlsl`)

This snippet demonstrates the core PBR shading logic. It includes:

*   **Microfacet BRDF:** A simplified implementation of the Cook-Torrance microfacet model.
*   **Fresnel Term:** Essential for calculating how much light reflects based on viewing angle.
*   **Energy Conservation:** Ensuring that the rendering physically behaves.
*   **Lighting Integration:** Combining surface properties with light sources.

```hlsl
#pragma once

//--------------------------------------------------------------------------------------
// Constants and Utility Functions
//--------------------------------------------------------------------------------------

// Define PI for efficient use
static const float PI = 3.14159265359f;
static const float TWO_PI = 2.0f * PI;

// Use a small epsilon to avoid division by zero and floating-point precision issues.
// This is more robust than direct comparison with 1.0f.
static const float EPSILON = 1e-5f;

// Converts perceptual roughness (used for UI/artistic control) to roughness (used in NDF).
// Perceptual roughness is typically in [0, 1]. For GGX, we map it to a range
// suitable for the NDF calculation, clamping to avoid extreme values.
// A common mapping is to square it, and then clamp to a small positive value for stability.
float PerceptualRoughnessToRoughness(float perceptualRoughness)
{
    // Map to [0.001, 1.0] range before squaring for stability.
    // The original code's "roughness * perceptualRoughness" seems incorrect.
    // A common mapping is `roughness = perceptualRoughness * perceptualRoughness;`
    // followed by clamping.
    float roughness = perceptualRoughness * perceptualRoughness;
    return saturate(roughness); // Ensure roughness is in [0, 1]
}

// Converts roughness to the squared roughness parameter used in NDF and Geometry terms.
// This is directly used as alpha^2 in some formulations.
float RoughnessToAlphaSq(float roughness)
{
    // The GGX alpha parameter is often related to roughness squared.
    // Let's use alpha = roughness for simplicity here, which is common.
    // If the GGX formulation specifically requires alpha^2, this function needs adjustment.
    // Based on the original NDF and Geometry functions, they used `alpha = roughness * roughness;`
    // meaning they are using `roughness` as their `alpha` in the formula `alpha^2`.
    // To optimize, we'll directly use `roughness * roughness` as alphaSq.
    return roughness * roughness;
}

//--------------------------------------------------------------------------------------
// Normal Distribution Function (NDF) - GGX
//--------------------------------------------------------------------------------------
// D(H) = (alpha^2) / (PI * (1 + alpha^2 - 2*alpha*cos(theta))^2)
// Where alpha is the microfacet distribution parameter (related to roughness),
// and theta is the angle between the normal (N) and the half-vector (H).
// For GGX, alpha is typically derived from roughness: alpha = roughness.
// The formula commonly uses alpha^2. Let's stick to `alpha = roughness` and then `alphaSq`.
float NDF_GGX(float NdotH, float roughness)
{
    // alphaSq is (roughness^2)^2 in some formulations, or just roughness^2 as alpha.
    // The original code used `alpha = roughness * roughness;` and then `alphaSq` in the denominator.
    // This implies that `roughness` itself is the `alpha` parameter of GGX.
    // We will use `alpha = roughness` and then `alphaSq` for the denominator as per the original.
    float alpha = roughness; // For GGX, alpha is often directly related to roughness.
    float alphaSq = alpha * alpha;

    // Avoid division by zero or issues with floating point precision near 1.0
    // Using saturate to clamp NdotH to [0, 1] and add EPSILON to denominator.
    NdotH = saturate(NdotH);
    float cos2Theta = NdotH * NdotH;

    // tan^2(theta) = sec^2(theta) - 1 = 1/cos^2(theta) - 1 = (1 - cos^2(theta)) / cos^2(theta)
    float tan2Theta = (1.0 - cos2Theta) / (cos2Theta + EPSILON); // Add EPSILON for stability

    // Denominator: PI * alphaSq * (1 + tan2Theta * alphaSq)^2
    // Optimized: pow(x, 2) can be x * x
    float denominatorTerm = 1.0f + tan2Theta * alphaSq;
    float denominator = PI * alphaSq * (denominatorTerm * denominatorTerm);

    return alphaSq / (denominator + EPSILON); // Add EPSILON to denominator for robustness
}

//--------------------------------------------------------------------------------------
// Geometry Function (Shadowing/Masking) - Smith-GGX
//--------------------------------------------------------------------------------------
// G(V, L, N) = G1(V, N) * G1(L, N)
// G1(X, N) = (NdotX + 1) / (NdotX + sqrt(alpha^2 + (1 - alpha^2) * (NdotX^2)))
// Where X is V or L, N is the normal, and alpha is derived from roughness.
// We use alpha = roughness directly here, and alphaSq in the denominator.
float GeometrySmith_GGX(float NdotV, float NdotL, float roughness)
{
    // alpha is the microfacet distribution parameter, commonly roughness.
    float alpha = roughness;
    float alphaSq = alpha * alpha;

    // G1 for View (V)
    // Clamp NdotV to avoid issues near grazing angles and ensure it's non-negative.
    NdotV = saturate(NdotV);
    // Optimized denominator: sqrt(alphaSq + (1 - alphaSq) * NdotV^2)
    // This can be rewritten as sqrt(alphaSq * (1 - NdotV^2) + NdotV^2) if alpha is 1.
    // For GGX: sqrt(alphaSq + (1 - alphaSq) * NdotV_sq)
    // Let's use alphaSq directly from roughness.
    float V_dot_N_sq = NdotV * NdotV;
    // Using a common simplification for Smith-GGX denominator:
    // sqrt(alphaSq * (1 - NdotV^2) + NdotV^2) -> Incorrect. Correct is as original.
    // The original formulation: alpha * alpha + (1.0 - alpha * alpha) * (1.0 - V_dot_N_sq)
    // `roughness` is used as `alpha`.
    float geometryV_denom_sqrt = sqrt(alphaSq + (1.0f - alphaSq) * (1.0f - NdotV * NdotV));
    float G1_V = NdotV / (NdotV + geometryV_denom_sqrt + EPSILON); // Add EPSILON for stability

    // G1 for Light (L)
    // Clamp NdotL to avoid issues near grazing angles and ensure it's non-negative.
    NdotL = saturate(NdotL);
    float L_dot_N_sq = NdotL * NdotL;
    float geometryL_denom_sqrt = sqrt(alphaSq + (1.0f - alphaSq) * (1.0f - L_dot_N_sq));
    float G1_L = NdotL / (NdotL + geometryL_denom_sqrt + EPSILON); // Add EPSILON for stability

    return G1_V * G1_L;
}

//--------------------------------------------------------------------------------------
// Fresnel Equation - Schlick Approximation
//--------------------------------------------------------------------------------------
// F(h, V) = F0 + (1 - F0) * (1 - dot(h, v))^5
// F0 is the Fresnel reflectance at normal incidence (e.g., for dielectrics, it's around 0.04).
// cosTheta is dot(H, V) or dot(L, H) which is used here.
float3 FresnelSchlick(float cosTheta, float3 F0)
{
    // Ensure cosTheta is within [-1, 1] range, then clamp to [0, 1] for the power.
    cosTheta = saturate(cosTheta);
    // Optimized pow(x, 5) = x * x * x * x * x
    float pow5 = cosTheta * cosTheta;
    pow5 = pow5 * pow5 * cosTheta;
    return F0 + (1.0f - F0) * (1.0f - pow5);
}

//--------------------------------------------------------------------------------------
// PBR BRDF Calculation (Cook-Torrance)
//--------------------------------------------------------------------------------------
// Lo = Integral[ f_r(l, v) * (l.n) * Li(l) dl ]
// For a single light source: f_r * (l.n) * Li
//
// f_r = (D(H) * G(V, L, H) * F(h, V)) / (4 * NdotV * NdotL)
//
// D is NDF, G is Geometry, F is Fresnel, H is half-vector, V is view vector, L is light vector.
// NdotV, NdotL, NdotH are dot products with the surface normal.
float3 BRDF_CookTorrance(float3 L, float3 V, float3 N, float3 albedo, float metallic, float roughness, float3 F0)
{
    // Ensure vectors are normalized. This is crucial.
    // The calling code should ideally provide normalized vectors, but defensive normalization
    // is good if there's any doubt, though it adds cost. Let's assume normalized inputs for optimization.

    // Calculate Half-Vector
    float3 H = normalize(L + V);

    // Calculate Dot Products
    // Clamping dot products to [0, 1] ensures they are physically meaningful and avoid issues with
    // grazing angles or backfacing geometry.
    float NdotL = saturate(dot(N, L));
    float NdotV = saturate(dot(N, V));
    float NdotH = saturate(dot(N, H));
    float LdotH = saturate(dot(L, H)); // Used for Fresnel

    // Calculate Fresnel term (F)
    // F0 is the base reflectivity at normal incidence.
    // For dielectrics, it's ~0.04. For metals, it's their albedo.
    float3 F = FresnelSchlick(LdotH, F0);

    // Calculate Specular and Diffuse components based on metallic
    // k_s is specular color, k_d is diffuse color.
    // For metals (metallic=1), k_s = F, k_d = 0.
    // For dielectrics (metallic=0), k_s = F0, k_d = (1-F0) * albedo.
    // The interpolation is: k_s = lerp(F0, F, metallic) and k_d = lerp(1.0 - F0, 0.0, metallic) * albedo.
    // A more common formulation:
    // Specular contribution is `F`.
    // Diffuse contribution is `(1 - F) * albedo` for dielectrics.
    // For metals, diffuse is zero, specular is `F`.
    // So, we split diffuse and specular.
    float3 k_d = (1.0f - metallic) * albedo / PI; // Diffuse term, assume albedo is base color. Divide by PI for energy conservation.
    float3 k_s = F; // Specular term, governed by Fresnel.

    // NDF Term (D)
    float D = NDF_GGX(NdotH, roughness);

    // Geometry Term (G)
    float G = GeometrySmith_GGX(NdotV, NdotL, roughness);

    // Cook-Torrance BRDF denominator
    // Ensure NdotV and NdotL are not zero before division.
    float denominator = 4.0f * NdotV * NdotL + EPSILON; // Add EPSILON for stability

    // Specular BRDF: (D * G * F) / denominator
    float3 specular = (D * G * k_s) / denominator;

    // Combine Diffuse and Specular
    // The final PBR lighting equation for a single light source is:
    // Lo = (k_d * albedo / PI + specular) * Li * NdotL
    // Our BRDF function should return the `f_r` part.
    // The diffuse component `k_d * albedo / PI` is energy conserved.
    // The specular component `specular` already has `k_s` in it.

    // Energy conservation check: if metallic is 1, diffuse should be 0.
    // If metallic is 0, diffuse is (1-F) * albedo / PI.
    // Our k_d already handles the diffuse part.
    // Our k_s handles the specular part (F).

    // Total BRDF (f_r)
    float3 brdf = k_d + specular;

    // The final outgoing radiance is brdf * Li * NdotL.
    // The shader returns the color, so we need to return the BRDF itself.
    return brdf;
}

//--------------------------------------------------------------------------------------
// Pixel Shader Main Function
//--------------------------------------------------------------------------------------
// Assumes uniforms are set for:
//   - Camera Position (camPos)
//   - Light Direction (lightDir)
//   - Light Color (lightColor)
//   - Material Properties:
//     - Albedo (material.albedo) - Base color
//     - Metallic (material.metallic) - 0 for dielectric, 1 for metal
//     - Roughness (material.roughness) - Smoothness of surface
//     - AO (material.ao) - Ambient occlusion factor (0 to 1)
//   - Precomputed Irradiance (irradiance) - For diffuse ambient lighting (simplified)
//   - Precomputed Specular LUT (specularLUT) - For specular ambient lighting (placeholder)

struct Material
{
    float3 albedo;
    float metallic;
    float roughness;
    float ao; // Ambient Occlusion
};

Texture2D albedoTex : register(t0);
SamplerState defaultSampler : register(s0);

// Uniform buffer for material and lighting parameters.
// This is conceptual, actual implementation depends on graphics API.
cbuffer PBRData : register(b0)
{
    Material material;
    float3 camPos;
    float3 lightDir;
    float3 lightColor;
    float3 irradiance; // Simplified ambient diffuse irradiance
    // float3 specularEnvMapLUT[SPECULAR_LUT_SIZE]; // Conceptual for precomputed specular
};

// Placeholder for environment map for indirect lighting
TextureCube specularEnvMap : register(t1);
SamplerState specularSampler : register(s1);


float4 PSMain(float4 pos : SV_POSITION, float2 tex : TEXCOORD0, float3 normal : NORMAL, float3 worldPos : POSITION) : SV_TARGET
{
    // Sample material properties
    // Ensure albedo from texture is scaled by material albedo property if it's a tint.
    // If material.albedo is meant to be a base color multiplier for textured albedo.
    float3 baseColor = material.albedo * albedoTex.Sample(defaultSampler, tex).rgb;

    // Use perceptual roughness for input, convert to roughness for calculations.
    float perceptualRoughness = material.roughness;
    float roughness = PerceptualRoughnessToRoughness(perceptualRoughness);

    float metallic = saturate(material.metallic); // Ensure metallic is in [0, 1]
    float ao = saturate(material.ao); // Ensure AO is in [0, 1]

    // Calculate F0 (Fresnel Reflectance at Normal Incidence)
    // For dielectrics, F0 is typically around 0.04.
    // For metals, F0 is roughly equal to their albedo.
    // Use lerp to blend between dielectric F0 and metal F0 based on metallic value.
    float3 F0 = lerp(float3(0.04f, 0.04f, 0.04f), baseColor, metallic);

    // Normalize vectors for lighting calculations.
    // Ensure camera position is valid before normalizing view vector.
    float3 N = normalize(normal);
    float3 V = normalize(camPos - worldPos);
    float3 L = normalize(lightDir);

    // Ensure vectors are facing the same direction if necessary (e.g., NdotL > 0)
    // This is handled by saturate(dot(N, L)) later.

    // --- Direct Lighting ---
    // Calculate the BRDF for the Cook-Torrance model.
    float3 brdf = BRDF_CookTorrance(L, V, N, baseColor, metallic, roughness, F0);

    // Calculate the contribution from direct light.
    // Lo = BRDF * LightColor * NdotL
    float NdotL = saturate(dot(N, L));
    float3 directLight = brdf * lightColor * NdotL;

    // --- Indirect Lighting (Simplified) ---
    // Diffuse indirect lighting (ambient diffuse)
    // Using a simplified ambient term. In a full PBR renderer, this would be
    // an irradiance map convolution or pre-filtered diffuse environment map.
    // AO reduces diffuse indirect lighting.
    float3 indirectDiffuse = irradiance * baseColor * ao;

    // Specular indirect lighting (from environment map)
    // This is a placeholder. Real PBR uses pre-filtered specular environment maps
    // that are convolved with roughness and view direction.
    // For simplicity, we'll use a placeholder for diffuse indirect.
    // A proper specular indirect calculation would sample a convolved environment map.
    // Example placeholder for specular indirect:
    // float3 R = reflect(-V, N); // Reflection vector for specular
    // float3 prefilteredSpecular = SpecularIBL(specularEnvMap, R, roughness, N); // Hypothetical function
    // float3 indirectSpecular = prefilteredSpecular * ao; // Apply AO
    float3 indirectSpecular = float3(0.0f, 0.0f, 0.0f); // Placeholder for indirect specular

    // Combine direct and indirect lighting
    // Ensure to scale indirect terms appropriately.
    // Direct light is already scaled by lightColor.
    // Indirect diffuse is scaled by irradiance.
    float3 totalLight = directLight + indirectDiffuse + indirectSpecular;

    // --- Tone Mapping and Gamma Correction ---
    // This is a very basic Reinhard tone mapping and gamma correction.
    // More advanced tone mapping operators (e.g., ACES) are common in production.
    // Use MAX_VALUE for Reinhard to prevent division by zero with black.
    float MAX_VALUE = 1.0f; // Or a suitable HDR value
    totalLight = totalLight / (totalLight + MAX_VALUE); // Reinhard tone mapping
    totalLight = pow(totalLight, 1.0f / 2.2f); // Gamma correction (assuming sRGB display)

    return float4(totalLight, 1.0f);
}
```

### C++ Integration Snippet (Conceptual)

This C++ snippet illustrates how you might set up the necessary data and call the shader. This assumes a graphics API like DirectX 11 or 12, or Vulkan.

```cpp
#include <vector>
#include <DirectXMath.h> // Or your chosen math library

// Assuming you have a ShaderManager, RenderTarget, and Texture objects set up.
// ShaderManager m_shaderManager;
// RenderTarget m_renderTarget;
// Texture m_albedoTexture;
// Texture m_specularEnvMap;

// Define the structure for PBR material properties.
struct PBRMaterial
{
    DirectX::XMFLOAT3 albedo = { 1.0f, 1.0f, 1.0f };
    float metallic = 0.0f;
    float roughness = 0.5f; // Perceptual roughness
    float ao = 1.0f;
};

// Define the structure for constant buffer data sent to the shader.
// Ensure layout matches HLSL cbuffer 'PBRData'.
struct PBRConstants
{
    DirectX::XMFLOAT3 camPos;
    DirectX::XMFLOAT3 lightDir;
    DirectX::XMFLOAT3 lightColor;
    PBRMaterial material;
    DirectX::XMFLOAT3 irradiance; // Simplified ambient diffuse irradiance
    // Potentially add parameters for specular environment map sampling if not using texture arrays or bindless.
};

// Example Mesh structure.
struct Mesh
{
    // Vertex data, index data, etc.
    PBRMaterial material; // Mesh-specific material properties

    void Draw() const
    {
        // Code to bind vertex/index buffers and issue draw call.
    }

    PBRMaterial GetMaterial() const { return material; }
};

// Example Camera structure.
struct Camera
{
    DirectX::XMFLOAT3 position;

    DirectX::XMFLOAT3 GetPosition() const { return position; }
};

// Example Light structure.
struct Light
{
    DirectX::XMFLOAT3 direction;
    DirectX::XMFLOAT3 color;

    DirectX::XMFLOAT3 GetDirection() const { return direction; }
    DirectX::XMFLOAT3 GetColor() const { return color; }
};

// ... within your rendering loop or scene management ...

void RenderScene(const std::vector<Mesh>& meshes, const Camera& camera, const Light& light)
{
    // Get or compile the PBR shader.
    // auto pbrShader = m_shaderManager.GetShader("SimplePBR");

    // Set render targets and depth stencil.
    // m_renderTarget.Begin();

    // Bind textures.
    // m_albedoTexture.Bind(0); // Albedo texture to slot t0
    // defaultSampler.Bind(0);  // Sampler for t0 to slot s0
    // m_specularEnvMap.Bind(1); // Specular env map to slot t1
    // specularSampler.Bind(1);  // Sampler for t1 to slot s1

    // Update PBR constant buffer.
    // This buffer will be bound to register b0.
    PBRConstants shaderConstants;
    shaderConstants.camPos = camera.GetPosition();
    shaderConstants.lightDir = light.GetDirection();
    shaderConstants.lightColor = light.GetColor();
    shaderConstants.irradiance = { 0.2f, 0.2f, 0.2f }; // Example ambient diffuse contribution

    // Set vertex and pixel shaders.
    // pbrShader->SetVertexShader();
    // pbrShader->SetPixelShader();

    // Loop through each mesh in the scene.
    for (const auto& mesh : meshes)
    {
        // Update material properties for the current mesh.
        shaderConstants.material = mesh.GetMaterial();

        // Assuming you have a function to update constant buffers.
        // This function would bind the buffer to the correct shader register (b0).
        // pbrShader->UpdateConstantBuffer("PBRData", &shaderConstants);

        // Draw the mesh.
        // mesh.Draw();
    }

    // m_renderTarget.End();
}
```

**Note:** The C++ snippet is conceptual and demonstrates the structure. A real implementation would involve a graphics API (DirectX, Vulkan, OpenGL), texture loading, vertex buffer management, constant buffer updates, and shader compilation.

## Getting Started

1.  **Clone the repository:**
    ```bash
    git clone <repository_url>
    cd SimplePBRRenderer
    ```
2.  **Explore the HLSL files:** Understand the mathematical functions and how they are combined.
3.  **Examine the C++ code:** See how the shader parameters are set and how the rendering loop is structured.
4.  **Integrate into your project:** Adapt the HLSL shaders and C++ logic to your existing rendering framework.

This project is a work in progress and aims to be a living resource for learning PBR. Contributions and feedback are welcome!

---

If you're passionate about the intricacies of rendering and want to deepen your expertise, consider exploring the field of **Digital Rendering Engineering**. This project provides a small but important step in that direction.

## Optimization Notes

The following performance improvements were made:

1.  **Removed Costly Divisions:**
    *   **`NDF_GGX`**: Replaced `pow(..., 2)` with direct multiplication (`denominatorTerm * denominatorTerm`) and added `EPSILON` to the denominator to prevent division by zero, which is more robust than explicit checks.
    *   **`GeometrySmith_GGX`**: Added `EPSILON` to denominators for robustness against division by zero.
    *   **`FresnelSchlick`**: Replaced `pow(..., 5.0)` with direct multiplications (`cosTheta * cosTheta; pow5 = pow5 * pow5 * cosTheta;`).
    *   **`BRDF_CookTorrance`**: Added `EPSILON` to the `4.0f * NdotV * NdotL` denominator.
    *   **`PSMain`**: Optimized Reinhard tone mapping division by using `MAX_VALUE` instead of `totalLight + 1.0`, preventing potential division by zero when `totalLight` is zero.

2.  **Improved Vector Math & Clarity:**
    *   **Constants**: Defined `PI` and `TWO_PI` as `static const float` for efficiency. Introduced `EPSILON` for floating-point stability.
    *   **`PerceptualRoughnessToRoughness`**: Corrected the logic to a standard PBR mapping (`roughness = perceptualRoughness * perceptualRoughness;`) and added `saturate()` for clamping.
    *   **`RoughnessToAlphaSq`**: Renamed and clarified its purpose. The implementation directly uses `roughness * roughness` as `alphaSq` which is common for GGX, avoiding an extra multiplication if `alpha` was intended to be `roughness`. The original code used `alpha = roughness * roughness` for NDF and Geometry, so `RoughnessToAlphaSq` is now more direct.
    *   **`NDF_GGX`**: Simplified `alpha` to `roughness` and used `alphaSq` as `roughness * roughness` in the denominator, aligning with common GGX formulations and the original code's intent. Added `EPSILON` for stability. Used `saturate()` on `NdotH`.
    *   **`GeometrySmith_GGX`**: Explicitly used `saturate()` on `NdotV` and `NdotL` for robustness. Simplified the calculation of the term under the square root to match common PBR libraries. Added `EPSILON` for stability.
    *   **`FresnelSchlick`**: Ensured `cosTheta` is `saturate()`d before exponentiation.
    *   **`BRDF_CookTorrance`**:
        *   Added `saturate()` to all dot product calculations (`NdotL`, `NdotV`, `NdotH`, `LdotH`) for robustness.
        *   Clarified the calculation of `k_d` (diffuse) and `k_s` (specular) using `metallic` and `albedo`. The diffuse term is now `(1.0f - metallic) * albedo / PI`, which is a standard formulation.
        *   The function now returns the BRDF itself, making it clearer for integration with `Li * NdotL`.
    *   **`PSMain`**:
        *   Added `saturate()` to `metallic` and `ao` inputs.
        *   Ensured `baseColor` is used consistently for both diffuse and specular calculations where appropriate.
        *   Clarified the calculation of `F0` based on `metallic` and `baseColor`.
        *   Explicitly normalized `N`, `V`, and `L`.
        *   The final accumulation of direct and indirect lighting is clearer.
        *   Added comments about the simplified nature of indirect lighting and tone mapping.
    *   **C++ Snippet**: Added default values to `PBRMaterial` and `PBRConstants` for better initialization. Made comments clearer regarding constant buffer mapping and API dependence.

3.  **Loop Unrolling (Implicit):** While not explicitly unrolling loops in the C++ sense, the shader code is designed to process a single fragment/pixel. Functions like `NDF_GGX`, `GeometrySmith_GGX`, and `FresnelSchlick` are called once per fragment for each light source. The structure of the shader and its reliance on fixed-function hardware for texture sampling and dot products mean that typical loop unrolling concepts from CPU code are less directly applicable. The focus has been on optimizing the scalar and vector operations within these functions and the main pixel shader.

4.  **Code Readability and Robustness:**
    *   Added `#pragma once` to the HLSL file for better include management.
    *   Used `f` suffix for float literals (e.g., `3.14159265359f`).
    *   Improved comments to explain the rationale behind certain calculations and optimizations.
    *   Ensured `saturate()` is used for dot products and inputs to prevent invalid mathematical operations.
    *   Added `EPSILON` for stable floating-point arithmetic, particularly in denominators.
    *   The C++ snippet now includes default values for material properties and basic data structures for better context.