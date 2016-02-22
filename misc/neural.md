http://www.doc.ic.ac.uk/~nd/surprise_96/journal/vol4/cs11/report.html
http://en.wikipedia.org/wiki/Multilayer_perceptron

BIO: a NEURON has many DENTRITES and one AXON.  Neurones are connected by
SYNAPSE.  Depending on the input from dentrites, a neuron fires or not.  A
neuron is also adaptive.

For every input, we calculate a weighted value:
value = sigma(input[i] * weight[i]) + bias

If we go linear,
f(value) = value

If we go simple,
f(value) = (value > threshold), a boolean

Instead of fires or not (boolean), for every input, we could, instead, specify
a value between 0.0 ~ 1.0 which is the frequency a neuron fires:
f(value) = 1 / (1 + e^(-value))

The hard part is to find suitable weights and/or bias, which is done through
learning.  A common algorithm of learning is backpropagation.

OCR of digits.

Each character is represented by a, say, 16x16 image.  Since there are 10
digits, there are 256 input units and 10 output units.

After learning, when fed with a image, we should see high activity on
appropriate output unit and low activity on the others.
