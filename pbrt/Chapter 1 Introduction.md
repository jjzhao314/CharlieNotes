一个简单的Ray Tracer需要有以下部分组成：
# camera
![](https://pbr-book.org/3ed-2018/Introduction/Film%20in%20front.svg)
图中的眼睛就是一个摄像机的抽象。可见物体在眼睛与“胶片“的四个顶点所连接而成的四面体中（Film右侧，这部分被称为Viewing Volume）。从相机和图像中的一点连接一条线，这条线就是所谓的光线，这条光线将会和Viewing Volume中的物体作用并得到该图像点的颜色。遍历过所有的图像点后，就得到了一张图片。
# Ray-Object Intersection

在产生光线后，我们需要遍历场景（scene）中的每个物体，以此来判断哪些物体会和光线相交，并找到最近的交点。

$$
r(t) = o + td
$$
o代表光线原点，d代表方向向量，t是一个正参数。通过指定t即可找到光线上一个指定的点。当一个表面通过函数$F(x, y, z) = 0$定义时，可以将光线的表达式代入，得到t，即可获得交点。例如对于一个球体来说：

$$
x^2 + y^2 + z^2 - r^2 = 0
$$
那么将光线代入后得到：

$$
(o_x + td_x)^2 + (o_y + td_y)^2 + (o_z + td_z)^2 - r^2 = 0
$$
显然，有根代表有交点，最小的t给出了此交点。没有根的话就是二者不相交。当然，还需要知道交点处的材质、几何信息（法线）等才能进入后面的渲染。

但是显然，一个个遍历场景中物体是一个很慢很慢的办法。可以使用加速结构来加速。通常可以降低到$O(I\log N)$ ，I代表了像素数量，N代表场景中物体的数量（不过建立加速结构本身是$O(\log N)$的）。

# Light Distribution

找到一条光线和物体的交点的目的是：得到此点的颜色。那么就需要知道有多少光到达了这一点。因此，我们需要知道光源的照明分布。简单的点光源只需要知道位置即可，但这种光源在现实中并不存在。

对于一个点光源来说，假设其有一个功率$\Phi$，并且这个点光源被一个单位球包围，光源恰好在圆心位置，那么在单位球的一个单位面积上的功率就是$\frac{\Phi}{4\pi}$。假如算上光源和表面微小面积$dA$之间的夹角的话，那么单位面积的功率（irradiance）的微分$dE$可以写作：

$$
dE = \frac{\Phi \cos \theta}{4\pi r^2}
$$
![](https://pbr-book.org/3ed-2018/Introduction/Basic%20reflection%20setting.svg)

场景中有多个光源只需要将其相加即可。

# Visibility

上一节的光分布忽略了shadows。只有当光线和点的连线没有被挡住，才能为此点贡献光照。
![](https://pbr-book.org/3ed-2018/Introduction/Two%20lights%20one%20blocker.svg)

当然这很容易检测。以p点为起点，方向朝着光源，发射一条shadow ray，将会得到一个t，可以通过比较这个t得到是否被遮挡。

# Surface Scattering

入射光在到达p点后，会被散射出去。我们关心朝着相机散射的光线及其能量，如图

![](https://pbr-book.org/3ed-2018/Introduction/Surface%20scattering%20geometry.svg)

场景中的物体会带有材质，而所谓的材质就是描述了给定入射光，如何产生出射光的一个函数，称之为BRDF（bidirectional reflectance distribution function）。p点的BRDF可以写为：

$$
f_r(p, \omega_o, \omega_i)
$$

那么可以得到伪代码：

```pesuodo
for each light:
	if light is not blocked:
		incident_light = light.L(point)
		amount_reflected = surface.BRDF(hit_point, caera_vector, light_vector)
		L += amount_reflected * incident_light
```

注意这里的L不同于之前的$dE$，$L$同样代表radiance，是另一个单位。

除BRDF外，还有描述透射的BTDF、更加复杂的多次散射的BSSDF，以及一个更通用的表示：BSDF。

# Indirect Light Transport

间接光，其实就是在发生散射后继续追踪这个光线的意思。也就是递归的调用程序。这样可以很好的渲染镜子这一类物体。到达一点的光可以通过渲染方程来计算：

$$
L_o(p, \omega_o) = L_e(p, \omega_o) + \int_{S^2}f(p, \omega_o, \omega_i)L_i(p, \omega_i)|cos\theta_i|d\omega_i
$$
其中$L_e(p, \omega_o)$代表p点朝着wo方向的自己发射的光，后面的积分代表光源对于此点的贡献。应该不难看懂。这个方程很难解，因为涉及到递归。Whitted算法通过忽略大部分的方向，只计算直接光和完美反射、折射方向的间接光来求解渲染方程。当然，程序中只要规定递归次数，就可以直接进行计算。只不过如何确定合理的递归次数是一个问题。同时也要权衡性能和图片质量的问题。

# Ray Propagation

目前为止，光的传播被假定为在真空中进行。但是事实上，这是不现实的。灰尘、烟等会导致光的能量损失。这些介质可能通过吸收或散射光线以衰减光。可以通过计算透射率来模拟。也有些介质会增加光，例如火焰。这些渲染起来比较麻烦，后面会说。

