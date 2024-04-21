Machine Learning
================

## Background

- <https://en.wikipedia.org/wiki/History_of_artificial_intelligence>
  - Birth of artificial intelligence (1941-56)
    - Alan Turing
    - artificial neural network (ANN)
    - symbolic reasoning
    - dartmouth workshop
  - Early successes (1956-1974)
    - ANN up to 4 layers
  - First AI winter (1974–1980)
    - limited compute power
  - Boom (1980–1987)
    - expert systems
    - ANN with backpropagation
  - Bust: second AI winter (1987–1993)
  - AI (1993–2011)
    - thanks to increased compute power and focus on specific domains
    - deep blue
    - probabilistic reasoning
  - Deep learning, big data (2011–2020)
  - Large language models, AI era (2020–present)
- <https://en.wikipedia.org/wiki/Timeline_of_machine_learning>
  - 1950s: machine learning pioneered
  - 1960s: probabilistic inference with bayesian methods 
  - 1970s: ai winter
  - 1980s: backpropagation, reinforcement learning
  - 1990s: Support-vector machines (SVMs), recurrent neural networks (RNNs)
  - 2000s: unsupervised machine learning
  - 2010s: deep learning
  - 2020s: large language models (LLMs) and generative AI
- <https://en.wikipedia.org/wiki/Deep_learning>
  - artificial neural network with multiple layers
  - deep neural networks
    - feedforward neural networks (FNNs)
    - recurrent neural networks (RNNs)
    - convolutional neural networks (CNNs)
    - transformers
  - training
    - supervised learning
    - unsupervised learning
    - reinforcement learning
- <https://en.wikipedia.org/wiki/Transformer_(deep_learning_architecture)>
  - primary components
    - tokenizer converts input into tokens
      - e.g., a sentence is broken down into an array of words or subwords
      - each word or subword is a token
    - an embedding layer converts tokens to vectors
      - token embedding: each token is converted to a vector individually
      - positional embedding: each token is converted to a vector based on its
        relative position in the token array
      - for each token, the two vectors are summed together to form the final
        vector
      - if there are M tokens and each token is converted to a N-dim vector,
        we have a MxN matrix
    - tranformer layers transform vectors repeatedly
      - the transformer layers can either be an encoder or a decoder
    - an optional un-embedding layer converts vectors to tokens
  - each encoder layer has two major components
    - a self-attention mechanism, which weights the input vectors
    - a feed-forward neural network, which process input vector individually
    - the resulting vectos are passed down to the next encoder layer
    - I guess this is the process of understanding the input, or "extracting
      the features"
      - the extracted features will be fed into the decoders
  - each decoder layer has three major components
    - a self-attention mechanism, just like an encoder
    - an attention mechanism, which takes inputs from encoders
    - a feed-forward neural network, just like an encoder
    - I guess this is the process of generating the output
      - the output tokens are generated one-by-one
      - the first output token is generated solely from the extracted features
      - the following output tokens are geneated based on the extracted
        features as well as prior output tokens
- <https://en.wikipedia.org/wiki/Large_language_model>
  - transformer-based
  - Google BERT, LaMDA, PaLM, and Gemini
  - OpenAI GPT-{1,2,3,3.5,4}
  - Meta Llama

## TensorFlow Lite

- build with cmake
  - `git clone https://github.com/tensorflow/tensorflow.git`
  - `cmake -S tensorflow/lite -B out -G Ninja -DTFLITE_ENABLE_GPU=ON`
  - `ninja -C out tensorflow-lite benchmark_model`
- `benchmark_model`
  - it seems after initial setup and inference, each inference does
    - `clEnqueueWriteBuffer`
    - `clSetKernelArg` and `clEnqueueNDRangeKernel` for
      - `bhwc_to_tensor` once
      - `main_function` many times
      - `clFlush`
      - `tensor_to_bhwc` once
    - `clEnqueueReadBuffer`
    - `clFinish`
  - gpu time breakdown
    - `clEnqueueWriteBuffer`: 2.5%
    - `clEnqueueNDRangeKernel(bhwc_to_tensor)`: 1.5%
    - `clEnqueueNDRangeKernel(main_function)`: 84%
    - `clEnqueueNDRangeKernel(tensor_to_bhwc)`: 1%
    - `clEnqueueReadBuffer`: 11%
