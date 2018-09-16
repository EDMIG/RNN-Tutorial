# Tensorflow에서 BasicDecoder에 넘겨 줄 수 있는 사용자 정의 RNN Wrapper Class를 만들어 보자.

### [쉬운 것부터 시작해보자]

아래 그림은 tf.nn.dynamic_rnn과 tf.contrib.seq2seq.dynamic_decode의 입력 구조를 비교한 그림이다.

![decode](./dynamic-rnn-decode2.png)
* Cell의 대표적인 예: BasicRNNCell, GRUCell, BasicLSTMCell
* 이런 cell은 RNNCell을 상속받은 class들이다.
* RNNCell을 상속받아 사용자 정의 RNN Wrapper class를 만들어  BasicDecoder로 넘겨줄 수 있다.
* 초간단으로 만들어지 RNN Wrapper의 sample code를 살펴보자.

```python
from tensorflow.contrib.rnn import RNNCell

class MyRnnWrapper(RNNCell):
    # property(output_size, state_size) 2개와 call을 정의하면 된다.
    def __init__(self,state_dim,name=None):
        super(MyRnnWrapper, self).__init__(name=name)
        self.sate_size = state_dim

    @property
    def output_size(self):
        return self.sate_size  

    @property
    def state_size(self):
        return self.sate_size  

    def call(self, inputs, state):
        # 이 call 함수를 통해 cell과 cell이 연결된다.
        cell_output = inputs
        next_state = state + 0.1
        return cell_output, next_state 
```
* 위 코드에서 볼 수 있듯이, RNNCell을 상속받아 class MyRnnWrapper 정의한다.
* MyRnnWrapper에서 반드시 구현하여야 하는 부분은 perperty output_size와 state_size 이다. 그리고 call이라 이름 붙혀진 class method를 구현해야 한다.
* output_size는 RNN Model에서 출력될 결과물의 dimension이고 state_size는 cell과 cell를 연결하는 hidden state의 크기이다. 
* call 함수(method)는 input과 직전 cell에서 넘겨 받은 hidden state값을 넘겨 받아, 필요한 계산을 수행한 후, 다음 단계로 넘겨 줄 next_state와 cell_output를 구하는 역할을 수행한다.

![decode](./cell-cell.png)
* 위 sample code은 넘겨받은 input을 그대로 output으로 내보내고, 넘겨 받은 state는 test 삼아, 0.1을 더해 next_state로 넘겨주는 의미 없는 example이다.
* 지금까지 만든 MyRnnWrapper를 dynamic_decode로 넘겨 돌려볼 수 있는 간단한 예를 만들어 돌려보자.

```python
# coding: utf-8
SOS_token = 0
EOS_token = 4
vocab_size = 5
x_data = np.array([[SOS_token, 3, 3, 2, 3, 2],[SOS_token, 3, 1, 2, 3, 1],[SOS_token, 1, 3, 2, 2, 1]], dtype=np.int32)

print("data shape: ", x_data.shape)
sess = tf.InteractiveSession()

output_dim = vocab_size
batch_size = len(x_data)
seq_length = x_data.shape[1]
embedding_dim = 2

init = np.arange(vocab_size*embedding_dim).reshape(vocab_size,-1)

train_mode = True
alignment_history_flag = False
with tf.variable_scope('test') as scope:
    # Make rnn
    cell = MyRnnWrapper(embedding_dim,"xxx")

    embedding = tf.get_variable("embedding", initializer=init.astype(np.float32),dtype = tf.float32)
    inputs = tf.nn.embedding_lookup(embedding, x_data) # batch_size  x seq_length x embedding_dim

    #######################################################

    initial_state = cell.zero_state(batch_size, tf.float32) #(batch_size x hidden_dim) x layer 개수 
    
    helper = tf.contrib.seq2seq.TrainingHelper(inputs, np.array([seq_length]*batch_size))

    decoder = tf.contrib.seq2seq.BasicDecoder(cell=cell,helper=helper,initial_state=initial_state,output_layer=None)    
    outputs, last_state, last_sequence_lengths = tf.contrib.seq2seq.dynamic_decode(decoder=decoder,output_time_major=False,impute_finished=True,maximum_iterations=10)

    
    ######################################################
    sess.run(tf.global_variables_initializer())
    print("\ninputs: ",inputs)
    inputs_ = sess.run(inputs) 
    print("\n",inputs_)    
    
    print("\noutputs: ",outputs)
    outputs_ = sess.run(outputs.rnn_output) 
    print("\n",outputs_) 

    print("\nlast state: ",last_state)
    last_state_ = sess.run(last_state) 
    print("\n",last_state_) 
"""
inputs:  Tensor("test/embedding_lookup:0", shape=(3, 6, 2), dtype=float32)

 [[[0. 1.]
  [6. 7.]
  [6. 7.]
  [4. 5.]
  [6. 7.]
  [4. 5.]]

 [[0. 1.]
  [6. 7.]
  [2. 3.]
  [4. 5.]
  [6. 7.]
  [2. 3.]]

 [[0. 1.]
  [2. 3.]
  [6. 7.]
  [4. 5.]
  [4. 5.]
  [2. 3.]]]

outputs:  BasicDecoderOutput(rnn_output=<tf.Tensor 'test/decoder/transpose:0' shape=(3, ?, 2) dtype=float32>, sample_id=<tf.Tensor 'test/decoder/transpose_1:0' shape=(3, ?) dtype=int32>)

 [[[0. 1.]
  [6. 7.]
  [6. 7.]
  [4. 5.]
  [6. 7.]
  [4. 5.]]

 [[0. 1.]
  [6. 7.]
  [2. 3.]
  [4. 5.]
  [6. 7.]
  [2. 3.]]

 [[0. 1.]
  [2. 3.]
  [6. 7.]
  [4. 5.]
  [4. 5.]
  [2. 3.]]]

last state:  Tensor("test/decoder/while/Exit_3:0", shape=(3, 2), dtype=float32)

 [[0.6 0.6]
 [0.6 0.6]
 [0.6 0.6]]
"""    
```

### [이제 BasicRNNCell을 직접 만들어보자]


