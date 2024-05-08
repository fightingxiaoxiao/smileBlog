---
title: 在OpenFOAM中实现动网格与流场的信息交互
date: 2022-08-19T14:49:00.000Z
author: Xiaoxiao
category: CFD
summary:  风吹雪的OpenFOAM动网格实现
tags:
  - CFD
  - OpenFOAM
  - C++
img: https://smilemax-1300318971.cos.ap-chengdu.myqcloud.com/blogImage/SummerRender_1.PNG
mathjax: true
cover: true
---

## 物理模型简介
在风吹雪、风吹沙、尘埃物输运等物理现象中，随着地表的颗粒物被风吹起，或是颗粒因为动能不足而发生沉积，其堆积形态会随着时间的推移而发生变化：

<img src="https://smilemax-1300318971.cos.ap-chengdu.myqcloud.com/blogImage/2022-8/snowdrift.png" alt="风致雪漂形成的雪堆" style="zoom:40%;" />

下面就是一个用于描述这类现象的侵蚀/沉积模型[<sup>[1]</sup>](#refer-anchor-1)，作者认为边界处的剪切风速的大小和颗粒物浓度决定了颗粒床的侵蚀/沉积量，进而可以推算出底部边界的演化过程：

$$q_{\mathrm{dep}} = -\phi w_f \left(1-\frac{u_{*}^{2}}{u_{*t}^{2}}\right)$$

$$q_{\mathrm{ero}}=-A_\mathrm{ero} \left(u_{*}^{2}-u_{*t}^2\right)$$

## 可编程的动网格边界

但是如果想在OpenFOAM中直接再现这个模型是会遇到麻烦的，因为常见的OpenFOAM动网格模型都是“强迫型”的。也就是说，网格的移动是需要事先指定的，流场的改变取决于网格的变化。这显然与我们的期望不符。因为在上述问题中，网格的变化来源于流场，而流场也会因为网格的变动发生进一步的改变，这两者之间存在耦合关系。

当然对于可定制性很强的OpenFOAM来说，解决这个问题只需要植入一段在算例中动态编译的代码。首先我们注意到动网格的边界条件来自于`0/pointMotionU`或`0/pointDisplacement`文件（取决于你使用的动网格求解器是基于速度还是位移），因此直接使用`codedFixedValue`这个重量级选手即可达到实时调整网格的目的。通过可编程接口，我们可以在定义边界条件中通过访问`patch().lookupPatchField`获取当前边界位置的场。例如我们已在自定义的求解器中创建了一个体标量场`deltaH`用于计算和存放网格格心和边界面心的侵蚀/沉积高度（非零值仅出现在特定边界），则可以通过下列方式在`codedFixedValue`的代码段中进行访问：

```cpp
//仅访问当前边界的场
fvPatchField<scalar> deltaH
(
	patch().lookupPatchField<volScalarField, scalar>("deltaH")
);
```

## 对节点边界的特殊化处理

但是，无论`0/pointMotionU`还是`0/pointDisplacement`均属于`pointVectorField`，与之对应的边界类是[`Foam::pointPatch`](https://www.openfoam.com/documentation/guides/v2112/api/classFoam_1_1pointPatch.html)，其数据存储在格点上。而在有限体积法表述下的边界是[`Foam::fvPatch`](https://www.openfoam.com/documentation/guides/v2112/api/classFoam_1_1fvPatch.html)，其数据存储在面心。这就带来两个问题：
1. `deltaH`场不存在于`pointField`中，因此在定义动网格边界条件时，无法通过上述的`patch().lookupPatchField`进行查找。
2. 格点数据和面心数据显然是无法对齐的，需要进行插值。

第一个问题很好解决。`fvPatch`和`pointPatch`虽然存在差异，但它们的指向的patch编号是一致的，因此可以通过`this->db()`访问到全局的网格信息和场信息，然后提取对应编号的边界场。自此，我们获得了一个`deltaHp`场，它负责存储边界面心处的`deltaH`值。
```cpp
label patchIndex = patch().index(); // 获取patch编号

// 访问全局场
const volScalarField &deltaH
(
	this->db().objectRegistry::lookupObject<volScalarField>("deltaH");
);

const scalarField& deltaHp = deltaH.boundaryField()[patchIndex];
```

第二个问题稍复杂一些，因为从单纯的CFD计算角度出发，基于体心的有限体积法没有将数据插值到节点的必要。不过幸运的是，OpenFOAM提供了一个[`PrimitivePatchInterpolation`](https://www.openfoam.com/documentation/guides/v2112/api/classFoam_1_1PrimitivePatchInterpolation.html)类，其中包含了从面向点插值的方法`faceToPointInterpolate`。简单看下源码，插值过程主要分为两步，首先是在节点上遍历所有从属的面，计算节点到面的距离，以倒数的形式计入面对点的权重并进行归一化，之后遍历所有面，把带权重的值累加进节点即可。本质上就是一个线性插值。

```cpp
//PrimitivePatchInterpolation<Patch>::makeFaceToPointWeights
forAll(pointFaces, pointi)
{
	const labelList& curFaces = pointFaces[pointi];
	scalarList& pw = weights[pointi];
 	pw.setSize(curFaces.size());
	scalar sumw = 0.0;
	forAll(curFaces, facei)
	{
        pw[facei] =
            1.0/mag(faces[curFaces[facei]].centre(points) - points[pointi]);
        sumw += pw[facei];
	}
	forAll(curFaces, facei)
	{
        pw[facei] /= sumw;
	}
}

//PrimitivePatchInterpolation<Patch>::faceToPointInterpolate
forAll(result, facei)
{
 	const labelList& curPoints = localFaces[facei];

	forAll(curPoints, pointi)
  	{
		result[facei] += pf[curPoints[pointi]];
	}
	result[facei] /= curPoints.size();
}

```
原理搞明白了，接下来就只需要调用`primitivePatchInterpolation`就行了。`primitivePatchInterpolation`构造函数的形参是一个`PrimitivePatch`对象，我没有找到直接访问的方法。但是我们可以通过刚才获取的`deltaH`场反溯网格，即利用`deltaH.mesh().boundaryMesh()`获得`polyBoundaryMesh`，这是一个存储`polyPatch`的List，而`polyPatch`恰巧继承自`PrimitivePatch`。自此，整个插值过程得以完成，我们获得了一个名为`deltaHpp`的场，它对应边界上每个节点的`deltaH`值。
```cpp
// 实例化插值对象
primitivePatchInterpolation facePointInterp(deltaH.mesh().boundaryMesh()[patchIndex]);

tmp<scalarField> deltaHpp = facePointInterp.faceToPointInterpolate(deltaHp); // 面心向节点插值
```
最后，只需要将获得的高度变化场赋予边界的节点场即可。
```cpp
pointField pVec(deltaHpp().size());

forAll(pVec, i)
{
	pVec[i][0] = 0;
	pVec[i][1] = 0;
	pVec[i][2] = deltaHpp()[i];
}

(*this) == pVec;
```

相关的求解器代码和计算案例可见https://github.com/fightingxiaoxiao/driftScalarDyFoam ，我也使用这个思路实现了对屋面积雪和方块周边积雪的简单预测[<sup>[2]</sup>](#refer-anchor-2)，相关论文发表于https://doi.org/10.3389/feart.2022.822140 。

<img src="https://smilemax-1300318971.cos.ap-chengdu.myqcloud.com/blogImage/2022-8/output.gif" alt="方块周边的积雪演化" style="zoom:50%;" />

## 参考文献
<a class="target-fix" id="refer-anchor-1"></a>

[1] ZHOU X, KANG L, GU M, 等. Numerical simulation and wind tunnel test for redistribution of snow on a flat roof[J/OL]. Journal of Wind Engineering and Industrial Aerodynamics, 2016, 153: 92-105. https://doi.org/10.1016/j.jweia.2016.03.008.

<a class="target-fix" id="refer-anchor-2"></a> 

[2] CHEN X, YU Z. DriftScalarDyFoam: An OpenFOAM-Based Multistage Solver for Drifting Snow and Its Distribution Around Buildings[J/OL]. Frontiers in Earth Science, 2022, 10[2022-03-16]. https://www.frontiersin.org/article/10.3389/feart.2022.822140.