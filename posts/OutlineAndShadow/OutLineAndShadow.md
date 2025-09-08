---
title: 外框線和線段陰影
description: Sobel 算子 和 URP 光線
date: 2025-09-08
tags:
  - comic game
layout: layouts/post.njk
---
鑒於我的遊戲是以漫畫為主題，同時我又希望可以減少自己在建模上的loading，所以我決定做一個類似漫畫線稿的視覺風格。因此外框線和線段陰影便是其中最重要的東西。

## 外框線

原本我是打算利用老方法-額外放一個建模然後只渲染背面的方式來完成這個外框線，但是這就會遇到兩個問題。第一是這樣多少有點吃效能，畢竟面數需要乘以二(也有可能有其他好方法排除這個問題)。再來就是他的外框線會有點粗細不一，這跟漫畫的線段很明顯衝突了。

之後參考了[Soble-OutLine](https://www.vertexfragment.com/ramblings/unity-postprocessing-sobel-outline/)文章我發覺似乎可以利用screen space 來完成渲染，這樣第一效能不會因為模型的多寡而改變，效果也比較好，完美的解決了我的問題。

首先因為我適用 2022 版本的URP 渲染管道，利用Full Screen Pass Reader Feature 的情況下我只需要寫frag function 就好(我原本打算自己寫Render Feature 但是真的看不懂在幹嘛)。接著我需要算出depth和normal。

``` hlsl
float SobelDepth(float2 uv, float2 texelSize){
    float center = SampleSceneDepth(uv);
    float right  = SampleSceneDepth(uv + float2(texelSize.x, 0));
    float left   = SampleSceneDepth(uv - float2(texelSize.x, 0));
    float top    = SampleSceneDepth(uv + float2(0, texelSize.y));
    float bottom = SampleSceneDepth(uv - float2(0, texelSize.y));

    // 計算各方向的法線差異
    float diff = 0;   
    diff += abs(center - top);   
    diff -= abs(center - bottom);   
    diff += abs(center - right);   
    diff -= abs(center - left);   

    // 合併水平和垂直差異
    return diff;
}
float SobelNormal(float2 uv, float2 texelSize) {
    float3 center = SampleSceneNormals(uv);
    float3 right  = SampleSceneNormals(uv + float2(texelSize.x, 0));
    float3 left   = SampleSceneNormals(uv - float2(texelSize.x, 0));
    float3 up     = SampleSceneNormals(uv + float2(0, texelSize.y));
    float3 down   = SampleSceneNormals(uv - float2(0, texelSize.y));

    float gx = dot(right - center, center) - dot(left - center, center);
    float gy = dot(up - center, center) - dot(down - center, center);

    return sqrt(gx*gx + gy*gy);
}
```

外框線:
![alt text](..\..\img\OutlineAndShadow\image.png)
就可以得到這樣的結果

## 陰影

這裡我參考了[Moebius-Style](https://www.youtube.com/watch?v=jlKNOirh66E)這個影片來做我的陰影，首先先利用Shadow Map和法線與 光源方向的點積 判斷是不是在陰影裡
``` hlsl
 float isInShadow(float2 uv,float3 worldPos,float3 worldNormal ){
    // 計算 Lambert 光照（法線與光源方向的點積）
    Light mainLight = GetMainLight();
    float3 lightDir = normalize(mainLight.direction);
    float NdotL = dot(worldNormal, lightDir);
    float lambertTerm = saturate(NdotL);

    // 獲取Shadow Map
    float4 shadowPos = TransformWorldToShadowCoord(worldPos);
    half shadowAttenuation = MainLightRealtimeShadow(shadowPos);

    float lightingFactor = lambertTerm * shadowAttenuation;

    return lightingFactor;
}
```
最後如果是在陰影的畫就計算亮度接著利用亮度決定要用哪個陰影

> 這部分我還在考慮是否要用點陣來代替

但是事情沒有這麼簡單如果就這樣渲染在 lit 的模型上他在需要陰影的部分還是會留下原先引擎渲染的灰階陰影，同時我又不能直接從 asset 中關閉陰影，因為我的 shader 計算部分是建立在 shadow map 上，所以我需要自己寫一個不接收任何陰影，但是又會生成 shadow map 的shader。

這個比我想像的容易，我只需要複製原版的 simple lit 並保留 我需要用到的 pass 包括 DepthOnly,
DepthNormals 當然還有重要的 ShadowCaster，最後在 frag 裡面只渲染純色就可以了。

最後結果:
![alt text](..\..\img\OutlineAndShadow\image-1.png)

整體看起來挺不錯的但是過於複雜的模型會變得很奇怪。