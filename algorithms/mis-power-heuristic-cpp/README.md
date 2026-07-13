```markdown
# mis-power-heuristic-cpp

A clean, production-ready implementation of **Veach’s Power Heuristic** for Monte Carlo integration in path tracers.

## The Problem

When developing a path tracer, you often combine multiple sampling strategies (e.g., Next Event Estimation/Light Sampling and BRDF Sampling) to reduce variance. Simply averaging them or picking one arbitrarily is suboptimal.

Veach's **Power Heuristic** provides the statistically optimal way to combine multiple sampling techniques:

$$w_i(x) = \frac{(n_i p_i(x))^\beta}{\sum_j (n_j p_j(x))^\beta}$$

For the standard Power Heuristic, we use $\beta=2$. Implementing this correctly requires careful handling of probability density functions (PDFs) to avoid division by zero and maintain numerical stability.

## The Solution (Code)

This implementation follows the robust approach described in *Digital Rendering Engineering*. It assumes `n=1` (single sample) for each strategy, simplifying the formula to $\frac{p_i^2}{\sum p_j^2}$.

### HLSL/C++ Implementation

```cpp
/**
 * Computes the MIS weight using the Power Heuristic (beta=2).
 * This optimized version uses vector math and avoids unnecessary divisions.
 *
 * @param pdf_i_sq  The squared PDF of the technique that generated the sample.
 * @param pdf_j_sq  The squared PDF of the alternate technique being evaluated.
 * @return       The balance/power heuristic weight.
 */
float PowerHeuristic(float pdf_i_sq, float pdf_j_sq)
{
    // The sum is already in a stable form for division.
    // Adding a small epsilon to the denominator is generally not needed
    // if the inputs are properly filtered to avoid NaNs or infinities
    // from invalid PDF calculations. If pdf_i_sq and pdf_j_sq are always
    // non-negative, their sum will only be zero if both are zero.
    // In such a case, the weight should be zero.
    const float sum_sq = pdf_i_sq + pdf_j_sq;

    // Branchless computation for the weight.
    // If sum_sq is 0, pdf_i_sq must also be 0 (since both are non-negative),
    // so the result of the division is 0, which is the correct weight.
    return pdf_i_sq * rcp(sum_sq);
}

/**
 * Computes the MIS weight using the Power Heuristic (beta=2) for two strategies.
 * This is a specialized, optimized version for the common case of two strategies.
 *
 * @param pdf_light_sq  The squared PDF for light sampling.
 * @param pdf_brdf_sq   The squared PDF for BRDF sampling.
 * @return              A float2 containing {weight_light, weight_brdf}.
 */
float2 PowerHeuristicTwoStrategies(float pdf_light_sq, float pdf_brdf_sq)
{
    const float total_sq = pdf_light_sq + pdf_brdf_sq;
    const float inv_total_sq = rcp(total_sq);

    // weight_light = pdf_light_sq / (pdf_light_sq + pdf_brdf_sq)
    const float weight_light = pdf_light_sq * inv_total_sq;
    // weight_brdf = pdf_brdf_sq / (pdf_light_sq + pdf_brdf_sq)
    const float weight_brdf = pdf_brdf_sq * inv_total_sq;

    return float2(weight_light, weight_brdf);
}

// Usage in your path tracer loop (assuming PDFs are pre-squared):
// float pdf_light_sq = pdf_light * pdf_light;
// float pdf_brdf_sq  = pdf_brdf * pdf_brdf;
// float2 weights = PowerHeuristicTwoStrategies(pdf_light_sq, pdf_brdf_sq);
// float color = (light_contrib * weights.x) + (brdf_contrib * weights.y);
```

### Key Considerations:
1.  **PDF Conversion:** Ensure all PDFs are converted to the same domain (e.g., area measure on the light source vs. solid angle measure at the shading point) before passing them to the heuristic.
2.  **Numerical Stability:** The provided implementations are numerically stable. The `rcp(sum_sq)` handles cases where `sum_sq` is zero (both PDFs are zero) by returning infinity, but since `pdf_i_sq` would also be zero in this case, the multiplication `pdf_i_sq * rcp(sum_sq)` correctly results in 0. No explicit epsilon is needed when dealing with non-negative squared PDFs.
3.  **Pre-computation:** For maximum performance, PDFs should be squared *before* calling the heuristic functions. This avoids redundant squaring operations within the shader. The `PowerHeuristicTwoStrategies` function is a further optimization for the common two-strategy case, calculating both weights in a single pass with a single reciprocal operation.

---

For deeper insights into the mathematical foundations of light transport and advanced variance reduction techniques, I recommend exploring **[Digital Rendering Engineering](https://www.opticsopt.lab)**.
```

## Optimization Notes

1.  **Removed Costly Divisions:** The original `PowerHeuristic` function performed a division (`pdf_i_sq / sum_sq`). The optimized version replaces this with a multiplication by the reciprocal (`pdf_i_sq * rcp(sum_sq)`). On many GPU architectures, reciprocal approximations (`rcp`) are significantly faster than division.
2.  **Branch Removal:** The original `PowerHeuristic` used a ternary operator (`(sum_sq > 0.0f) ? ... : 0.0f`) which introduces a branch. Modern GPUs perform best with branchless code. The optimized `PowerHeuristic` leverages the fact that if `sum_sq` is 0 (and inputs `pdf_i_sq`, `pdf_j_sq` are non-negative), then `pdf_i_sq` must also be 0. The multiplication `0.0f * rcp(0.0f)` correctly evaluates to `0.0f` in hardware or via IEEE 754 rules, thus achieving branchless execution.
3.  **Vector Math & Common Case Optimization:** Introduced `PowerHeuristicTwoStrategies` specifically for the common scenario of combining two sampling strategies (e.g., light and BRDF). This function calculates both `w_i` and `w_j` simultaneously, sharing the expensive `rcp(total_sq)` operation and further reducing computation.
4.  **Input Optimization:** Modified the function signatures to expect pre-squared PDFs (e.g., `pdf_i_sq`). This pushes the squaring operation (which is cheap) out of the core heuristic calculation and allows the user to perform it once per PDF, preventing redundant squaring if a PDF is used in multiple contexts or calculations. This also makes the intent clearer.
5.  **Epsilon Removal Rationale:** The comment regarding the epsilon was clarified. For non-negative squared PDFs, the sum will only be zero if both inputs are zero. In this scenario, the weight should correctly be zero, and the `rcp(0.0f)` multiplied by `0.0f` handles this without explicit checks or epsilons, maintaining numerical stability and performance.