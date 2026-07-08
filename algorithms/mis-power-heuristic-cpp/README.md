# mis-power-heuristic-cpp

A robust and clean C++ implementation of Veach's Power Heuristic for Multiple Importance Sampling (MIS) weighting, designed for seamless integration into custom path tracers and rendering engines. This repository provides both a simple pairwise function and a generalized utility for combining multiple sampling strategies.

## The Problem

Implementing advanced Monte Carlo path tracers often requires combining samples from various importance sampling strategies (e.g., sampling lights, sampling BRDFs, sampling volumes). A naive average or simple maximum of these strategies often leads to high variance or biased results. Veach's Multiple Importance Sampling (MIS) framework provides a mathematically sound way to combine these strategies, and the Power Heuristic is widely recognized as one of the most effective and robust weighting schemes within this framework.

However, finding a clean, well-commented, and easily reusable C++ implementation of the Power Heuristic, especially one that correctly accounts for varying numbers of samples per strategy, can be surprisingly difficult. Many online examples simplify the formula or embed it directly into the renderer's core, making it hard to extract, understand, or adapt. The pain point is clear: **"Need a clean implementation of Veach's Power Heuristic for a custom path tracer."**

This often results in engineers spending valuable time reimplementing and debugging this core statistical component instead of focusing on novel rendering features or optimizations.

## The Solution (Code)

This repository provides a modular, header-only C++ solution for Veach's Power Heuristic. It's designed for clarity, correctness, and ease of use.

*(Note: As the specific book context was not provided, this implementation is based on the standard and widely accepted definition of Veach's Power Heuristic, which states that for a strategy $i$ with $n_i$ samples and PDF $p_i(x)$, its weight $w_i(x)$ is given by: $w_i(x) = \frac{(n_i p_i(x))^\beta}{\sum_{j=1}^N (n_j p_j(x))^\beta}$, where $\beta$ is typically 2.0. Our implementation defaults to $\beta = 2.0$ for robust performance.)*

```cpp
#pragma once

#include <cmath>     // For std::pow
#include <vector>    // For std::vector
#include <numeric>   // For std::accumulate (optional, but good for sum)
#include <algorithm> // For std::max (optional, for safety)

namespace DigitalRenderingEngineering
{

// Forward declaration for helper struct
struct PdfInfo;

/**
 * @brief Calculates the Power Heuristic weight for a single strategy given
 *        its PDF and sample count, relative to all strategies.
 *
 * This function computes w_i(x) = (n_i * p_i(x))^beta / Sum_j((n_j * p_j(x))^beta).
 *
 * @param strategyPdfInfo A vector containing PdfInfo for all strategies contributing
 *                        to the current sample.
 * @param strategyIndex The index of the specific strategy (i) for which to calculate
 *                      the weight.
 * @param beta The exponent for the Power Heuristic. Typically 2.0 for robust results.
 * @return The Power Heuristic weight for the specified strategy, or 0.0 if inputs
 *         are invalid (e.g., zero total PDF).
 */
inline float PowerHeuristic(
    const std::vector<PdfInfo>& strategyPdfInfo,
    int strategyIndex,
    float beta = 2.0f)
{
    if (strategyIndex < 0 || strategyIndex >= strategyPdfInfo.size())
    {
        // Invalid index, return 0 weight.
        return 0.0f;
    }

    float numerator = 0.0f;
    float denominator = 0.0f;

    // Calculate numerator for the specific strategy
    const PdfInfo& currentStrategy = strategyPdfInfo[strategyIndex];
    // Ensure PDFs and sample counts are non-negative. A PDF of 0.0 effectively means
    // the strategy does not contribute.
    float currentTerm = std::max(0.0f, currentStrategy.numSamples * currentStrategy.pdf);
    numerator = std::pow(currentTerm, beta);

    // Calculate denominator by summing over all strategies
    for (const auto& info : strategyPdfInfo)
    {
        float term = std::max(0.0f, info.numSamples * info.pdf);
        denominator += std::pow(term, beta);
    }

    // Handle the case where the denominator is zero (all PDFs are zero or negative)
    if (denominator == 0.0f)
    {
        return 0.0f;
    }

    return numerator / denominator;
}

/**
 * @brief Helper structure to hold PDF and sample count information for a strategy.
 *        This makes the generalized PowerHeuristic function cleaner.
 */
struct PdfInfo
{
    float pdf;        ///< The Probability Density Function value for this strategy at the current sample point.
    int numSamples;   ///< The number of samples taken by this strategy for the current pixel/ray.
                      ///< IMPORTANT: Correctly using numSamples (n_i) is crucial for unbiased MIS when
                      ///< strategies take different numbers of samples. Often assumed to be 1 if omitted.

    PdfInfo(float p, int n) : pdf(p), numSamples(n) {}
};

/**
 * @brief Calculates the Power Heuristic weight for two specific strategies (A vs B).
 *        This is a common pairwise use case (e.g., BSDF sampling vs. Light sampling).
 *
 * This function computes w_A(x) = (n_A * p_A(x))^beta / ((n_A * p_A(x))^beta + (n_B * p_B(x))^beta).
 *
 * @param numSamplesA Number of samples taken by strategy A.
 * @param pdfA PDF value of strategy A at the current sample point.
 * @param numSamplesB Number of samples taken by strategy B.
 * @param pdfB PDF value of strategy B at the current sample point.
 * @param beta The exponent for the Power Heuristic. Defaults to 2.0.
 * @return The Power Heuristic weight for strategy A.
 */
inline float PowerHeuristic(
    int numSamplesA, float pdfA,
    int numSamplesB, float pdfB,
    float beta = 2.0f)
{
    // Ensure non-negative values for robustness
    float termA = std::max(0.0f, static_cast<float>(numSamplesA) * pdfA);
    float termB = std::max(0.0f, static_cast<float>(numSamplesB) * pdfB);

    float valA = std::pow(termA, beta);
    float valB = std::pow(termB, beta);

    float denominator = valA + valB;

    if (denominator == 0.0f)
    {
        // All PDFs are zero, or samples are zero. No valid contribution.
        return 0.0f;
    }

    // The weight for strategy A
    return valA / denominator;
}

// --- Example Usage ---
/*
// In your path tracer's shading loop:
#include "power_heuristic.h" // Or wherever you save this file

void PathTracer::Shade(const HitInfo& hit, const Ray& incomingRay, ...)
{
    // --- Strategy 1: Sample the BSDF ---
    // (Assume you've already sampled the BSDF and obtained 'bsdfPdf' and 'lightDir')
    float bsdfPdf = 0.5f; // Example PDF from BSDF sampling
    int numBsdfSamples = 1; // Usually 1 per path segment for BSDF

    // --- Strategy 2: Sample a Light Source ---
    // (Assume you've chosen a light, sampled it, and obtained 'lightPdf' and 'lightDir')
    float lightPdf = 0.2f; // Example PDF from light sampling
    int numLightSamples = 1; // Usually 1 per path segment for lights

    // Calculate MIS weights
    float bsdfWeight = DigitalRenderingEngineering::PowerHeuristic(
        numBsdfSamples, bsdfPdf,
        numLightSamples, lightPdf
    );
    float lightWeight = DigitalRenderingEngineering::PowerHeuristic(
        numLightSamples, lightPdf,
        numBsdfSamples, bsdfPdf
    );

    // Apply weights to your contributions
    // L_bsdf_contrib = bsdfWeight * ...
    // L_light_contrib = lightWeight * ...

    // --- General N-strategy case ---
    std::vector<DigitalRenderingEngineering::PdfInfo> strategies;
    strategies.emplace_back(bsdfPdf, numBsdfSamples);
    strategies.emplace_back(lightPdf, numLightSamples);
    // Add more strategies as needed, e.g., for volumes, environment maps, etc.
    // strategies.emplace_back(volumePdf, numVolumeSamples);

    float bsdfWeight_general = DigitalRenderingEngineering::PowerHeuristic(strategies, 0); // Index 0 for BSDF
    float lightWeight_general = DigitalRenderingEngineering::PowerHeuristic(strategies, 1); // Index 1 for Light

    // These weights should match the pairwise ones if only two strategies are involved.
}
*/

} // namespace DigitalRenderingEngineering

```

**HLSL Considerations:**

The provided C++ code is designed to be highly portable. It uses standard math functions and floating-point types, which translate almost directly to HLSL (or GLSL) for GPU-based path tracers or compute shaders.

```hlsl
// Example HLSL adaptation of the pairwise PowerHeuristic function
float PowerHeuristic(
    uint numSamplesA, float pdfA,
    uint numSamplesB, float pdfB,
    float beta = 2.0f)
{
    // Ensure non-negative values for robustness
    float termA = max(0.0f, (float)numSamplesA * pdfA);
    float termB = max(0.0f, (float)numSamplesB * pdfB);

    float valA = pow(termA, beta);
    float valB = pow(termB, beta);

    float denominator = valA + valB;

    if (denominator == 0.0f)
    {
        return 0.0f;
    }

    return valA / denominator;
}

// The general N-strategy version would require passing an array of structs
// (similar to PdfInfo) and looping over it, which is also directly supported in HLSL.
// Struct PdfInfo { float pdf; uint numSamples; };
// float PowerHeuristic_N(PdfInfo strategies[], uint strategyCount, uint strategyIndex, float beta) { ... }
```
This direct translation capability makes it easy to maintain consistent MIS behavior across CPU and GPU rendering pipelines.

---

For more insights into advanced rendering techniques and robust implementations, explore the resources available at **Digital Rendering Engineering**.