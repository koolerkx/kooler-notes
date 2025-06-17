# DirectX シェーダーのデータ取得メカニズム

スロット（Slot）は、GPUがバッファーからデータを読み取るための「読み取りポイント」。

固定の変数のように理解するとわかりやすい。

GPUシェーダーがバッファーから読み込むとき、スロットやセマンティックスでどの読みたいデータを探す。

## バッファーリソース

まず、バッファーには以下のように様々な種類がある：

- Constant Buffer 定数バッファー
- Vertex Buffer 頂点バッファー
- Texture SRV
- Sampler
- Shader Buffer シェーダーバッファー
  - VS, PS, GS 等々


## シェーダーファイルデータ受け取り方法

シェーダーはバッファーからデータを読み込む。

ではシェーダーはどのように、バッファーのデータを引数のように、データを読み込むか。方法は二つある：

1. セマンティックス（Semantics）による読み取り
    → 主にシェーダーバッファー（頂点入力など）からの読み込みに使われる

2. バッファースロット

   → 定数バッファーなど、セマンティックスを使わないバッファーで使用される

バッファーの種類ごとにスロットの種類と上限がある

```
リソース種別ごとに分類
┌────────────┬────────────┬────────────┬────────────┐
│  b0~b13    │  t0~t127   │  s0~s15    │  u0~uN     │
│ Constant   │ Texture    │ Sampler    │ UAV        │
│ Buffers    │ SRV        │ Sampler    │ RWBuffer   │
└────────────┴────────────┴────────────┴────────────┘
     ↑              ↑            ↑           ↑
  VSSetCB     PSSetSRV    PSSetSamplers  CSSetUAV
```


### Semantics セマンティックス

セマンティックスとは、HLSL内の変数とGPU上のメモリとを関連付けるための「決まった名前」。

```
+------------------+   +-------------+   +--------------------+
| VB (slot 0)      |-->| InputLayout |-->| Shader (HLSL)      |
| pos | norm | uv  |   | "POSITION"  |   | float3 : POSITION; |
+------------------+   | "NORMAL"    |   | float3 : NORMAL;   |
                       | "TEXCOORD0" |   | float2 : TEXCOORD; |
                       +-------------+   +--------------------+
```

GPUにデータを転送しただけでは、どのように読むかは分かりません。そのため：

「どこから」読むかは「バッファービュー（Buffer View）」、

「何として」読むかは「入力レイアウト（Input Layout）」で定義する必要があります。

ちなみにセマンティックスは、入力レイアウトを通じてデータを探します。 つまり、頂点処理段階（VS、GS）でのみ使われる情報です。

#### 入力レイアウト

1. 頂点データの構造体を定義：

   ```cpp
   struct Vertex {
       XMFLOAT3 position;
       XMFLOAT3 normal;
       XMFLOAT2 texcoord;
   };
   ```

   

2. 入力レイアウトの記述（D3D11 / D3D12 共通）：

   ```cpp
   D3D12_INPUT_ELEMENT_DESC layout[] =
   {
       // semanticName, semanticIndex, format, inputSlot, alignedByteOffset, inputSlotClass, instanceDataStepRate
       { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0,   D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
       { "NORMAL",   0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 12,  D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
       { "TEXCOORD", 0, DXGI_FORMAT_R32G32_FLOAT,    0, 24,  D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
   };
   ```

   

3. バッファーのパイプライン設定：

   ```cpp
   g_pContext->IASetVertexBuffers(0, 1, &g_pVertexBuffer, &stride, &offset);
   ```

4. シェーダー側でセマンティックスを用いた入力変数を定義：

   ```c
   struct VSInput
   {
       // 変数の名前はどうでもいい、値はSemanticsで定義されるから
       float3 position : POSITION;
       float3 normal   : NORMAL;
       float2 texcoord : TEXCOORD0;
   };
   
   VSOutput VSMain(VSInput input)
   {
       VSOutput output;
       return output;
   }
   ```

### スロットからの読み取り

セマンティックスを使わず、スロットから直接データを読み取る例（定数バッファーなど）：

1. C++側でデータをGPUに転送：

   ```cpp
   // cpp
   struct MatrixPack
   {
       XMFLOAT4X4 mtx;
   };
   
   MatrixPack matrix;
   XMStoreFloat4x4(&matrix.mtx, XMMatrixTranspose(...));
   
   UpdateSubresource(g_pVSConstantBuffer, ..., &matrix);
   
   // ここで、０からスロット一つは、行列が入っている、という意味
   VSSetConstantBuffers(0, 1, &g_pVSConstantBuffer);
   ```

2. HLSLシェーダー側で、どのスロットのデータを使うか指定：

    ```c
    // hlsl
    
    // `ConstantBuffer`の名前はどうでもいい、値はスロット(`register(b0)`)で定義するから
    cbuffer ConstantBuffer : register(b0)
    {
        float4x4 mtx;
    };
    
    // または、スロット順で自動割り当てする糖衣構文
    float4x4 mtx;
    ```

※糖衣構文はスロットの順番で割り当てられるので、宣言順に注意が必要です。

## 参考

全体像：

```
   ╭──────────────────── CPU ─────────────────────╮
   │                                              │
   │ Constant Buffer (C++)                        │
   │  ┌────────────┐   UpdateSubresource()        │
   │  │ WorldMatrix│ ─────────────────────────────┐
   │  └────────────┘                              │
   │                                              ▼
   │                               register(b0) ┌─────────────┐
   │                               ┌────────────▶ float4x4 mtx│
   │                               │            └─────────────┘
   │                               │
   │                               ▼
   │ ConstantBuffer slot 0 (b0) ┌───────────────────────────────┐
   │                            │ Used in Vertex Shader          │
   │                            └───────────────────────────────┘
   │
   │ Vertex Buffer (C++)                          ▼
   │  ┌────────────┐ IASetVertexBuffers(0, ...) ┌─────────────┐
   │  │ Position   │───────────────────────────▶ POSITION0    │
   │  ├────────────┤ IASetVertexBuffers(1, ...) ├─────────────┤
   │  │ Color      │───────────────────────────▶ COLOR0       │
   │  └────────────┘                             └─────────────┘
   │                                               VS_INPUT
   ╰──────────────────────────────────────────────────────────╯
```

