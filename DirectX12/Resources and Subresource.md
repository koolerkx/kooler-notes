# Resource／Subresource の仕組み

頂点バッファーの初期化

1. `Map`/ `Unmap`
2. `UpdateSubresource` <- これ

> :information_source: UpdateSubresource とは、データをCPUからGPUに転送する
> VertexBuffer, IndexBuffer, ConstantBufferなどのGPUリソースを転送する

## Resources

GPUで確保したメモリ領域

- `ID3D11Texture2D`
- `ID3D11Buffer`
- `ID3D12Resource`

## Subresource (DX11)

リソースの一部分

1. バッファーは Subresourceの一つです
2. テクスチャの一部分

### Texture2D テクスチャ２D

1. Texture Array
2. MipMap Levels

```
テクスチャ二つとMip三つの場合

Texture2DArray (2 Layers, 3 Mips) — Resource
┌──────────────────────────────────────────┐
│ Resource ID3D11Texture2D / ID3D12Resource│
│                                          │
│  Layer 0                                 │
│  ┌──────────────┐                        │
│  │ Mip 0 (256x256) ──▶ Subresource 0     │
│  ├──────────────┤                        │
│  │ Mip 1 (128x128) ──▶ Subresource 1     │
│  ├──────────────┤                        │
│  │ Mip 2 (64x64)   ──▶ Subresource 2     │
│  └──────────────┘                        │
│                                          │
│  Layer 1                                 │
│  ┌──────────────┐                        │
│  │ Mip 0 (256x256) ──▶ Subresource 3     │
│  ├──────────────┤                        │
│  │ Mip 1 (128x128) ──▶ Subresource 4     │
│  ├──────────────┤                        │
│  │ Mip 2 (64x64)   ──▶ Subresource 5     │
│  └──────────────┘                        │
└──────────────────────────────────────────┘

```

> :information_source: Mipは違う解像度の画像、計算量減少とサンプラー最適化のための存在、LODに関係ある

> :information_source: Array Layerは同じテクスチャの中で、違うデータのこと
>
> 例えば、違うキャラが同じテクスチャを使うとき

> `D3D12CalcSubresource` でサブリソースの位置を計算する
>
> 裏では：SubresourceIndex = mip + (arraySlice * M) + (planeSlice * M * A)

## 例

### DirectX11 `UpdateSubresource`

```cpp
// Subresource index = 1 (mip = 1, arraySlice = 0)
UINT mipLevel = 1;
UINT arraySlice = 0;
UINT subresourceIndex = D3D11CalcSubresource(mipLevel, arraySlice, 3); // mipLevels = 3

g_pContext->UpdateSubresource(pTexture, subresourceIndex, nullptr, &mip1Data, rowPitch, 0);
```

### DirectX12 `UpdateSubresources`

```cpp
UINT subresourceIndex = D3D12CalcSubresource(
    mipLevel, arraySlice, planeSlice, mipLevels, arraySize
);

// Offsetとレイアウトを計算
D3D12_PLACED_SUBRESOURCE_FOOTPRINT layout;
device->GetCopyableFootprints(
    &texDesc, 0, 1, 0,
    &layout, nullptr, nullptr, &totalBytes
);

// ヒープにコピーして、アップロード待ち
UpdateSubresources(
    commandList,
    pDstResource,          // default heap resource (texture)
    pUploadResource,       // upload heap resource
    layout.Offset,         // offset in upload buffer
    subresourceIndex, 1,   // index and count
    &subresourceData       // 転送したデータ
);
```

