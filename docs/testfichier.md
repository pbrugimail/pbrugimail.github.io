# Orsntein Model 1.  mean=0 theta=1


 $h_{n+1} = h_n + \theta (\mu - h_n)dt + \sigma \sqrt{dt} \epsilon_{n+1}$ with $\epsilon_{n+1} \sim N(0,1)$

$x_{n+1}= h_{n+1}-h_n$ 

## test subtitle 

We estimate the probabilities for $x_{n+1}$ to belong to some arbitrary buckets, based on the observations $(x_1,x_2, \cdots, x_n)$.

The estimation is done by comparing the probabilities predicted by the model for the buckets to the one-hot encoding $y_h$ of the bucket of $x_{n+1}$

We compare these predicted probabilities to the the target probability 
 $p_j(x_{n+1}|h_n,model)$ for the bucket $B_j$.

The $(h_1, \cdots, h_{n+1})$ are simulated with $h_0=\mu$ and from them the $(x_1,x_2, \cdots, x_{n+1})$ \\

dataset is the family of the sequences $(x_1,x_1, \cdots, x_{n+1})$

We simulate 4291 sequences

If $\theta=0$ we find the same results as in the pure normal model (model3) with the same seed.

Attention "Shuffle= False" for replication but is not recommended to estimate

https://kazemnejad.com/blog/transformer_architecture_positional_encoding/


For reproductibility (model 3= model8)

1. fixed seeds
2. kernel initializers
3. no shuffle in the optimizer

https://colab.research.google.com/notebooks/pro.ipynb

Your resources are not unlimited in Colab. To make the most of Colab, avoid using resources when you don't need them. For example, only use a GPU when required and close Colab tabs when finished.




# Libraries and Models


```python
import numpy as np
from scipy.stats import norm
from sklearn.preprocessing import OneHotEncoder
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers, initializers
from tensorflow.keras.layers import Input, MultiHeadAttention
from tensorflow.keras.models import Model
import matplotlib.pyplot as plt
from keras.callbacks import Callback
```

# Data Generation 


```python
############# Input parameters  ################################################
seed=1
np.random.seed(seed) 
tf.random.set_seed(seed)

mean = 0                 # mean parameter for the (h_i)
theta = 1                 # mean-reversion coefficient 
sigma = 1                 # standard deviation GT modif
std=sigma
dt = 1
#dt= 1/255                    # time step for Orsntein Uhlenbeck
n_buckets = 7             # number of buckets
n_classes = n_buckets
seq_length = 5          # length of the (x_i) sequence used for prediction
nb_total = 4291           # number of trajectories to generate
nb_train = 3601          # number of trajectories for training
pe =1                     # pe=1 for positional encoding, otherwise 0

################# Data generation ##############################################
def ornstein_uhlenbeck(dt, num_steps, num_trajectories, mean, theta, sigma):
    h = np.zeros((num_steps, num_trajectories))
    h[0, :] = mean
    for t in range(1, num_steps):
        dh = theta * (mean - h[t-1, :]) * dt + sigma * np.sqrt(dt) * np.random.normal(size=(num_trajectories,))
        h[t, :] = h[t-1, :] + dh
    return h

def new_ornstein_uhlenbeck(dt, num_steps, num_trajectories, mean, theta, sigma):
    x=np.zeros((num_steps, num_trajectories))
    sqrtdt=np.sqrt(dt)
    x[0, :] = mean
    for jj in range(num_steps-1):
        x[jj+1, :] = x[jj,:] + theta*(mean - x[jj, :])*dt + sigma*sqrtdt*np.random.randn(num_trajectories)        
    return x

trajectories = ornstein_uhlenbeck(dt, seq_length+2, nb_total, mean, theta, sigma)
trajectories2 = ornstein_uhlenbeck(dt, int(1/dt), nb_total, mean, theta, sigma)
trajectories2 = ornstein_uhlenbeck(dt, seq_length+2, nb_total, mean, theta, sigma)
trajectories = trajectories2[-(seq_length+2):,:]# GT modif
dataseto = np.diff(trajectories, axis=0)
dataseto[0,:]=trajectories[0,:]#GT modif
#dataset=dataseto.reshape(dataseto.shape[1],dataseto.shape[0],1)
dataset=(dataseto.T).copy()#transpose
dataset=dataset[:,:,None]#add one dimension
```


```python
if(True):#debug only GT modif
    plt.figure('ou plot',figsize=(4*3,3))
    plt.subplot(1,4,1)
    plt.plot(trajectories[:,:10])
    plt.subplot(1,4,2)
    plt.scatter(trajectories[-2,:],trajectories[-1,:])

    plt.subplot(1,4,3)
    plt.plot(trajectories2[:,:10])
    plt.subplot(1,4,4)
    plt.scatter(trajectories2[-2,:],trajectories2[-1,:])
```


    
![png](images/output_5_0.png)
    



```python
################## Position encoding for (x_1,...x_n) ##########################
def get_positional_encoding(seq_length, input_dim):
    position = np.arange(seq_length)[:, np.newaxis]
    div_term = np.exp(np.arange(0, input_dim, 2) * -(np.log(10000.0) / input_dim))
    positional_encoding = np.zeros((seq_length, input_dim))
    positional_encoding[:, 0::2] = np.sin(position * div_term)
    positional_encoding[:, 1::2] = np.cos(position * div_term)
    return positional_encoding

input_dim = dataset.shape[2]
positional_encoding = get_positional_encoding(seq_length, input_dim)

################# train, validation and test sets ##############################
train_set = dataset[:nb_train,:,:] # indices (0,3600)
x_train= train_set[:,:seq_length,:]+ positional_encoding*pe 
y_train= train_set[:,seq_length,:] # y=x_{n+1}

test_set = dataset[nb_train:,:,:] # indices (3601, end)
x_test= test_set[:,:seq_length,:]+ positional_encoding*pe
y_test= test_set[:,seq_length,:]
hidden = trajectories[-2,:] # extract h_n for each sequence

################# One hot Encoding of the x_{n+1} ##############################
echantillon = y_train  # Assuming this should be y_train
quantiles = np.percentile(echantillon, np.linspace(0, 100, n_classes + 1))
buckets = quantiles[1:-1]  # Defining the extremities of the buckets
encoder = OneHotEncoder(categories='auto')
encoder.fit(np.arange(1, n_classes + 1).reshape(-1, 1))

def update_bucket(y):
    bucket = np.ones_like(y)
    for j in range(y.shape[0]):
        for i in range(len(buckets)):
            if y[j] > buckets[i]:
                bucket[j] += 1
    bucket_encoded = encoder.transform(bucket.reshape(-1, 1))
    bucket_encoded_dense = bucket_encoded.toarray()
    return bucket_encoded_dense

yh_test = update_bucket(y_test)
yh_train = update_bucket(y_train)
```


```python
dataset.shape,nb_train, dataset[:nb_train,:,:].shape,test_set.shape, train_set[12,seq_length,:],train_set[12,-1,:]
```




    ((4291, 6, 1),
     3601,
     (3601, 6, 1),
     (690, 6, 1),
     array([-1.37908911]),
     array([-1.37908911]))




```python
plt.scatter(x_train[:,-2,0],x_train[:,-1,0])
```




    <matplotlib.collections.PathCollection at 0x7f037c749c40>




    
![png](images/output_8_1.png)
    



```python
#plt.hist(y_test)
#buckets
plt.figure('hist')
plt.subplot(1,3,1)
plt.hist(y_train,alpha=0.5,density=True)
plt.legend(['train'])
plt.subplot(1,3,2)
plt.hist(y_test,alpha=0.5,density=True)
#plt.hist(trajectories2[-1,:]-trajectories2[-2,:],alpha=0.5,density=True)
plt.legend(['test'])
plt.subplot(1,3,3)#now take choice at random
plt.hist(dataset[np.random.choice(dataset.shape[0],y_test.shape[0]),-1,0],alpha=0.5,density=True)
plt.legend(['other test'])

(np.mean(y_train),np.mean(y_test),
np.sum(yh_train,axis=0),np.sum(yh_test,axis=0),
buckets,np.percentile(y_test, np.linspace(0, 100, n_classes + 1))[1:-1],
np.percentile(y_train, [(jj+1)/n_classes for jj in range(n_classes)]),
np.percentile(y_test, [(jj+1)/n_classes for jj in range(n_classes)]),
 ' traj2',
np.percentile(trajectories2[-1,:]-trajectories2[-2,:], [(jj+1)/n_classes for jj in range(n_classes)]),
 np.min(y_train),np.max(y_train), np.min(y_test),np.max(y_test),
)
```




    (0.004603860726345346,
     0.011161597647578432,
     array([515., 514., 514., 515., 514., 514., 515.]),
     array([102., 101., 101.,  78., 115.,  88., 105.]),
     array([-1.5479577 , -0.80670365, -0.25563457,  0.26803618,  0.80969967,
             1.5339966 ]),
     array([-1.5627276 , -0.85904886, -0.30205722,  0.34134502,  0.79836054,
             1.56827421]),
     array([-4.44472201, -3.88288979, -3.61015139, -3.52828632, -3.45739792,
            -3.37985013, -3.29411496]),
     array([-3.70843862, -3.5403157 , -3.28413025, -3.24036306, -3.09373415,
            -3.06825096, -3.06236183]),
     ' traj2',
     array([-4.34645412, -3.77569145, -3.59824096, -3.51079371, -3.4352208 ,
            -3.30901424, -3.28211322]),
     -5.54744557087057,
     6.463217835967607,
     -3.774340569448924,
     5.389314722232845)




    
![png](images/output_9_1.png)
    


# Model Build and Compile


```python
###################### Parameters #####################################
head_size=256               # will be the dimension of : the query, key and value vectors for each head 
num_heads=4                 # total hidden dimension is 256*4 = 1024
ff_dim=4                    # number of filters in the FeedForward Layer
num_transformer_blocks=4    # Nb of iterations of the Encoder block
mlp_units=[128]             # list of the number of units of the final "dense layers" - just one with 128 neurons
mlp_units=[10]             # list of the number of units of the final "dense layers" - just one with 128 neurons GT modif
mlp_dropout=0.4             # frequency of dropout for the dropout layers
mlp_dropout=0.0             # frequency of dropout for the dropout layers GT modif 
dropout=0.25                # frequency of dropout inside the MultiHEad Attention
input_shape = x_train.shape[1:] # (500,1)

# Definition of the Encoder Block 
def transformer_encoder(inputs, head_size, num_heads, ff_dim, dropout=0):
    # Normalization batch by batch and affine parameters to learn
    x = layers.LayerNormalization(epsilon=1e-6)(inputs)
    x = layers.MultiHeadAttention(key_dim=head_size, num_heads=num_heads, dropout=dropout)(x, x)
    x = layers.Dropout(dropout)(x)
#    res = x + inputs
    res = x + inputs 

    # Feed Forward Part
    x = layers.LayerNormalization(epsilon=1e-6)(res)
    x = layers.Conv1D(filters=ff_dim, kernel_size=1, activation="relu", kernel_initializer=initializers.RandomNormal(seed=seed))(x)
    x = layers.Dropout(dropout)(x)
    x = layers.Conv1D(filters=inputs.shape[-1], kernel_size=1, kernel_initializer=initializers.RandomNormal(seed=seed))(x)
    return x + res

def build_model(
    input_shape,
    head_size,
    num_heads,
    ff_dim,
    num_transformer_blocks,
    mlp_units,
    dropout=0,
    mlp_dropout=0,
):
    inputs = keras.Input(shape=input_shape)
    x = inputs
    for _ in range(num_transformer_blocks):
        x = transformer_encoder(x, head_size, num_heads, ff_dim, dropout)

    x = layers.GlobalAveragePooling1D(data_format="channels_first")(x)
    for dim in mlp_units:
        x = layers.Dense(dim, activation="relu", kernel_initializer=initializers.RandomNormal(seed=seed))(x)
        x = layers.Dropout(mlp_dropout)(x)
    outputs = layers.Dense(n_buckets, activation="softmax", kernel_initializer=initializers.RandomNormal(seed=seed))(x)
    return keras.Model(inputs, outputs)

def build_model_fc(input_shape,head_size,num_heads,ff_dim,
    num_transformer_blocks, mlp_units, dropout=0,mlp_dropout=0):
    inputs = keras.Input(shape=(input_shape[0],))
    x = inputs
    for _ in range(num_transformer_blocks):#GT modif
      for dim in mlp_units:
        x = layers.Dense(dim, activation="relu", kernel_initializer=initializers.RandomNormal(seed=seed))(x)
    outputs = layers.Dense(n_buckets, activation="softmax", kernel_initializer=initializers.RandomNormal(seed=seed))(x)
    return keras.Model(inputs, outputs)



model = build_model(
    input_shape,              # (seq_length, d_model) here (500,1)
    head_size=head_size,      # dimension of the query key and value vectors 
    num_heads=num_heads,      # usually 4
    ff_dim=ff_dim,            # number of filters in the feed forward
    num_transformer_blocks=num_transformer_blocks, # usually iteration of 4 encoders
    mlp_units=mlp_units,      # list of the final "dense layers" to add - just one with 128 neurons
    mlp_dropout=mlp_dropout,  # probability of dropout in the final dense lawyer of shape (None,128)
    dropout=0.25)             # probability of dropout in the attention score matrix and dropout layer of the Attention mechanism

model.compile(
    loss="categorical_crossentropy", 
    optimizer=keras.optimizers.Adam(learning_rate=1e-3),
    metrics=["categorical_accuracy"])
#model.summary()
```


```python
model.summary()
```

    Model: "model"
    __________________________________________________________________________________________________
     Layer (type)                   Output Shape         Param #     Connected to                     
    ==================================================================================================
     input_1 (InputLayer)           [(None, 5, 1)]       0           []                               
                                                                                                      
     layer_normalization (LayerNorm  (None, 5, 1)        2           ['input_1[0][0]']                
     alization)                                                                                       
                                                                                                      
     multi_head_attention (MultiHea  (None, 5, 1)        7169        ['layer_normalization[0][0]',    
     dAttention)                                                      'layer_normalization[0][0]']    
                                                                                                      
     dropout (Dropout)              (None, 5, 1)         0           ['multi_head_attention[0][0]']   
                                                                                                      
     tf.__operators__.add (TFOpLamb  (None, 5, 1)        0           ['dropout[0][0]',                
     da)                                                              'input_1[0][0]']                
                                                                                                      
     layer_normalization_1 (LayerNo  (None, 5, 1)        2           ['tf.__operators__.add[0][0]']   
     rmalization)                                                                                     
                                                                                                      
     conv1d (Conv1D)                (None, 5, 4)         8           ['layer_normalization_1[0][0]']  
                                                                                                      
     dropout_1 (Dropout)            (None, 5, 4)         0           ['conv1d[0][0]']                 
                                                                                                      
     conv1d_1 (Conv1D)              (None, 5, 1)         5           ['dropout_1[0][0]']              
                                                                                                      
     tf.__operators__.add_1 (TFOpLa  (None, 5, 1)        0           ['conv1d_1[0][0]',               
     mbda)                                                            'tf.__operators__.add[0][0]']   
                                                                                                      
     layer_normalization_2 (LayerNo  (None, 5, 1)        2           ['tf.__operators__.add_1[0][0]'] 
     rmalization)                                                                                     
                                                                                                      
     multi_head_attention_1 (MultiH  (None, 5, 1)        7169        ['layer_normalization_2[0][0]',  
     eadAttention)                                                    'layer_normalization_2[0][0]']  
                                                                                                      
     dropout_2 (Dropout)            (None, 5, 1)         0           ['multi_head_attention_1[0][0]'] 
                                                                                                      
     tf.__operators__.add_2 (TFOpLa  (None, 5, 1)        0           ['dropout_2[0][0]',              
     mbda)                                                            'tf.__operators__.add_1[0][0]'] 
                                                                                                      
     layer_normalization_3 (LayerNo  (None, 5, 1)        2           ['tf.__operators__.add_2[0][0]'] 
     rmalization)                                                                                     
                                                                                                      
     conv1d_2 (Conv1D)              (None, 5, 4)         8           ['layer_normalization_3[0][0]']  
                                                                                                      
     dropout_3 (Dropout)            (None, 5, 4)         0           ['conv1d_2[0][0]']               
                                                                                                      
     conv1d_3 (Conv1D)              (None, 5, 1)         5           ['dropout_3[0][0]']              
                                                                                                      
     tf.__operators__.add_3 (TFOpLa  (None, 5, 1)        0           ['conv1d_3[0][0]',               
     mbda)                                                            'tf.__operators__.add_2[0][0]'] 
                                                                                                      
     layer_normalization_4 (LayerNo  (None, 5, 1)        2           ['tf.__operators__.add_3[0][0]'] 
     rmalization)                                                                                     
                                                                                                      
     multi_head_attention_2 (MultiH  (None, 5, 1)        7169        ['layer_normalization_4[0][0]',  
     eadAttention)                                                    'layer_normalization_4[0][0]']  
                                                                                                      
     dropout_4 (Dropout)            (None, 5, 1)         0           ['multi_head_attention_2[0][0]'] 
                                                                                                      
     tf.__operators__.add_4 (TFOpLa  (None, 5, 1)        0           ['dropout_4[0][0]',              
     mbda)                                                            'tf.__operators__.add_3[0][0]'] 
                                                                                                      
     layer_normalization_5 (LayerNo  (None, 5, 1)        2           ['tf.__operators__.add_4[0][0]'] 
     rmalization)                                                                                     
                                                                                                      
     conv1d_4 (Conv1D)              (None, 5, 4)         8           ['layer_normalization_5[0][0]']  
                                                                                                      
     dropout_5 (Dropout)            (None, 5, 4)         0           ['conv1d_4[0][0]']               
                                                                                                      
     conv1d_5 (Conv1D)              (None, 5, 1)         5           ['dropout_5[0][0]']              
                                                                                                      
     tf.__operators__.add_5 (TFOpLa  (None, 5, 1)        0           ['conv1d_5[0][0]',               
     mbda)                                                            'tf.__operators__.add_4[0][0]'] 
                                                                                                      
     layer_normalization_6 (LayerNo  (None, 5, 1)        2           ['tf.__operators__.add_5[0][0]'] 
     rmalization)                                                                                     
                                                                                                      
     multi_head_attention_3 (MultiH  (None, 5, 1)        7169        ['layer_normalization_6[0][0]',  
     eadAttention)                                                    'layer_normalization_6[0][0]']  
                                                                                                      
     dropout_6 (Dropout)            (None, 5, 1)         0           ['multi_head_attention_3[0][0]'] 
                                                                                                      
     tf.__operators__.add_6 (TFOpLa  (None, 5, 1)        0           ['dropout_6[0][0]',              
     mbda)                                                            'tf.__operators__.add_5[0][0]'] 
                                                                                                      
     layer_normalization_7 (LayerNo  (None, 5, 1)        2           ['tf.__operators__.add_6[0][0]'] 
     rmalization)                                                                                     
                                                                                                      
     conv1d_6 (Conv1D)              (None, 5, 4)         8           ['layer_normalization_7[0][0]']  
                                                                                                      
     dropout_7 (Dropout)            (None, 5, 4)         0           ['conv1d_6[0][0]']               
                                                                                                      
     conv1d_7 (Conv1D)              (None, 5, 1)         5           ['dropout_7[0][0]']              
                                                                                                      
     tf.__operators__.add_7 (TFOpLa  (None, 5, 1)        0           ['conv1d_7[0][0]',               
     mbda)                                                            'tf.__operators__.add_6[0][0]'] 
                                                                                                      
     global_average_pooling1d (Glob  (None, 5)           0           ['tf.__operators__.add_7[0][0]'] 
     alAveragePooling1D)                                                                              
                                                                                                      
     dense (Dense)                  (None, 10)           60          ['global_average_pooling1d[0][0]'
                                                                     ]                                
                                                                                                      
     dropout_8 (Dropout)            (None, 10)           0           ['dense[0][0]']                  
                                                                                                      
     dense_1 (Dense)                (None, 7)            77          ['dropout_8[0][0]']              
                                                                                                      
    ==================================================================================================
    Total params: 28,881
    Trainable params: 28,881
    Non-trainable params: 0
    __________________________________________________________________________________________________


# Model Run


```python
# Restart after patience until total number of epochs
desired_epochs = 50# GT modif
patience =3
patience =10#GT modif
model_split =0.2
total_epochs = 0

callbacks = [
    keras.callbacks.EarlyStopping(patience=patience, restore_best_weights=True),
    # Add any other desired callbacks
    keras.callbacks.History()#GT modif
]

while total_epochs < desired_epochs:
    historical = model.fit(
        x_train,
        yh_train,
        validation_split=model_split,
        epochs=desired_epochs - total_epochs,
        batch_size=64,      
        shuffle=True,
        callbacks=callbacks,
        verbose=1
    )  
    if callbacks[0].stopped_epoch is not None:
        print("Early stopping occurred at epoch:", callbacks[0].stopped_epoch +1)
        #total_epochs += callbacks[0].stopped_epoch + 1  # Increment total epochs with early stopping epoch
        total_epochs=len(model.history.history['val_loss'])#GT modif
        print("total_epochs so far",total_epochs)
    else:
        total_epochs = desired_epochs  # Reached desired epochs without early stopping

```

    Epoch 1/50
    45/45 [==============================] - 65s 84ms/step - loss: 1.9447 - categorical_accuracy: 0.1583 - val_loss: 1.9416 - val_categorical_accuracy: 0.1540
    Epoch 2/50
    45/45 [==============================] - 1s 25ms/step - loss: 1.9313 - categorical_accuracy: 0.1715 - val_loss: 1.9213 - val_categorical_accuracy: 0.1831
    Epoch 3/50
    45/45 [==============================] - 1s 23ms/step - loss: 1.9039 - categorical_accuracy: 0.2069 - val_loss: 1.8902 - val_categorical_accuracy: 0.2219
    Epoch 4/50
    45/45 [==============================] - 1s 27ms/step - loss: 1.8665 - categorical_accuracy: 0.2333 - val_loss: 1.8577 - val_categorical_accuracy: 0.2275
    Epoch 5/50
    45/45 [==============================] - 1s 28ms/step - loss: 1.8379 - categorical_accuracy: 0.2448 - val_loss: 1.8318 - val_categorical_accuracy: 0.2302
    Epoch 6/50
    45/45 [==============================] - 1s 28ms/step - loss: 1.8162 - categorical_accuracy: 0.2451 - val_loss: 1.8142 - val_categorical_accuracy: 0.2413
    Epoch 7/50
    45/45 [==============================] - 1s 27ms/step - loss: 1.7926 - categorical_accuracy: 0.2559 - val_loss: 1.7971 - val_categorical_accuracy: 0.2497
    Epoch 8/50
    45/45 [==============================] - 1s 25ms/step - loss: 1.7768 - categorical_accuracy: 0.2608 - val_loss: 1.8010 - val_categorical_accuracy: 0.2288
    Epoch 9/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.7651 - categorical_accuracy: 0.2639 - val_loss: 1.7757 - val_categorical_accuracy: 0.2552
    Epoch 10/50
    45/45 [==============================] - 1s 28ms/step - loss: 1.7520 - categorical_accuracy: 0.2663 - val_loss: 1.7656 - val_categorical_accuracy: 0.2635
    Epoch 11/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.7414 - categorical_accuracy: 0.2656 - val_loss: 1.7586 - val_categorical_accuracy: 0.2510
    Epoch 12/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.7341 - categorical_accuracy: 0.2743 - val_loss: 1.7545 - val_categorical_accuracy: 0.2483
    Epoch 13/50
    45/45 [==============================] - 1s 24ms/step - loss: 1.7283 - categorical_accuracy: 0.2826 - val_loss: 1.7459 - val_categorical_accuracy: 0.2691
    Epoch 14/50
    45/45 [==============================] - 1s 27ms/step - loss: 1.7214 - categorical_accuracy: 0.2840 - val_loss: 1.7455 - val_categorical_accuracy: 0.2621
    Epoch 15/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.7174 - categorical_accuracy: 0.2792 - val_loss: 1.7363 - val_categorical_accuracy: 0.2607
    Epoch 16/50
    45/45 [==============================] - 1s 30ms/step - loss: 1.7138 - categorical_accuracy: 0.2830 - val_loss: 1.7341 - val_categorical_accuracy: 0.2621
    Epoch 17/50
    45/45 [==============================] - 1s 28ms/step - loss: 1.7099 - categorical_accuracy: 0.2858 - val_loss: 1.7351 - val_categorical_accuracy: 0.2691
    Epoch 18/50
    45/45 [==============================] - 1s 27ms/step - loss: 1.7083 - categorical_accuracy: 0.2906 - val_loss: 1.7297 - val_categorical_accuracy: 0.2594
    Epoch 19/50
    45/45 [==============================] - 1s 31ms/step - loss: 1.7069 - categorical_accuracy: 0.2941 - val_loss: 1.7270 - val_categorical_accuracy: 0.2594
    Epoch 20/50
    45/45 [==============================] - 1s 27ms/step - loss: 1.7050 - categorical_accuracy: 0.2903 - val_loss: 1.7249 - val_categorical_accuracy: 0.2607
    Epoch 21/50
    45/45 [==============================] - 1s 27ms/step - loss: 1.7026 - categorical_accuracy: 0.2931 - val_loss: 1.7236 - val_categorical_accuracy: 0.2635
    Epoch 22/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.6998 - categorical_accuracy: 0.2889 - val_loss: 1.7238 - val_categorical_accuracy: 0.2663
    Epoch 23/50
    45/45 [==============================] - 1s 23ms/step - loss: 1.7002 - categorical_accuracy: 0.2958 - val_loss: 1.7223 - val_categorical_accuracy: 0.2649
    Epoch 24/50
    45/45 [==============================] - 1s 27ms/step - loss: 1.7006 - categorical_accuracy: 0.2875 - val_loss: 1.7195 - val_categorical_accuracy: 0.2635
    Epoch 25/50
    45/45 [==============================] - 1s 30ms/step - loss: 1.6982 - categorical_accuracy: 0.2927 - val_loss: 1.7188 - val_categorical_accuracy: 0.2649
    Epoch 26/50
    45/45 [==============================] - 1s 25ms/step - loss: 1.6985 - categorical_accuracy: 0.2965 - val_loss: 1.7181 - val_categorical_accuracy: 0.2691
    Epoch 27/50
    45/45 [==============================] - 1s 21ms/step - loss: 1.6978 - categorical_accuracy: 0.2948 - val_loss: 1.7181 - val_categorical_accuracy: 0.2607
    Epoch 28/50
    45/45 [==============================] - 1s 27ms/step - loss: 1.6958 - categorical_accuracy: 0.2951 - val_loss: 1.7179 - val_categorical_accuracy: 0.2677
    Epoch 29/50
    45/45 [==============================] - 1s 24ms/step - loss: 1.6959 - categorical_accuracy: 0.2993 - val_loss: 1.7213 - val_categorical_accuracy: 0.2802
    Epoch 30/50
    45/45 [==============================] - 1s 28ms/step - loss: 1.6950 - categorical_accuracy: 0.2965 - val_loss: 1.7166 - val_categorical_accuracy: 0.2774
    Epoch 31/50
    45/45 [==============================] - 1s 29ms/step - loss: 1.6960 - categorical_accuracy: 0.2937 - val_loss: 1.7170 - val_categorical_accuracy: 0.2802
    Epoch 32/50
    45/45 [==============================] - 1s 22ms/step - loss: 1.6951 - categorical_accuracy: 0.2983 - val_loss: 1.7213 - val_categorical_accuracy: 0.2677
    Epoch 33/50
    45/45 [==============================] - 1s 22ms/step - loss: 1.6947 - categorical_accuracy: 0.3000 - val_loss: 1.7187 - val_categorical_accuracy: 0.2774
    Epoch 34/50
    45/45 [==============================] - 1s 24ms/step - loss: 1.6945 - categorical_accuracy: 0.3038 - val_loss: 1.7159 - val_categorical_accuracy: 0.2718
    Epoch 35/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.6950 - categorical_accuracy: 0.2962 - val_loss: 1.7148 - val_categorical_accuracy: 0.2802
    Epoch 36/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.6931 - categorical_accuracy: 0.2965 - val_loss: 1.7157 - val_categorical_accuracy: 0.2760
    Epoch 37/50
    45/45 [==============================] - 1s 25ms/step - loss: 1.6933 - categorical_accuracy: 0.2990 - val_loss: 1.7165 - val_categorical_accuracy: 0.2788
    Epoch 38/50
    45/45 [==============================] - 1s 27ms/step - loss: 1.6932 - categorical_accuracy: 0.3003 - val_loss: 1.7162 - val_categorical_accuracy: 0.2746
    Epoch 39/50
    45/45 [==============================] - 1s 29ms/step - loss: 1.6933 - categorical_accuracy: 0.2958 - val_loss: 1.7143 - val_categorical_accuracy: 0.2926
    Epoch 40/50
    45/45 [==============================] - 1s 25ms/step - loss: 1.6930 - categorical_accuracy: 0.2976 - val_loss: 1.7151 - val_categorical_accuracy: 0.2802
    Epoch 41/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.6924 - categorical_accuracy: 0.3028 - val_loss: 1.7141 - val_categorical_accuracy: 0.2899
    Epoch 42/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.6920 - categorical_accuracy: 0.3010 - val_loss: 1.7139 - val_categorical_accuracy: 0.2940
    Epoch 43/50
    45/45 [==============================] - 1s 24ms/step - loss: 1.6928 - categorical_accuracy: 0.3035 - val_loss: 1.7147 - val_categorical_accuracy: 0.2885
    Epoch 44/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.6937 - categorical_accuracy: 0.2962 - val_loss: 1.7146 - val_categorical_accuracy: 0.2899
    Epoch 45/50
    45/45 [==============================] - 1s 28ms/step - loss: 1.6926 - categorical_accuracy: 0.3042 - val_loss: 1.7137 - val_categorical_accuracy: 0.2760
    Epoch 46/50
    45/45 [==============================] - 1s 24ms/step - loss: 1.6924 - categorical_accuracy: 0.3035 - val_loss: 1.7152 - val_categorical_accuracy: 0.2760
    Epoch 47/50
    45/45 [==============================] - 1s 27ms/step - loss: 1.6928 - categorical_accuracy: 0.3049 - val_loss: 1.7147 - val_categorical_accuracy: 0.2816
    Epoch 48/50
    45/45 [==============================] - 1s 28ms/step - loss: 1.6924 - categorical_accuracy: 0.2986 - val_loss: 1.7136 - val_categorical_accuracy: 0.2802
    Epoch 49/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.6918 - categorical_accuracy: 0.3000 - val_loss: 1.7151 - val_categorical_accuracy: 0.2802
    Epoch 50/50
    45/45 [==============================] - 1s 26ms/step - loss: 1.6931 - categorical_accuracy: 0.2986 - val_loss: 1.7136 - val_categorical_accuracy: 0.2843
    Early stopping occurred at epoch: 1
    total_epochs so far 50


# Checking and Analyzing the Results

### Calculating the target and predicted probabilities


```python
##### Calculating the target probabilities p_j(x_{n+1}|h_n) for each trajectory 
coeff=sigma*np.sqrt(dt)
def calculate_proba(buckets, hidden_states): #hidden_states will be a np of the h_n
    result = []
    for c in hidden_states:
        new_array = (buckets - theta*(mean-c))/coeff
        cdf_values = norm.cdf(new_array)
        cdf_values = np.insert(cdf_values, [0, len(cdf_values)], [0, 1])
        differences = np.diff(cdf_values)
        result.append(differences)       
    result=np.vstack(result)
    return result

p_target =calculate_proba(buckets, hidden)
p_target_train = p_target[:nb_train,:]
p_target_test =p_target[nb_train:,:]

#### Calculating the predicted probabilities p_j(x_{n+1}|x_1,..,x_n) for each trajectory 
pred_test=model.predict(x_test)
mean_pred_test = np.mean(pred_test, axis=0)
pred_train=model.predict(x_train)
mean_pred_train = np.mean(pred_train, axis=0)
###### model evaluation ########################################################
model.evaluate(x_test, yh_test, verbose=1)
```

    22/22 [==============================] - 2s 8ms/step
    113/113 [==============================] - 1s 7ms/step
    22/22 [==============================] - 0s 7ms/step - loss: 1.7262 - categorical_accuracy: 0.2899





    [1.7261608839035034, 0.28985506296157837]



###Average Cross Entropy between two families of distribution

The cross entropy $H(P,Q)$ where $P$ is the reference probability is defined as 
$$H(P,Q)=-\sum\limits_{i=1}^n p_iln(q_i)\geq H(P,P)$$

When there rare two families of probabilities $F_P=\{P^j\}$ and $F_Q=\{Q^j\}$ we calculate the average cross entropy as 
$$H(F_P,F_Q) = \frac{1}{m}\sum\limits_{j=1}^m H(P^j,Q^j)\geq H(F_P,F_P)$$


```python
############### analysing the cross entropies numbers ##########################
def individual_cross_entropies(family1, family2):
    n = family1.shape[0]  # Number of distributions in each family
    cross_entropies = np.zeros(n)

    for i in range(n):
        cross_entropy = -np.sum(family1[i] * np.log(family2[i]))
        cross_entropies[i] = cross_entropy
    return cross_entropies

###### target average_loss on train_set 
z_1=np.mean(individual_cross_entropies(yh_train,p_target_train))
z_2=np.mean(individual_cross_entropies(yh_train,pred_train))
print('ideal loss on train_set:',z_1 )
print('loss obtained on train_set:',z_2 )
###### target average_loss on test_set 
z_1=np.mean(individual_cross_entropies(p_target_test,p_target_test))
z_2=np.mean(individual_cross_entropies(p_target_test,pred_test))
print('ideal loss on test_set:',z_1 )
print('loss obtained on test_set:',z_2 )
```

    ideal loss on train_set: 1.606546799141291
    loss obtained on train_set: 1.6949231457922136
    ideal loss on test_set: 1.6159212836579495
    loss obtained on test_set: 1.7048638754808851



```python
########### visualisation predicted proba on train_set #########################
fig, axes = plt.subplots(nrows=1, ncols=6, figsize=(14, 4)) 
pred_train_a=np.mean(pred_train,axis=0)
for i, ax in enumerate(axes.flat):
    ax.hist(pred_train[:, i], bins=10)
    ax.set_xlabel('Predicted_Train')
    ax.set_ylabel('Nb of predictions')
    ax.set_title('Bucket' + str(i))
    ax.set_title('Bucket'+str(i)+':'+str(np.round(100*pred_train_a[i],1))+'%')
    ax.grid(True)
    ax.set_xlim(0, 0.5)
    ax.axvline(x=pred_train_a[i], color='r', linestyle='--')
plt.show()

########### visualisation predicted proba on test_set ##########################
fig, axes = plt.subplots(nrows=1, ncols=6, figsize=(14, 4)) 
pred_test_a=np.mean(pred_test,axis=0)
for i, ax in enumerate(axes.flat):
    ax.hist(pred_test[:, i], bins=10)
    ax.set_xlabel('Predicted_Test')
    ax.set_ylabel('Nb of predictions')
    ax.set_title('Bucket' + str(i))
    ax.set_title('Bucket'+str(i)+':'+str(np.round(100*pred_test_a[i],1))+'%')
    ax.grid(True)
    ax.set_xlim(0, 0.5)
    ax.axvline(x=pred_test_a[i], color='r', linestyle='--')
plt.show()

########## visualisation target probabilities on test_set #####################
p_target_test=p_target[nb_train:,:]
p_target_test_a= np.average(p_target_test, axis=0)
fig, axes = plt.subplots(nrows=1, ncols=6, figsize=(14, 4)) 
for i, ax in enumerate(axes.flat):
    ax.hist(p_target_test[:, i], bins=10, range=(0, 0.5))
    ax.set_xlabel('Target Proba Test')
    ax.set_ylabel('Observations')
    ax.axvline(x=p_target_test_a[i], color='r', linestyle='--')
    ax.set_title('Bucket'+str(i)+':'+str(np.round(100*p_target_test_a[i],1))+'%')
    ax.grid(True)
plt.tight_layout()  # Ajuste automatiquement les espacements entre les sous-graphes
plt.show()

# Create a scatter plot for the test set
plt.figure("scatter",figsize=(14,3))
for jj in range(n_buckets):
  plt.subplot(1,n_buckets,jj+1)
  plt.scatter(hidden[nb_train:], p_target_test[:,jj],s=10)
  plt.scatter(hidden[nb_train:], pred_test[:,jj],s=5,alpha=0.2)
# Add labels and title
  plt.xlabel('hidden')
  plt.ylabel('proba test')
  plt.ylim(0, 0.5) 
  plt.title('bucket '+ str(jj))
plt.show()

# Create a scatter plot for the train set
plt.figure("scatter",figsize=(14,3))
for jj in range(n_buckets):
  plt.subplot(1,n_buckets,jj+1)
  plt.scatter(hidden[:nb_train], p_target_train[:,jj],s=10)
  plt.scatter(hidden[:nb_train], pred_train[:,jj],s=5,alpha=0.05)
# Add labels and title
  plt.xlabel('hidden')
  plt.ylabel('proba train')
  plt.ylim(0, 0.5)
  plt.title('bucket '+ str(jj))
```


    
![png](images/output_20_0.png)
    



    
![png](images/output_20_1.png)
    



    
![png](images/output_20_2.png)
    



    
![png](images/output_20_3.png)
    



    
![png](images/output_20_4.png)
    


# Checking dimensions and Runs Data


```python
###################### Checking input data #####################################
print("dataset.shape, (x_1,...x_{n+1}) :",dataset.shape)
print("train_set.shape :",train_set.shape)
print("x_train.shape, sequences (x_1,..x_n) :",x_train.shape)
print("y_train.shape :",y_train.shape)
print("yh_train.shape :",yh_train.shape)
print("x_test.shape :",x_test.shape)
print("y_test.shape :",y_test.shape)
print("yh_test.shape :",yh_test.shape)
print("buckets :", buckets)
print("np.mean(yh_train,axis=0):", "\n",np.round(np.mean(yh_train,axis=0),4))
print("np.mean(yh_test,axis=0):", "\n",np.round(np.mean(yh_test,axis=0),4))

print("trajectories.shape, (h_0,...h_{n+1}) :",trajectories.shape)
print("positional_encoding.shape", positional_encoding.shape)
print("hidden.shape :",hidden.shape )
print("p_target.shape :",p_target.shape )
print("np.mean(p_target_train,axis=0):", "\n",np.round(np.mean(p_target_train,axis=0),4))
print("np.mean(p_target_test,axis=0):", "\n",np.round(np.mean(p_target_test,axis=0),4))

###############@ occurence of the buckets in the test_ data #####################
proba_test = np.mean(yh_test, axis=0)
proba_train = np.mean(yh_train, axis=0)
print("Occurence of the buckets in the train_set:", "\n",np.round(proba_train,3))
print("Occurence of the buckets in the test_set:","\n",np.round(proba_test,3))
print("average target proba per bucket:","\n",np.round(p_target_test_a,3)) 


#print(historical.history['loss'])
#print(historical.history['val_loss'])
#print(historical.history['categorical_accuracy'])
#print(historical.history['val_categorical_accuracy'])
```

    dataset.shape, (x_1,...x_{n+1}) : (4291, 6, 1)
    train_set.shape : (3601, 6, 1)
    x_train.shape, sequences (x_1,..x_n) : (3601, 5, 1)
    y_train.shape : (3601, 1)
    yh_train.shape : (3601, 7)
    x_test.shape : (690, 5, 1)
    y_test.shape : (690, 1)
    yh_test.shape : (690, 7)
    buckets : [-1.5479577  -0.80670365 -0.25563457  0.26803618  0.80969967  1.5339966 ]
    np.mean(yh_train,axis=0): 
     [0.143  0.1427 0.1427 0.143  0.1427 0.1427 0.143 ]
    np.mean(yh_test,axis=0): 
     [0.1478 0.1464 0.1464 0.113  0.1667 0.1275 0.1522]
    trajectories.shape, (h_0,...h_{n+1}) : (7, 4291)
    positional_encoding.shape (5, 1)
    hidden.shape : (4291,)
    p_target.shape : (4291, 7)
    np.mean(p_target_train,axis=0): 
     [0.1369 0.1467 0.1434 0.1464 0.1411 0.1445 0.1409]
    np.mean(p_target_test,axis=0): 
     [0.1404 0.1465 0.1424 0.1451 0.1399 0.1435 0.1421]
    Occurence of the buckets in the train_set: 
     [0.143 0.143 0.143 0.143 0.143 0.143 0.143]
    Occurence of the buckets in the test_set: 
     [0.148 0.146 0.146 0.113 0.167 0.128 0.152]
    average target proba per bucket: 
     [0.14  0.147 0.142 0.145 0.14  0.144 0.142]



```python

```
