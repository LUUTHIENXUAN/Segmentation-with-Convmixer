# Segmentation-with-Convmixer

```
import tensorflow as tf
from tensorflow.keras import layers

def activation_block(x):
    x = layers.Activation("gelu")(x)
    return layers.BatchNormalization()(x)


def conv_stem(x, filters: int, patch_size: int):
    x = layers.Conv2D(filters, kernel_size=patch_size, strides=patch_size)(x)
    return activation_block(x)


def conv_mixer_block(x, filters: int, kernel_size: int):
    # Depthwise convolution.
    x0 = x
    x = layers.DepthwiseConv2D(kernel_size=kernel_size, padding="same")(x)
    x = layers.Add()([activation_block(x), x0])  # Residual.
    
    # Pointwise convolution.
    x = layers.Conv2D(filters, kernel_size=1)(x)
    x = activation_block(x)

    return x


def get_conv_mixer_256_8(
    image_size=32, filters=256, depth=8, kernel_size=5, patch_size=2, num_classes=10
):
    """ConvMixer-256/8: https://openreview.net/pdf?id=TVHS5Y4dNvM.
    The hyperparameter values are taken from the paper.
    """
    inputs = keras.Input(shape=img_size + (3,))
    #x = layers.Rescaling(scale=1.0 / 255)(inputs)

    # Extract patch embeddings.
    x = conv_stem(inputs, filters, patch_size)

    # ConvMixer blocks.
    previous_block = []
    for _ in range(depth):
        x = conv_mixer_block(x, filters, kernel_size)
        
    x = layers.Conv2DTranspose(1,kernel_size=patch_size, strides=patch_size)(x) + tf.math.reduce_max(inputs, axis=3, keepdims=True)
    
    # Add a per-pixel classification layer
    outputs = layers.Conv2D(num_classes, 3, activation="softmax", padding="same")(x)

    return keras.Model(inputs, outputs)

# Free up RAM in case the model definition cells were run multiple times
keras.backend.clear_session()

# Build model
model = get_conv_mixer_256_8(image_size=img_size, num_classes=num_classes)
model.summary()
```
