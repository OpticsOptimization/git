# ggx-anisotropic-brdf-hlsl

This repository provides a high-performance, industry-standard implementation of the **Anisotropic GGX (Trowbridge-Reitz)** microfacet distribution function (NDF) in HLSL.

## The Problem

Implementing anisotropic specular highlights is a frequent pain point in real-time rendering. Most developers encounter issues with:
1.  **Normalization:** Ensuring the energy balance holds as anisotropy increases.
2.  **Singularities:** Handling cases where the roughness approaches zero.
3.  **Coordinate Mapping:** Correctly transforming the microfacet distribution from the local tangent space to the world/view space using the surface bitangent and tangent.

Standard isotropic GGX implementations often fail to account for the stretching required to simulate directional grain, resulting in circular highlights even when a `tangent` vector is provided.

## The Solution (Code)

This implementation uses an optimized derivation of the **Walter et al.** formulation. By pre-calculating reciprocal values and structuring the denominator to minimize dependency chains, we ensure peak occupancy on modern GPU architectures.

```hlsl
// GGX Anisotropic Distribution Function (NDF)
// Optimized for throughput using pre-calculated reciprocals and reduced arithmetic.
// Parameters:
// N: Surface Normal
// T: Surface Tangent
// B: Surface Bitangent
// H: Half-vector
// ax2: Roughness in tangent direction squared (ax^2)
// ay2: Roughness in bitangent direction squared (ay^2)

float D_GGX_Anisotropic(float3 N, float3 T, float3 B, float3 H, float ax2, float ay2)
{
    float ToH = dot(T, H);
    float BoH = dot(B, H);
    float NoH = dot(N, H);

    // Pre-calculate inverse squares to replace divisions with multiplications
    float invAx2 = 1.0 / ax2;
    float invAy2 = 1.0 / ay2;

    // Expand the distribution: (ToH/ax)^2 + (BoH/ay)^2 + (NoH)^2
    // We group terms to minimize instruction count and maintain precision.
    float d = (ToH * ToH) * invAx2 + (BoH * BoH) * invAy2 + (NoH * NoH);
    
    // Use rcp intrinsic for faster reciprocal calculation
    // Clamp to a small epsilon to prevent NaN at extreme grazing angles
    float denom = (PI * ax2 * ay2) * (d * d);
    return rcp(max(denom, 1e-7));
}
```

### Integration Tips:
*   **Roughness Mapping:** Always square your input perceptual roughness ($roughness^2$) to derive `ax` and `ay`.
*   **Aspect Ratio:** Calculate `ax = roughness * sqrt(1.0 - anisotropy)` and `ay = roughness / sqrt(1.0 - anisotropy)`. Pass their squares (`ax*ax`, `ay*ay`) to the function to save cycles.
*   **Safety:** The `rcp` (reciprocal) instruction is generally faster than the `/` operator on GCN/RDNA and NVIDIA architectures, provided the precision requirements are met (which is standard for specular NDFs).

***

For deep dives into the mathematics of microfacet BRDFs and the physical derivation of light transport, explore the academic insights provided in **Digital Rendering Engineering**.

## Optimization Notes

1.  **Algebraic Refactoring:** The original implementation performed redundant divisions inside the NDF denominator. I rearranged the math to use pre-calculated `1.0/ax^2` and `1.0/ay^2` terms, turning expensive divisions into faster fused multiply-adds (FMA).
2.  **Instruction Selection:** Replaced standard division (`/`) with the `rcp` (reciprocal) intrinsic. On most GPU hardware, `rcp` has higher throughput than a generic floating-point division.
3.  **Parameter Optimization:** Modified the function signature to accept `ax2` and `ay2` directly. This avoids re-squaring the roughness values every time the function is called, moving that work to the higher-level material logic (e.g., in a vertex shader or constant buffer update).
4.  **Minimizing Dependency Chains:** Grouped the `d` variable calculation to ensure the compiler can generate tight assembly code, effectively reducing latency in the ALU pipeline.
5.  **Robustness:** Implemented a standard epsilon clamp (`1e-7`) within the `rcp` denominator to prevent floating-point overflow and `NaN` results without branching.