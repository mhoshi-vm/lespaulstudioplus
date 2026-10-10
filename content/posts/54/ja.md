---
title: "CPU(E5 2697v4) に Qwen3.6:35b-a3b を動かすためのメモ"
date: "2026-09-14T19:00:00+09:00"
tags: ["PrivateAI", "LLM", "CPU", "AI", "Qwen"]
thumbnail: img_1.png
---

[前回](../52) の記事で GPU 向けのLLMをチューニングしましたが、そこで得た知見がCPU側で動かすLLMに役立った(?)のでそのメモです。
化石のようなCPUなので、スピードは全く期待していないですが、MoEモデルを使うことでそれまで悩まされていた正確性がかなり良くなった印象です。
<!--more-->

## 遅い！が正確に返答！

まずは、細かい説明より、結果を。
今回は ollama を使って起動しました。

```
export OLLAMA_CONTEXT_LENGTH=32768
export OLLAMA_KV_CACHE_TYPE=f16  
                                  
export OLLAMA_FLASH_ATTENTION=1   
export OLLAMA_KEEP_ALIVE=-1       
export OLLAMA_LOAD_TIMEOUT=0      
export OLLAMA_NUM_PARALLEL=1      

ollama serve &

# Pull once, then run with the reasoning-effort default from "Extra Parameters"
ollama pull qwen3.6:35b-a3b
```

GPUと違い、Token per second という指標は違う気がするので、画像のOCRをやらせてみました。


![img.gif](img.gif)

過去に 10B 以下パラメーターのモデルで試した時は、最初は応答が返ってきて驚きはしました。が、
まともなことをやらせると、非常に小さいコンテキストサイズで前後のやり取りが全く覚えられない、といった点や


使い始めたばかりですが、以下のような所感です。

- CPUが化石なので、もう少し早くすれば
- Claude Opus 5 などと比較すると、同じタスクでもだいぶ完了までに時間がかかる印象がある。
- 出来上がったコードは Opus 5 と比較しても（私レベルだと）遜色がない。
- A100 に、Qwen3.8:27b が乗ると、vRAM をほぼ使い尽くすため、その他何も乗る余地がない。
- vRAM から溢れないよう、256k のコンテキストウィンドウを 220k に絞っている。（もう少し攻められる気もする）
- 256k のコンテキストウィンドウが意外にも大きな弱点になりうる。
特に大規模なコードだと、最初の読み込み＋プラン策定だけで、半分近くのコンテキストを使い切ってしまうことがある。
コンテキストが限界に到達すると基本的に新しいタスクが受け付けられなくなる。
- 220K の max_model_len を設定してしまったせいで、フル長のリクエストの場合、基本的に 1 Concurrent Request ほどしか受け付けられない（実測では 225,280 トークンあたり 1.10x で、短いリクエストなら複数同時も可能）。
- 私レベルだと自由に使える Claude という印象で、単純に楽しい。

## A100 の特徴

私も GPU を利用した LLM は初めてですが、改めて各 GPU のキャラクター差が大きいことがわかりました。今回私が学んだ A100 の特徴です。

- データセンター向けモデルなので、vRAM のメモリ帯域が 1.6TB/s あります（80GB モデルだと 2.0TB/s）。
最新のハイエンドなコンシューマー向け GPU と比べても遜色ない（もしくはそれ以上の）帯域があり、これが Token/s に大きく寄与している。
- vRAM が 40GB あり、ミッドレンジの 20B ~ 30B パラメーターの LLM モデルを載せることができる。
- ただ、コンテキスト分の vRAM も必要であり、フル精度（bf16）のモデルではなく、量子化されたモデルを使って可能な限り、メモリの消費量をおさえることが望ましい。
- CPU 側のメモリにオフロードも可能だが、それを行うと極端にパフォーマンスが落ちるので慎重にやるべき。
- A100 の最大の弱点は「古い」こと。6年前のモデルなので、現代のモデルの量子化手法 FP8（H100/H200 モデル想定）、FP4、NVFP4（Blackwell モデル想定）などの量子化に対応できないため、メモリ圧縮の効率がわるい。
- 今でも高価（になってしまった）

## vLLM パラメーターの意味

以下のような意味をもちます。

- **cyankiwi/Qwen3.8-27B-AWQ-INT4** ：上にもあるよう、A100 だと、現代的な量子化がデフォルトでは選べないです。
（ネット上では動かした猛者もいるようですが・・・）[cyan.kiwi](https://cyan.kiwi/) が提供している、古い GPU に対して最適化されたモデルを使っています。
- **quantization compressed-tensors** : モデルは AWQ（Activation-aware Weight Quantization）の名前が入っているのですが、モデルカードによると [compressed tensors](https://pypi.org/project/compressed-tensors/) が有効になっていたようで、それを設定しました。
- **tensor-parallel-size** : 今回は GPU が1枚なので必然的に 1
- **gpu-memory-utilization** : これは起動時に `[gpu_worker.py:575] CUDA graph memory profiling is enabled (default since v0.21.0). The current --gpu-memory-utilization=0.9300 is equivalent to --gpu-memory-utilization=0.8999 without CUDA graph memory profiling. To maintain the same effective KV cache size as before, increase --gpu-memory-utilization to 0.9601. To disable, set VLLM_MEMORY_PROFILER_ESTIMATE_CUDAGRAPHS=0.`
というメッセージが vLLM から出てきたので、93%分のメモリを確保するために、この設定を入れました。もうすこし攻める余地はあると思う。
- **max-model-len**: `gpu-memory-utilization` を 93% に設定した結果、`[gpu_worker.py:560] Available KV cache memory: 15.45 GiB [kv_cache_utils.py:2177] GPU KV cache size: 246,735 tokens`という結果がかえってきた。
最終的には、この値まであげることができるのですが、一旦 220k で設定を止めました。なお、vLLM 起動時のログに「mamba page size」「qwen_gdn_attention_core」といった記述があり、これは qwen の [Gated DeltaNet](https://github.com/rasbt/LLMs-from-scratch/blob/main/ch04/08_deltanet/README.md)（線形アテンション）がハイブリッドで使われているためのようで、それによって KV Cache size がかなり圧縮されているようです。
- **kv-cache-dtype**: これは GPU 自体が対応しているかは関係なく、計算時のパラメーターなので、FP8 が設定可能なはずです。
FP16 より FP8 にすることで、メモリの使用量は半分にできる、というのが机上の考えです。ただ、上の max-model-len にあるよう、すでに KV Cache の圧縮がかなり効果がでているので、デフォルトの auto のままにしています。
今後コンテキストをさらに圧縮したい場合に検証します。
- **enable-auto-tool-choice** : Tool 実行をするためのパラメーターなので、ほぼ必須で入れています。
- **default-chat-template-kwargs '{"reasoning_effort": "low"}'** : Qwen のモデルはデフォルトで xhigh の理由づけが設定されており、
放っておくと、モデルが必要以上に長考を繰り返す可能性があるので、デフォルトを low にしておきます。
- **enable-prefix-caching** : これを起動した結果 `Warning: Prefix caching in Mamba cache 'align' mode is currently enabled. Its support for Mamba layers is experimental. Please report any issues you may observe.` と出ており、
Experimental ではあるものの、設定が有効になっているようで、Cache ヒット率も上がったので、有効にしました。

ちなみに、vLLM ではなく、Ollama でも最初は試していました。
ただし、Ollama の場合、以下のエラーに悩まされ、あまり細かいパフォーマンスチューニングができなかったです。

https://github.com/ollama/ollama/issues/17778

一般的にも、GPU 上で LLM を使い、より細かいチューニングをしたければ、vLLM を使用することが世の共通認識のようです。


## まとめ

A100 に最新のモデルをなかなかのスピードで動かせて嬉しいです。
