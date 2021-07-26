### [Integer Quantization for Deep Learning Inference: Principles and Empirical Evaluation](https://arxiv.org/abs/2004.09602)

Authors: Hao Wu, Patrick Judd, Xiaojie Zhang, Mikhail Isaev, Paulius Micikevicius (NVIDIA)

#### Summary
 - this paper review the mathematical aspects of quantization parameters and evaluate their choices on a wide range of neural network models for different application domains, including vision, speech, and language
 - focus on quantization techniques that are amenable to acceleration by processors with high-throughput integer math pipelines
 - a workflow for 8-bit quantization that is able to maintain accuracy within 1% of the floating-point baseline on all networks studied

#### Performance benefits for lower-precision formats
 - many processors provide **higher throughput** math pipelines the low-bit formats, which can speed up **math-intensive operations**, such as **convolutions** and **matrix multiplications**
 - smaller word sizes **reduce memory bandwidth pressure**, improving performance for bandwidth-limited computations
 - smaller word sizes lead to **lower memory size requirements**, which can improve cache utilization as well as other aspects of memory-system operation

#### Quantization fundamentals:
 - Two steps of unifrom quantization
   - First, choose the range of the real numbers to be quantized, clamping the values outside this range. 
   - Second, map the real values to integers representable by the bit-width of the quantized representation (round each mapped real value to the closest integer value).
 - Two fundamental operations:
   - Quantize: convert a real number to a quantized integer representation (e.g. from fp32 to int8)
   - Dequantize:convert a number from quantized integer representation to a real number (e.g. from int32 to fp16)
 - Affine quantization
 - Scale quantization

#### Tensor quantization granularity
 - At the coarsest, per-tensor granularity, the same quantization parameters are shared by all elements in the tensor. The finest granularity would have individual quantization parameters per element. Intermediate granularities reuse parameters over various dimensions of the tensor - per row or per column for 2D matrices, per channel for 3D (image-like) tensors, etc.
 - Integer matrix multiplication is possible as long as the quantization granularity is per-row or per-tensor for activations and per-column or per-tensor for weights
 - For maximum performance, activations should use per-tensor quantization granularity.
 - While both affine and scale quantization enable the use of integer arithmetic, affine quantization leads to more computationally expensive inference.

#### Calibration
 - Calibration is the process of choosing α and β for model weights and activations
 - Max: Use the maximum absolute value seen during calibration.
 - Entropy: Use **KL divergence** to minimize information loss between the original floating-point values and values that could be represented by the quantized format. This is the default method used by **TensorRT**.
 - Percentile: Set the range to a percentile of the distribution of absolute values seen during calibration. For example, 99% calibration would clip 1% of the largest magnitude values.

#### Post Training Quantization
 - Quantization parameters are calibrated offline by processing the trained model weights and activations generated by running inference on a sample dataset, **no further training is involved**.
 - An operation is quantized by quantizing all of its inputs (e.g. weights and activations). The output of a quantizated operation is not quantized to int8 because the operation that follows it may require higher precision, e.g. nonlinear operations. Furthermore, **consecutive operations can be executed with a fused implementation**, avoiding memory reads and writes for the intermediate values. Therefore we leave quantization of the output activations to the input of the next operation.

#### Techniques to recover accuracy
 - Partial quantization
   - quantization of one layer affects the inputs of others, finding the optimal set of layers to quantize can require evaluating an exponential number of configurations
   - one-at-a-time sensitivity analysis
 - Quantization-aware training
   - insert quantization operations in to the neural network before training or fine-tuning, to allow the network to adapt to the quantized weights and activations
   - One challenge to training in floating-point with quantization is that **the quantization operation’s derivative is undefined at the step boundaries and zero everywhere else**.
     - Straight-through Estimator (STE)
     - STE approximates the derivative of the fake quantization function to be 1 for inputs in the representable range [β, α]