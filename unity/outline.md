# 物体轮廓线渲染
前不久看到神海4中的效果,所以自己实现了一下,其中遇到一点问题,记录下来.

![](https://raw.githubusercontent.com/ZeroPunchMan/archive/main/img/uncharted4-outline.png)

## 实现方法
网上查找资料,画轮廓的方法基本都是把mesh放大一点,多渲染一遍.

比如Unity中的文本渲染Outline,也是类似这样实现的,不过没有放大而是偏移,额外渲染了4次,,把偏移量设置很大,可以看到如下效果.

![](https://raw.githubusercontent.com/ZeroPunchMan/archive/main/img/unity-text-outline.png)

这里的做法也是类似的,如果目标被遮挡,则渲染出轮廓线的贴图,然后后期blit到输出端.先看看效果图.

![](https://raw.githubusercontent.com/ZeroPunchMan/archive/main/img/outline-ok.gif)

首先要做的是判断遮挡,这里精度要求不高,用RayCast即可.

然后获取轮廓线贴图,用2个pass来获得,shader代码大致如下.

```
PASS  //PASS0 vs放大一点,ps输出为固定颜色
{
    v2f vert(appdata v)
    {
	    v2f o;
	    float3 norm = normalize(v.normal);
	    v.vertex.xyz += v.normal * 0.02;
	    o.vertex = UnityObjectToClipPos(v.vertex);
	    return o;
    }

    fixed4 frag(v2f i) : SV_Target
    {
	    fixed4 col = fixed4(1,0,1,1); //紫色
	    return col;
    }
}

PASS //PASS1 vs正常,ps输出颜色为透明
{
    v2f vert(appdata v)
	{
		v2f o;
		o.vertex = UnityObjectToClipPos(v.vertex);
		return o;
	}

	fixed4 frag(v2f i) : SV_Target
	{
		fixed4 col = fixed4(0,0,0,0); //透明
		return col;
	}
}
```

得到了这样的贴图.

![](https://raw.githubusercontent.com/ZeroPunchMan/archive/main/img/unity-outline.png)

最后对于被遮挡的物体,把贴图blit到输出.

## 遇到的问题
最初是直接用main camera来渲染轮廓线贴图,但是出现了问题,如下.

![](https://raw.githubusercontent.com/ZeroPunchMan/archive/main/img/outline-error.gif)

最后使用的办法是,在main camera之下附加一个camera,并且disable这个camera.用这个camera去渲染轮廓线,就没有这个问题了.
