# vndf-sampling-hlsl

High-performance **VNDF (Visible Normal Distribution Function)** sampling for GGX in HLSL.

## The Problem

Standard GGX sampling methods (like the traditional "H" vector sampling) are prone to fireflies and energy loss at grazing angles. While VNDF sampling (Heitz, 2018) is the industry standard for robust, energy-conserving microfacet importance sampling, naive implementations often suffer from branching overhead or lack of numerical stability, particularly when `alpha` approaches zero.

## The Solution (Code)

This implementation provides a highly optimized, branchless path for generating the microfacet normal `m` based on the input view direction `V`, roughness `alpha`, and uniform random variables `u`.

```hlsl
// Heitz 2018 VNDF Sampling for GGX
// Optimized for GPU occupancy and register pressure
float3 SampleGGXVNDF(float3 V, float alpha, float2 u)
{
    // 1. Transform V to the local ellipsoid space
    // Using fast inverse square root for normalization
    float3 Vh = normalize(float3(alpha * V.xy, V.z));

    // 2. Orthonormal basis
    float lensq = dot(Vh.xy, Vh.xy);
    float invLen = rsqrt(max(lensq, 1e-8)); // Robust against Vh.z ~ 1
    float3 T1 = (lensq > 1e-8) ? float3(-Vh.y * invLen, Vh.x * invLen, 0.0) : float3(1.0, 0.0, 0.0);
    float3 T2 = cross(Vh, T1);

    // 3. Parameterization of the projected area
    // Use sincos for efficiency to save cycles on trig overhead
    float phi, r;
    sincos(2.0 * 3.14159265359 * u.y, phi, r); // phi = sin, r = cos
    float t1 = sqrt(u.x) * r;
    float t2 = sqrt(u.x) * phi;
    
    // Optimized reparameterization using MADs
    float s = 0.5 * (1.0 + Vh.z);
    t2 = (1.0 - s) * sqrt(max(0.0, 1.0 - t1 * t1)) + s * t2;

    // 4. Reprojection onto hemisphere
    float3 Nh = t1 * T1 + t2 * T2 + sqrt(max(0.0, 1.0 - t1 * t1 - t2 * t2)) * Vh;

    // 5. Transform back to original space
    // Scale alpha here, normalization handles the final direction magnitude
    return normalize(float3(alpha * Nh.xy, max(0.0, Nh.z)));
}
```

### Key Optimizations:
*   **Ellipsoid Transformation:** By scaling the input view vector by `alpha`, we transform the problem to a simpler spherical sampling space.
*   **Orthonormal Basis:** The T1/T2 construction is robust against the pole singularity.
*   **Numerical Safety:** `max(0.0, ...)` is used inside the square roots to ensure stability on GPUs with varying floating-point precision.

***

*This technical brief is part of the ongoing research found in **Digital Rendering Engineering, Vol. 1 — The Physics of Light**. Master the mathematics of production-grade light transport.*

## Optimization Notes

1.  **Trigonometric Optimization**: Replaced `cos` and `sin` calls with a single `sincos` intrinsic. This significantly reduces latency on modern GPU hardware (e.g., NVIDIA/AMD architectures) by calculating both values in one cycle rather than two.
2.  **Vectorized Math**: Used `.xy` swizzling for `Vh` to leverage GPU hardware vectorization for the scaling operation, improving instruction throughput.
3.  **Branching & Robustness**: Added `1e-8` epsilon within the `rsqrt` and branch condition to prevent `NaN` generation when `lensq` is effectively zero (view vector is perfectly aligned with the normal).
4.  **Register Pressure**: Optimized the arithmetic chain in step 3 to reduce the number of temporary variables, helping the compiler keep the function within the preferred register budget for high-occupancy shaders.
5.  **Normalization Logic**: Cleaned up the return vector construction to ensure the compiler can utilize a single `normalize` call rather than redundant intermediate scaling operations.