---
title: Real Shading in Unreal Engine 4
date: 2024-07-10 09:58:20
tags: CG, paper
mathjax: true
---

# Real Shading in Unreal Engine 4

by Brian Karis, Epic Games

虽然是13年的paper了，但是也值得读一下

## Shading Model

### Diffues BRDF

Burley和Lambertian的diffuse model只有微小的差异，其他更复杂的model很难有效的使用IBL或者球谐光照
Lambertian Diffuse :
<div>$$ f(l,v)=\frac{c_{diff} } {\pi} $$</div>
$c_{diff}$是albedo

### Microfacet Specular BRDF

Cook-Torrance microfacet specular shading model:
<div>$$ f(l,v)=\frac{D(h)F(v,h)G(l,v,h)} {4(n·l)(n·v)} $$</div>

我们从Disney的model开始，评估每一项与更有效的代替方案相比的重要性


#### Specular D
D项为法线分布项( normal distribution function (NDF))，Disney用的GGX/Trowbridge-Reitz，效果对得起开销，使用Blinn-Phong的额外开销相当小，而且由于较长的“尾部”产生的明显、自然的视觉效果吸引了艺术家。我们还采用了Disney的$α = Roughness^2$的重新参数化。
$$D(h)=\frac{\alpha^2} {\pi((n·h)^2)(\alpha^2-1)+1}^2$$

#### Specular G
G项，几何阴影遮蔽项，评估了很多之后选了Schlick model，不过让$k=/alpha /2$，以更好地适配GGX模型中的Smith模型（一种用于计算微表面几何对遮挡效应的影响的数学模型）
通过修改k，当$\alpha=1$的时候，Schlick model和Smith完全匹配，在[0-1]的范围内，非常接近
我们依然使用Disney's modification来减少 "hotness"（过于光滑或反射过强的现象），在平方之前使用$\frac{Roughness+1}{2}$来remapping粗糙度，很重要的一点是，这个操作只用来分析光源，如果应用到IBL里面，在glancing angles(光线与表面接触的角度非常接近于平行)的结果会很黑
<div>$$ k=\frac{(Roughness+1)^2}{8} $$</div>
<div>$$ G_1(v)=\frac{n·v}{(n·v)(1-k)+k} $$</div>
<div>$$ G(l,v,h)=G_1(l)G_1(v) $$</div>

#### Specular F
对于Fresnel，我们用最典型的Schlick's approximation，但是做了一个小修改：我们用球面高斯(Spherical Gaussian)近似来替代power。它的计算效率略高并且没啥差异（imperceptible）公式是：
<div> F(v·h)=F_0+(1-F_0)2^{(-5.55473(v·n)-6.98316)(v·h)} </div>
其中$F_0$是在法线入射时的镜面反射率。（与法线平行）

## IBL
要将这种着色模型与IBL结合使用，需要解决radiance积分问题，通常使用重要性采样进行处理。下面的方程描述了这种数值积分过程：
<div>$$∫_H Li(l)f(l, v)cosθ_ldl ≈\frac{1}{N}∑_{k=1}^N\frac{L_i(l_k)f(l-k, v) cosθ_l_k} {p(l_k, v)}$$</div>
