---
title: Transformer：实现一个简单的大语言模型
date: 2026-11-15 15:47:31
tags:
  - Transformer
  - 深度学习
---

### 前言

了解了 Transformer 的整体架构后，我们知道，在实现机器翻译时，我们可以将源序列输入编码器，然后将编码后的源序列与目标序列输入到解码器，解码器会尝试预测下一个词元。那我们可不可以使用它来生成文本呢？当然可以，其实无论是机器翻译、或者文本生成，它本质都是根据一个已有的文本，来预测下一个词元，基于这个想法，出现了 **Decoder-Only** 架构，它摒弃了 Encoder，只使用 Decoder 来完成文本生成任务，而这也是 GPT 模型采用的构架。

下面我们基于 Transformer 的 Decoder 来实现一个简单的“文本生成”语言模型。

<!-- more -->

> 《Python深度学习》笔记

### 文本数据准备

训练语言模型的第一步是加载原始文本。我们的数据集是一个 `.jsonl` 文件（数据来自 jingyaogong/minimind_dataset），每行是一个 JSON 对象，其中 `text` 字段存放了一条训练文本。

```python
# DataLoader.py
import json

def load_texts(path: str, max_lines: int) -> list[str]:
    texts = []
    with open(path, 'r', encoding='utf-8') as f:
        for i, line in enumerate(f):
            if i >= max_lines:
                break
            try:
                data = json.loads(line)
                text = data.get("text", "")
                if text:
                    texts.append(text)
            except json.JSONDecodeError:
                continue
    return texts
```

### 构建词表

词表决定了模型能"看懂"哪些文字。传统方案按空格切词后统计频次，但对中文效果很差，而且遇到训练集里没有的词就束手无策（Out-of-Vocabulary，OOV）。

这里我们使用 **Byte-Level BPE**（字节对编码），它是 GPT-2/GPT-4 等模型使用的分词方案，核心思路：

1. 把每个字符先映射成 UTF-8 字节（256 个基础 token，天然覆盖所有语言）；
2. 反复合并出现频率最高的相邻字节对，直到词表达到目标大小。

这样做的好处是：任何文字都能被表示；对中文、英文、数字、标点全部友好，无需额外预处理。

```python
# BPEVocab.py
from tokenizers import Tokenizer
from tokenizers.models import BPE
from tokenizers.trainers import BpeTrainer
from tokenizers.pre_tokenizers import ByteLevel
from tokenizers.decoders import ByteLevel as ByteLevelDecoder
from tokenizers.processors import TemplateProcessing

class BPEVocab:
    PAD_TOKEN = "<pad>"
    BOS_TOKEN = "<bos>"   # Beginning of Sequence
    EOS_TOKEN = "<eos>"   # End of Sequence

    def __init__(self, texts: list[str], vocab_size: int = 6000):
        tokenizer = Tokenizer(BPE(unk_token=None))
        tokenizer.pre_tokenizer = ByteLevel(add_prefix_space=False)
        tokenizer.decoder       = ByteLevelDecoder()

        trainer = BpeTrainer(
            vocab_size=vocab_size,
            special_tokens=[self.PAD_TOKEN, self.BOS_TOKEN, self.EOS_TOKEN],
        )
        tokenizer.train_from_iterator(texts, trainer=trainer)

        # 训练后配置 post-processor，编码时自动加 BOS/EOS
        bos_id = tokenizer.token_to_id(self.BOS_TOKEN)
        eos_id = tokenizer.token_to_id(self.EOS_TOKEN)
        tokenizer.post_processor = TemplateProcessing(
            single=f"{self.BOS_TOKEN} $A {self.EOS_TOKEN}",
            special_tokens=[
                (self.BOS_TOKEN, bos_id),
                (self.EOS_TOKEN, eos_id),
            ],
        )

        self.tokenizer = tokenizer
        self.PAD = tokenizer.token_to_id(self.PAD_TOKEN)
        self.BOS = bos_id
        self.EOS = eos_id
```

三个核心方法：

```python
    def encode(self, text: str) -> list[int]:
        """编码文本（含 BOS/EOS）→ token id 列表"""
        return self.tokenizer.encode(text).ids

    def decode(self, ids: list[int]) -> str:
        """解码 token id 列表 → 文本（自动过滤特殊 token）"""
        skip = {self.PAD, self.BOS, self.EOS}
        filtered = [i for i in ids if i not in skip]
        return self.tokenizer.decode(filtered)

    def save(self, path: str):
        self.tokenizer.save(path)

    @classmethod
    def load(cls, path: str) -> "BPEVocab":
        obj = cls.__new__(cls)
        obj.tokenizer = Tokenizer.from_file(path)
        obj.PAD = obj.tokenizer.token_to_id(cls.PAD_TOKEN)
        obj.BOS = obj.tokenizer.token_to_id(cls.BOS_TOKEN)
        obj.EOS = obj.tokenizer.token_to_id(cls.EOS_TOKEN)
        return obj
```

### Decoder 实现

#### 位置嵌入

Transformer 本身没有"顺序"的概念，所有位置是并行处理的。为了让模型知道每个 token 在序列中的位置，我们需要为每个位置加一个**位置嵌入**，位置嵌入在训练中得到，然后将它与词嵌入相加：

```
最终嵌入 = 词嵌入 + 位置嵌入
```

```python
# TransformerDecoder.py
class PositionalEmbedding(layers.Layer):
    def __init__(self, sequence_length, input_dim, output_dim, **kwargs):
        super().__init__(**kwargs)
        self.token_embeddings    = layers.Embedding(input_dim=input_dim,        output_dim=output_dim)
        self.position_embeddings = layers.Embedding(input_dim=sequence_length,  output_dim=output_dim)
        self.sequence_length = sequence_length
        self.input_dim  = input_dim
        self.output_dim = output_dim

    def call(self, inputs):
        length    = tf.shape(inputs)[-1]
        positions = tf.range(start=0, limit=length, delta=1)
        embedded_tokens    = self.token_embeddings(inputs)
        embedded_positions = self.position_embeddings(positions)
        return embedded_tokens + embedded_positions

    def compute_mask(self, inputs, mask=None):
        # 告诉后续层哪些位置是真实 token（非 PAD）
        return ops.not_equal(inputs, 0)
```

#### 因果注意力掩码

在语言模型训练时，我们希望位置 $i$ 的 token 只能"看到"它之前的 token（包括自身），不能看到未来的信息——否则模型直接把答案"抄"过来，根本学不到任何东西。

这通过**因果掩码**（保留下三角）实现：

```python
def get_causal_attention_mask(self, inputs):
    input_shape = tf.shape(inputs)
    batch_size, sequence_length = input_shape[0], input_shape[1]
    i = tf.range(sequence_length)[:, tf.newaxis]
    j = tf.range(sequence_length)
    mask = tf.cast(i >= j, dtype="int32")          # 下三角矩阵
    mask = tf.reshape(mask, (1, sequence_length, sequence_length))
    mult = tf.concat([tf.expand_dims(batch_size, -1),
                      tf.constant([1, 1], dtype=tf.int32)], axis=0)
    return tf.tile(mask, mult)                      # 扩展到 batch 维度
```

#### Transformer Decoder 层

标准 Decoder 有两个注意力子层：一个用于自注意力，一个用于交叉注意力（与 Encoder 交互）。在 **Decoder-Only** 架构中，我们没有 Encoder，所以把交叉注意力的 key/value 也换成自身，退化为第二次自注意力（实际上两次都作用于同一输入 `x`）：

```python
class TransformerDecoder(layers.Layer):
    def __init__(self, embed_dim, dense_dim, num_heads, **kwargs):
        super().__init__(**kwargs)
        self.attention_1   = layers.MultiHeadAttention(num_heads=num_heads, key_dim=embed_dim)
        self.attention_2   = layers.MultiHeadAttention(num_heads=num_heads, key_dim=embed_dim)
        self.dense_proj    = keras.Sequential([
            layers.Dense(dense_dim, activation="relu"),
            layers.Dense(embed_dim),
        ])
        self.layernorm_1   = layers.LayerNormalization()
        self.layernorm_2   = layers.LayerNormalization()
        self.layernorm_3   = layers.LayerNormalization()
        self.supports_masking = True

    def call(self, inputs, encoder_outputs, mask=None):
        causal_mask = self.get_causal_attention_mask(inputs)

        # padding mask 与 causal mask 取交集
        if mask is not None:
            padding_mask = tf.cast(mask[:, tf.newaxis, :], dtype="int32")
            padding_mask = tf.minimum(padding_mask, causal_mask)
        else:
            padding_mask = causal_mask

        # 第一个注意力：因果自注意力
        attn1 = self.attention_1(query=inputs, value=inputs, key=inputs,
                                 attention_mask=causal_mask)
        attn1 = self.layernorm_1(inputs + attn1)

        # 第二个注意力：Decoder-Only 中 key/value 也来自自身
        attn2 = self.attention_2(query=attn1, value=encoder_outputs, key=encoder_outputs,
                                 attention_mask=padding_mask)
        attn2 = self.layernorm_2(attn1 + attn2)

        # 前馈网络
        proj = self.dense_proj(attn2)
        return self.layernorm_3(attn2 + proj)
```

> 每个子层之后都有**残差连接**（`inputs + attn`）和**层归一化**，这是 Transformer 保持训练稳定的关键设计。

### 训练

#### 构造训练数据对

语言模型的训练目标是：**给定前 $n$ 个 token，预测第 $n+1$ 个 token**。

因此，对于一段编码后的序列 `[<bos>, t1, t2, ..., tn, <eos>]`，我们把它切成一对：

- **input**：`[<bos>, t1, t2, ..., tn]`（去掉最后一个）
- **target**：`[t1, t2, ..., tn, <eos>]`（去掉第一个）

这样模型在每个位置 $i$ 的输出，就是对应位置 $i+1$ 的真实 token。

```python
def encode_texts(texts, vocab, seq_len):
    pad_id = vocab.PAD
    inputs_list, targets_list = [], []

    for text in texts:
        ids = vocab.encode(text)          # 含 BOS … EOS

        if len(ids) > seq_len:
            ids = ids[:seq_len]
        if len(ids) < 2:
            continue

        inp = ids[:-1]                    # input:  去掉最后一个 token
        tgt = ids[1:]                     # target: 去掉第一个 token

        # 不足 seq_len-1 的部分用 PAD 补齐
        pad_len = (seq_len - 1) - len(inp)
        inp = inp + [pad_id] * pad_len
        tgt = tgt + [pad_id] * pad_len

        inputs_list.append(inp)
        targets_list.append(tgt)

    return np.array(inputs_list, dtype=np.int32), np.array(targets_list, dtype=np.int32)
```

#### 构建训练数据集

```python
def make_dataset(inputs, targets, batch_size, shuffle=True):
    # sparse_categorical_crossentropy 要求 target 有最后一个维度
    targets_expanded = targets[..., np.newaxis]
    dataset = tf.data.Dataset.from_tensor_slices((inputs, targets_expanded))
    if shuffle:
        dataset = dataset.shuffle(buffer_size=min(len(inputs), 10000))
    return dataset.batch(batch_size, drop_remainder=True).prefetch(tf.data.AUTOTUNE)
```

#### 搭建模型

把位置嵌入层和多层 Decoder 堆叠起来，顶层加一个 `Dense` 层输出每个位置上词表所有词的概率分布：

```python
def build_model(vocab_size, seq_len, embed_dim, dense_dim, num_heads, num_layers):
    inputs = keras.Input(shape=(None,), dtype="int64", name="token_ids")

    # 位置嵌入
    x = PositionalEmbedding(seq_len, vocab_size, embed_dim)(inputs)

    # 堆叠 Decoder 层（Decoder-Only：传入 x, x）
    for i in range(num_layers):
        x = TransformerDecoder(embed_dim, dense_dim, num_heads)(x, x)

    # 语言模型头：输出词表上的概率分布
    outputs = layers.Dense(vocab_size, activation="softmax")(x)

    return keras.Model(inputs, outputs)
```

#### 训练配置与启动

```python
def train():
    os.makedirs(SAVE_DIR, exist_ok=True)

    # 1. 加载文本
    print(f"[1/5] 加载数据：{DATA_PATH}")
    texts = load_texts(DATA_PATH, max_lines=MAX_LINES)
    print(f"      共加载 {len(texts)} 条文本")

    # 2. 训练 BPE 词表
    print("[2/5] 训练 BPE 分词器 ...")
    vocab = BPEVocab(texts, vocab_size=VOCAB_SIZE)
    vocab.save(TOKENIZER_PATH)
    print(f"      词表已保存至 {TOKENIZER_PATH}")
    actual_vocab_size = len(vocab)

    # 3. 编码 & 构造训练对
    print("[3/5] 编码文本，构造自回归训练对 ...")
    inputs_arr, targets_arr = encode_texts(texts, vocab, SEQUENCE_LENGTH)
    print(f"      训练样本数：{len(inputs_arr)}，序列长度：{inputs_arr.shape[1]}")

    # 4. 划分训练 / 验证集（90% / 10%）
    split = int(len(inputs_arr) * 0.9)
    train_ds = make_dataset(inputs_arr[:split],  targets_arr[:split],  BATCH_SIZE, shuffle=True)
    val_ds   = make_dataset(inputs_arr[split:],  targets_arr[split:],  BATCH_SIZE, shuffle=False)

    # 5. 构建模型
    print("[4/5] 构建模型 ...")
    model = build_model(
        vocab_size  = actual_vocab_size,
        seq_len     = SEQUENCE_LENGTH,
        embed_dim   = EMBED_DIM,
        dense_dim   = DENSE_DIM,
        num_heads   = NUM_HEADS,
        num_layers  = NUM_LAYERS,
    )
    model.summary()

    optimizer = keras.optimizers.Adam(learning_rate=LEARNING_RATE)
    model.compile(
        optimizer = optimizer,
        loss      = "sparse_categorical_crossentropy",
        metrics   = ["sparse_categorical_accuracy"],
    )

    # 回调：ModelCheckpoint + EarlyStopping
    ckpt_path = os.path.join(SAVE_DIR, "model_epoch{epoch:02d}_val{val_loss:.4f}.weights.h5")
    callbacks = [
        #保存验证集 loss 最低的权重
        keras.callbacks.ModelCheckpoint(
            filepath          = ckpt_path,
            save_weights_only = True,
            monitor           = "val_loss",
            save_best_only    = True,
            verbose           = 1,
        ),
        # 验证集 loss 连续 3 轮不下降则停止
        keras.callbacks.EarlyStopping(
            monitor   = "val_loss",
            patience  = 3,
            restore_best_weights = True,
            verbose   = 1,
        ),
        # 验证集 loss 连续 2 轮不下降则学习率减半
        keras.callbacks.ReduceLROnPlateau(
            monitor  = "val_loss",
            factor   = 0.5,
            patience = 2,
            min_lr   = 1e-5,
            verbose  = 1,
        ),
    ]

    # 6. 训练
    print("[5/5] 开始训练 ...")
    model.fit(
        train_ds,
        epochs          = EPOCHS,
        validation_data = val_ds,
        callbacks       = callbacks,
    )

    # 保存最终权重
    final_weights = os.path.join(SAVE_DIR, "model_final.weights.h5")
    model.save_weights(final_weights)
    print(f"\n训练完成！最终权重已保存至 {final_weights}")
```

损失函数选用 `sparse_categorical_crossentropy`：与 `categorical_crossentropy` 的区别在于，它直接接受整数形式的标签（token id），不需要提前做 one-hot 编码，对词表较大的场景更节省内存。

### 推理

#### 温度采样

模型输出的是词表上每个 token 的概率分布，最简单的做法是取概率最大的词（贪心解码），但这样生成的文本往往重复、单调。**温度采样**通过调整分布来控制随机性：

- $T < 1$：分布变尖锐，高概率词被进一步放大，输出更保守；
- $T = 1$：保持原始分布；
- $T > 1$：分布变平坦，低概率词也有机会被选中，输出更随机、更有创意。

```python
def sample_next(predictions, temperature=1.0):
    predictions = np.asarray(predictions).astype("float64")
    predictions = np.log(predictions + 1e-8) / temperature  # 避免 log(0)
    exp_preds   = np.exp(predictions - predictions.max())   # 减 max 防数值溢出
    predictions = exp_preds / exp_preds.sum()
    probas = np.random.multinomial(1, predictions, 1)
    return np.argmax(probas)
```

#### 自回归生成

推理时，我们把输入(prompt)编码为 token id 序列，然后循环地：

1. 把当前序列送入模型，得到每个位置的概率分布；
2. 取**最后一个有效位置**的概率分布，采样出下一个 token；
3. 把新 token 追加到序列末尾；
4. 重复，直到生成 `<eos>` 或达到最大长度。

```python
def generate(model, vocab, prompt, generate_length=100,
             model_input_length=127, temperature=1.0):

    prompt_ids    = vocab.encode(prompt)      # 含 BOS
    generated_ids = list(prompt_ids)
    prompt_length = len(prompt_ids)

    for i in range(generate_length):
        # 超出窗口时，只取最近的 model_input_length 个 token
        window    = generated_ids[-model_input_length:]
        pad_len   = model_input_length - len(window)
        padded    = window + [vocab.PAD] * pad_len
        inp       = np.array([padded], dtype=np.int32)

        predictions = model(inp, training=False)

        # 当前"下一个 token"对应的预测位置
        pred_pos         = min(prompt_length - 1 + i, model_input_length - 1)
        next_token_probs = predictions[0, pred_pos, :].numpy()

        # 屏蔽特殊 token，不让模型生成 PAD 或 BOS
        next_token_probs[vocab.PAD] = 0.0
        next_token_probs[vocab.BOS] = 0.0

        next_id = sample_next(next_token_probs, temperature)
        generated_ids.append(int(next_id))

        if int(next_id) == vocab.EOS:
            break

    return vocab.decode(generated_ids)
```

#### 文本生成

加载词表和模型权重，重建模型来生成文本：

```python

def inference():
    args = parse_args()

    # 1. 加载词表
    print(f"[1/3] 加载词表：{args.tokenizer}")
    if not os.path.exists(args.tokenizer):
        raise FileNotFoundError(f"词表不存在：{args.tokenizer}，请先运行 pre_train.py")
    vocab = BPEVocab.load(args.tokenizer)
    print(f"      词表大小：{len(vocab)}")

    # 2. 重建模型并加载权重
    print(f"[2/3] 加载权重：{args.weights}")
    if not os.path.exists(args.weights):
        raise FileNotFoundError(f"权重不存在：{args.weights}，请先运行 pre_train.py")
    model = build_model(len(vocab), SEQUENCE_LENGTH, EMBED_DIM,
                        DENSE_DIM, NUM_HEADS, NUM_LAYERS)
    model(np.zeros((1, 10), dtype=np.int32), training=False)  # 触发初始化
    model.load_weights(args.weights)
    print("      权重加载完成。\n")

    # 3. 推理
    model_input_length = SEQUENCE_LENGTH - 1

    def run_prompt(prompt: str):
        for temperature in args.temperatures:
            print(f"== Generating with temperature {temperature}")
            result = generate(model, vocab, prompt,
                              generate_length    = args.generate_length,
                              model_input_length = model_input_length,
                              temperature        = temperature)
            print(result)
        print()

    if args.prompt:
        run_prompt(args.prompt)
    else:
        # 交互模式
        print("交互模式（输入 quit 退出）")
        while True:
            try:
                prompt = input("\n>>> Prompt: ").strip()
            except (EOFError, KeyboardInterrupt):
                break
            if not prompt:
                continue
            if prompt.lower() in {"quit", "exit"}:
                break
            run_prompt(prompt)


```
