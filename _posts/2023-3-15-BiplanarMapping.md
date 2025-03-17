---

layout:     post

title:      "Biplanar Mapping"

subtitle:   ""

date:       2023-3-15 12:00:00

author:     "Wanghan"

header-img: "img/kuiba01-2017-10-29.jpg"

tags:

    - 纹理技术

---
# Biplanar Mapping

应用于一些没有uv的模型，比如没有进行展uv，或者不基于网格或多边形的3d模型，例如隐式sdf函数。

## Box/RoundCube/Triplanar mapping

### 理论

Triplanar Mapping，即三维映射，使用平面贴图沿 X、Y 和 Z 轴将纹理映射三次（因此是Tri-planar。然后根据朝向角度（法线）在三个采样结果之间混合，根据法线轴向的权重使用最小拉伸采样结果。理论上，这样永远不会有拉伸纹理或硬接缝，甚至不需要对网格进行UV映射。
![](/img/in-post/BiplanarMapping/image-20250315171227939.png)


为了实现三维映射，我们首先需要生成三个平面UV，为此，我们使用vs的世界空间位置。如果我们使用vs的XZ世界空间坐标作为UV坐标来采样，这样就会得到一个从Y轴投影的平面贴图，同理还可以得到使用ZY坐标的X投影，XY坐标的Z投影。

我们还需要一个混合权重来决定使用哪一个UV集。我们根据世界空间顶点的法线来进行。我们可以使用每个轴的绝对值，举个例子，这样如果表面法线指向正向或负向Y轴，我们可以混合更多来自 Y 投影平面的纹理样本。

### 缺点

三维映射的一个缺点在于，因为法线映射依赖于由UV表示的网格的切线，并且由于我们使用世界空间坐标来表示我们的UV，这样网格的切线就会摇摆不定。我们仍可以使用法线贴图来进行光照，只是不正确。

同样，三维映射与传统的UV映射相比，没那么有效率。需要执行三次贴图采样。当我们只是采样漫反射纹理时这还能接受，但如果我们同样映射高光贴图、细节贴图等，我们就不得不去关注使用的采样点的数量了。

### unity中的实现

```c
Shader "Unlit/Triplanar Mapping"
{
    Properties
    {
        _AlbedoMap ("Albedo Map", 2D) = "white" {}
        _TextureScale("Texture Scale", float) = 1
        _TriplanarBlendSharpness("Blend Sharpness", float)=1
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            struct vertexInput
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
                float3 normal : NORMAL;
            };

            struct vertexOutput
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
                float3 worldPos : TEXCOORD1;
                float3 worldNormal : TEXCOORD2;
            };

            sampler2D _AlbedoMap;
            float4 _AlbedoMap_ST;
            float _TextureScale;
            float _TriplanarBlendSharpness;

            vertexOutput vert (vertexInput i)
            {
                vertexOutput o;
                o.vertex = UnityObjectToClipPos(i.vertex);
                o.uv = TRANSFORM_TEX(i.uv, _AlbedoMap);
                o.worldPos = mul(unity_ObjectToWorld, i.vertex).xyz;
                o.worldNormal = UnityObjectToWorldNormal(i.normal);
                return o;
            }

            fixed4 frag (vertexOutput i) : SV_Target
            {
                // find our UVs for each axis based on world position of the fragment
                half2 yUV = i.worldPos.xz / _TextureScale;
                half2 xUV = i.worldPos.zy / _TextureScale;
                half2 zUV = i.worldPos.xy / _TextureScale;

                // do texture samples from our albedo mao with each of the 3 UV set's we've just made.
                half3 yDiff = tex2D(_AlbedoMap, yUV);
                half3 xDiff = tex2D(_AlbedoMap, xUV);
                half3 zDiff = tex2D(_AlbedoMap, zUV);

                // get the absolute value of the world normal.
                // put the blend weights to the power of BlendSharpness, the higher the value,
                // the sharpness the transition between the planar maps will be.
                half3 blendWeights = pow(abs(i.worldNormal), _TriplanarBlendSharpness);

                // divide our blend mask by the sum of it's components, this will make x+y+z = 1
                blendWeights = blendWeights / (blendWeights.x + blendWeights.y + blendWeights.z);

                // finally, blend together all three samples based on the blend mask.
                fixed4 col = fixed4(xDiff * blendWeights.x + yDiff * blendWeights.y + zDiff * blendWeights.z, 1.0);
                return col;
            }
            ENDCG
        }
    }
}
```

shadertoy: [https://www.shadertoy.com/view/MtsGWH](https://www.shadertoy.com/view/MtsGWH)

## Biplanar mapping

将triplaner mapping的三次贴图采样减少到两次。实现这一点的最简单方法是选择三个规范投影中最相关的两个，并使用它们，同时完全忽略第三个。

```glsl
// "p" point being textured
// "n" surface normal at "p"
// "k" controls the sharpness of the blending in the transitions areas
// "s" texture sampler
vec4 biplanar( sampler2D sam, in vec3 p, in vec3 n, in float k )
{
    // grab coord derivatives for texturing
    // ddx ddy 表示表面集合在x,y方向上的变化率。
    vec3 dpdx = dFdx(p);
    vec3 dpdy = dFdy(p);
    n = abs(n);

    // determine major axis (in x; yz are following axis)
    ivec3 ma = (n.x>n.y && n.x>n.z) ? ivec3(0,1,2) :
               (n.y>n.z)            ? ivec3(1,2,0) :
                                      ivec3(2,0,1) ;
    // determine minor axis (in x; yz are following axis)
    ivec3 mi = (n.x<n.y && n.x<n.z) ? ivec3(0,1,2) :
               (n.y<n.z)            ? ivec3(1,2,0) :
                                      ivec3(2,0,1) ;
    // determine median axis (in x;  yz are following axis)
    ivec3 me = ivec3(3) - mi - ma;
    
    // project+fetch
    vec4 x = textureGrad( sam, vec2(   p[ma.y],   p[ma.z]), 
                               vec2(dpdx[ma.y],dpdx[ma.z]), 
                               vec2(dpdy[ma.y],dpdy[ma.z]) );
    vec4 y = textureGrad( sam, vec2(   p[me.y],   p[me.z]), 
                               vec2(dpdx[me.y],dpdx[me.z]),
                               vec2(dpdy[me.y],dpdy[me.z]) );
    
    // blend factors
    vec2 w = vec2(n[ma.x],n[me.x]);
    // make local support
    w = clamp( (w-0.5773)/(1.0-0.5773), 0.0, 1.0 );
    // shape transition
    w = pow( w, vec2(k/8.0) );
    // blend and return
    return (x*w.x + y*w.y) / (w.x + w.y);
}
```

这个函数所做的最重要的事情是选择三个投影中的哪两个与我们正在纹理化的表面最对齐。 找到最对齐的一个，我称之为“长轴”，非常简单 - 只需将法线的 X、Y 和 Z 分量相互比较，哪个最大（绝对值）表示哪一个 三个投影 X、Y 或 Z 最平行于表面。 只需进行几次比较即可轻松完成。

```glsl
//(0,1,2)=>(x,y,z) (1,2,0)=>(y,z.x)  (2,0,1)=>(z,x,y)
ivec3 ma = (n.x>n.y && n.x>n.z) ? ivec3(0,1,2) :
               (n.y>n.z)            ? ivec3(1,2,0) :
                                      ivec3(2,0,1) ;
```

这些比较很可能编译为条件移动而不是实际分支。 我将最大投影的索引存储在“ma”变量的第一个组件中。 如果 max.x 为 0，则表示最大投影沿 X 轴，如果值为 1，则参考 Y 轴，如果值为 2，则表示 Z 轴。 “ma”变量的第二个和第三个组件按字典顺序存储不是主要的“其他两个”轴，这对于获得正确的平面纹理很重要。

找到短轴，或者说三个投影中最差的一个。 它存储在“mi”变量中。

```glsl
//(0,1,2)=>(x,y,z) (1,2,0)=>(y,z.x)  (2,0,1)=>(z,x,y)
ivec3 mi = (n.x<n.y && n.x<n.z) ? ivec3(0,1,2) :
               (n.y<n.z)            ? ivec3(1,2,0) :
                                      ivec3(2,0,1) ;
```

找到中轴，将它存储在变量“me”中，执行操作 me = 3-ma-mi 将为我们提供所需的第二好的轴。 这是因为无论长轴和短轴采用 {0,1,2} 中的哪个值，三个轴索引的相加将是 3，因为 0+1+2=3。 所以 3-ma-mi 将揭示哪个轴是中间轴。

```glsl
ivec3 me = ivec3(3) - mi - ma;
```

一旦确定了两个最佳投影轴并将其存储在“ma”和“me”的第一个分量中，以及它们在这些变量的第二个和第三个分量中的以下词典轴索引，剩下要做的就是我们 一直在三线性映射中完成，但只有两个投影而不是三个。 也就是说，执行两个纹理提取，然后计算它们颜色的线性组合。 所以让我们先做这些提取：

```glsl
vec4 x = texture( sam, vec2( p[ma.y], p[ma.z]) );
vec4 y = texture( sam, vec2( p[me.y], p[me.z]) );
```

只有上面的代码的话会产生伪影，因为选择主轴或中轴时会突然跳变，相邻的两个点会采样不同轴的平面uv，会影响纹理采样器中的mipmap，因此，可以通过在选择轴之前获取“p”的梯度来避免不连续性，然后将这些梯度的适当分量传递给纹理采样函数。

```glsl
vec3 dpdx = dFdx(p);
vec3 dpdy = dFdy(p);
	
...

vec4 x = textureGrad( sam, vec2(   p[ma.y],   p[ma.z]), 
                           vec2(dpdx[ma.y],dpdx[ma.z]), 
                           vec2(dpdy[ma.y],dpdy[ma.z]) );
vec4 y = textureGrad( sam, vec2(   p[me.y],   p[me.z]), 
                           vec2(dpdx[me.y],dpdx[me.z]),
                           vec2(dpdy[me.y],dpdy[me.z]) );
```

```glsl
w = clamp( (w-0.5773)/(1.0-0.5773), 0.0, 1.0 );
```

如果没有这行代码，会有一些（通常很小的）纹理不连续性，这些不连续性自然会发生在法线点位于八个 (±1,±1,±1) 方向之一的区域。发生这种情况是因为在某些时候，次要投影方向和中间投影方向将交换位置，并且一个投影将被另一个投影替换。在大多数纹理和大多数混合成形系数“l”的实践中，如果有的话，很难看到不连续性，但不连续性总是存在的。幸运的是，通过重新映射权重，将 1/sqrt(3), 0.5773 映射为零，很容易摆脱它。

![](/img/in-post/BiplanarMapping/image-20250315171139536.png)

现在，这种高度重新映射消除了不连续性，但显着缩小了混合区域，因此双平面映射器的外观更接近于具有“k”等于 8 的积极混合因子的三平面映射。因此，您会发现一个划分在我为双平面映射器建议的代码中到 8.0，因为这使得双平面纹理映射器成为三平面映射器的视觉上令人满意的替代品。

```glsl
w = pow( w, vec2(k/8.0) );
```

![](/img/in-post/BiplanarMapping/image-20250315171255124.png)

Triplanar,k=1

![](/img/in-post/BiplanarMapping/image-20250315171303999.png)

Biplanar,k=8,no remapping

![](/img/in-post/BiplanarMapping/image-20250315171320995.png)

Biplanar,k=1,remapping

![](/img/in-post/BiplanarMapping/image-20250315171331103.png)

Triplanar,k8,remapping

参考文章：[https://iquilezles.org/articles/biplanar/](https://iquilezles.org/articles/biplanar/)