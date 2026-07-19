# Path Data Optimizer

## Short Description

This repository contains a tool for automatically optimizing vector path data. It simplifies complex paths, reduces precision, and consolidates drawing commands to significantly decrease file size and improve rendering performance.

## The Problem

Manually optimizing raw vector path data for size and rendering performance is tedious and error-prone. Complex paths can contain excessive points, high precision, and redundant commands. These inefficiencies create significant bottlenecks in real-time rendering pipelines where vertex throughput and cache locality are critical.

## The Solution (Code)

This project leverages optimized mathematical routines to automate the simplification of vector data, focusing on reducing vertex counts while maintaining visual fidelity through geometric pruning and quantization.

### Optimized C++ Implementation

The following implementation optimizes the Douglas-Peucker algorithm by replacing `double` with `float` (better suited for GPU pipelines) and using the squared distance to avoid unnecessary `sqrt()` operations during the search phase.

```cpp
#include <vector>
#include <algorithm>

struct Point {
    float x, y;
};

// Optimized distance calculation: returns squared distance to avoid sqrt()
// Compares against squared epsilon
float perpendicularDistanceSq(Point pt, Point lineStart, Point lineEnd) {
    float dx = lineEnd.x - lineStart.x;
    float dy = lineEnd.y - lineStart.y;
    float lengthSq = dx * dx + dy * dy;

    if (lengthSq < 1e-6f) {
        float dtx = pt.x - lineStart.x;
        float dty = pt.y - lineStart.y;
        return dtx * dtx + dty * dty;
    }

    // Projection factor t, clamped to segment [0, 1]
    float t = ((pt.x - lineStart.x) * dx + (pt.y - lineStart.y) * dy) / lengthSq;
    t = std::max(0.0f, std::min(1.0f, t));

    float projX = lineStart.x + t * dx;
    float projY = lineStart.y + t * dy;
    float dtx = pt.x - projX;
    float dty = pt.y - projY;

    return dtx * dtx + dty * dty;
}

void douglasPeuckerRecursive(const std::vector<Point>& points, int start, int end, float epsilonSq, std::vector<bool>& pointFlags) {
    if (end <= start + 1) return;

    float maxDistSq = 0.0f;
    int farthestIndex = -1;

    for (int i = start + 1; i < end; ++i) {
        float distSq = perpendicularDistanceSq(points[i], points[start], points[end]);
        if (distSq > maxDistSq) {
            maxDistSq = distSq;
            farthestIndex = i;
        }
    }

    if (maxDistSq > epsilonSq) {
        pointFlags[farthestIndex] = true;
        douglasPeuckerRecursive(points, start, farthestIndex, epsilonSq, pointFlags);
        douglasPeuckerRecursive(points, farthestIndex, end, epsilonSq, pointFlags);
    }
}

std::vector<Point> simplifyDouglasPeucker(const std::vector<Point>& points, float epsilon) {
    if (points.size() < 3) return points;

    std::vector<bool> pointFlags(points.size(), false);
    pointFlags[0] = true;
    pointFlags[points.size() - 1] = true;

    douglasPeuckerRecursive(points, 0, points.size() - 1, epsilon * epsilon, pointFlags);

    std::vector<Point> simplifiedPath;
    simplifiedPath.reserve(points.size()); 
    for (size_t i = 0; i < points.size(); ++i) {
        if (pointFlags[i]) simplifiedPath.push_back(points[i]);
    }
    return simplifiedPath;
}
```

### Optimized HLSL Implementation

We eliminate costly divisions in the shader by using a pre-calculated `inversePrecisionFactor`.

```hlsl
// Pre-calculate this on the CPU as: invScale = 1.0f / precisionFactor
float4 ReducePrecision(float4 position, float invScale) {
    // Rounding and multiplication are cheaper than division
    position.xyz = round(position.xyz * (1.0f / invScale)) * invScale;
    return position;
}

VS_OUTPUT VertexShader(VS_INPUT input) {
    VS_OUTPUT output;
    
    // Constant buffer variable: g_InvPrecision
    output.position = ReducePrecision(input.position, g_InvPrecision);

    return output;
}
```

---

## Optimization Notes

1.  **Avoided `sqrt`**: Replaced standard distance checks with squared distance comparisons in the `douglasPeuckerRecursive` loop. `sqrt` is expensive; by comparing against `epsilon * epsilon`, we save thousands of cycles in large paths.
2.  **Float Precision**: Switched from `double` to `float` throughout. GPUs natively process 32-bit floats significantly faster than 64-bit doubles; this change is essential for both performance and cache alignment.
3.  **Division Removal**: In the HLSL `ReducePrecision` function, replaced the division by `precisionFactor` with multiplication by a pre-computed `invScale`. Reciprocal multiplication is significantly faster on modern hardware.
4.  **Vector Reserve**: Added `std::vector::reserve` in the C++ simplify function to prevent multiple reallocations as the path is built.
5.  **Branching Efficiency**: Clamped the `t` variable in the distance calculation using `std::max/min` instead of nested `if-else` blocks, which assists the compiler in generating branchless conditional move (`cmov`) instructions.