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
<div>$$ f(l,v)=/frac{c_{diff} } {/pi} $$</div>
$c_{diff}$是albedo

### Microfacet Specular BRDF

Cook-Torrance microfacet specular shading model:
<div>$$ f(l,v)=/frac{D(h)F(v,h)G(l,v,h)} {4(n·l)(n·v)} $$</div>

我们从Disney的model开始，评估每一项与更有效的代替方案相比的重要性


#### Specular D
D项为法线分布项( normal distribution function (NDF))，Disney用的GGX/Trowbridge-Reitz，效果对得起开销，使用Blinn-Phong的额外开销相当小，而且由于较长的“尾部”产生的明显、自然的视觉效果吸引了艺术家。我们还采用了Disney的$α = Roughness^2$的重新参数化。
$$D(h)=/frac{/alpha^2} {/pi((n·h)^2)(/alpha^2-1)+1}^2$$

#### Specular G
G项，评估了很多之后选了Schlick model，但是$k=/alpha /2$
