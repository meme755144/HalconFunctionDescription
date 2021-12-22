# Halcon二維仿射變換實例探究
[原文網址](https://www.cnblogs.com/xh6300/p/7442164.html)

二維仿射變換，顧名思義就是在二維平面內，對對象進行**平移**、**旋轉**、**縮放** 等變換的行為(當然還有其他的變換，這裡僅論述這三種最常見的)。

Halcon中進行仿射變換的常見步驟如下：
1. 通過`hom_mat2d_identity`算子創建一個 **初始化矩陣** (即[1.0, 0.0, 0.0, 0.0, 1.0, 0.0])

2. 在初始化矩陣的基礎上，使用下面或其他方法生成仿射變換矩陣；(這幾個算子可以疊加或者重復使用)
	- `hom_mat2d_translate`(平移)
	- `hom_mat2d_rotate`(旋轉)
	- `hom_mat2d_scale`(縮放)
	- 等等等

3. 根據生成的變換矩陣執行仿射變換，執行仿射變換的算子通常有：`affine_trans_image`、`affine_trans_region`、`affine_trans_contour_xld`，即不管對於圖像、區域、XLD都可以執行仿射變換。

下面用一個完整程序分別展示hom_mat2d_translate(平移)、hom_mat2d_rotate(旋轉)、hom_mat2d_scale(縮放)這三個算子的的具體功能。(特別要注意程序注釋部分)

hom_mat2d_translate( : : HomMat2D, **Tx**, **Ty** : HomMat2DTranslate)

hom_mat2d_rotate( : : HomMat2D, **Phi**, **Px**, **Py** : HomMat2DRotate)

hom_mat2d_scale( : : HomMat2D, **Sx**, **Sy**, **Px**, **Py** : HomMat2DScale)


程序所用圖片如下：

![](https://github.com/meme755144/HalconFunctionDescription/blob/master/hom_mat2d%20%E7%B3%BB%E5%88%97%E8%A7%A3%E9%87%8B/Image/OriginLine.jpg)
``` csharp
read_image (Image, 'hogn-1.jpg')
threshold (Image, Region, 0, 200)
opening_circle (Region, Region, 1.5)
connection (Region, ConnectedRegions)
select_shape_std (ConnectedRegions, SelectedRegions, 'max_area', 70)
//得到變換的中心點
area_center (SelectedRegions, Area, Row, Column)
dev_set_draw ('margin')

//hom_mat2d_translate中的兩個參數的意思是：Tx和Ty分別代表Row方向和Column方向的平移量
dev_display (Image)
disp_cross (3600, Row, Column, 10, 40)
hom_mat2d_identity (HomMat2DIdentity)
hom_mat2d_translate (HomMat2DIdentity,30, 150, HomMat2DTranslate)
affine_trans_region (Region, RegionAffineTrans, HomMat2DTranslate, 'nearest_neighbor')

//hom_mat2d_rotate中的三個參數的意思是：旋轉角度（逆時針為正，弧度制），旋轉中心的row和column值
dev_display (Image)
disp_cross (3600, Row, Column, 10, 40)
hom_mat2d_rotate (HomMat2DIdentity, rad(20), Row, Column, HomMat2DRotate)
affine_trans_region (Region, RegionAffineTrans, HomMat2DRotate, 'nearest_neighbor')

//hom_mat2d_scale中的四個參數的意思是：Sx和Sy分別代表Row方向和Column方向的縮放係數，縮放中心的row和column值
dev_display (Image)
disp_cross (3600, Row, Column, 10, 40)
hom_mat2d_scale (HomMat2DIdentity, 2.0, 1.05, Row, Column, HomMat2DScale)
affine_trans_region (Region, RegionAffineTrans, HomMat2DScale, 'nearest_neighbor')
```

效果分別如下：

![Move Line](https://github.com/meme755144/HalconFunctionDescription/blob/master/hom_mat2d%20%E7%B3%BB%E5%88%97%E8%A7%A3%E9%87%8B/Image/MoveLine.png)

![Spin Line](https://github.com/meme755144/HalconFunctionDescription/blob/master/hom_mat2d%20%E7%B3%BB%E5%88%97%E8%A7%A3%E9%87%8B/Image/SpinLine.png)

![Scale Line](https://github.com/meme755144/HalconFunctionDescription/blob/master/hom_mat2d%20%E7%B3%BB%E5%88%97%E8%A7%A3%E9%87%8B/Image/ScaleLine.png)


有時候，並不需要創建初始化矩陣也可以執行仿射變換，例如`vector_angle_to_rigid`算子就是如此。

vector_angle_to_rigid( : : **Row1**, **Column1**, **Angle1**, **Row2**, **Column2**, **Angle2** : HomMat2D)

該算子意思是：先將圖像旋轉，旋轉角度為(Angle2 - Angle1) (逆時針為正),`旋轉中心坐標是(Row1, Column1)`。再將原圖的點(Row1, Column1)一一對應移到點 (Row2, Column2)上，移動的row和column方向的位移分別是( Row2 - Row1)、( Column2 - Column1),

如果`Row1 = Row2, Column1 = Column2`，那麼就完整等價于旋轉變換。可以執行下面的程序感受一下：

```csharp
read_image (Image, 'hogn-1.jpg')
Row := 100
Column := 200
dev_display (Image)


for Index := 1 to 150 by 1
    vector_angle_to_rigid (Row, Column, 0, Row, Column, rad(10), HomMat2D)
    disp_cross (3600, 100, 200, 10, 40)
    affine_trans_image (Image, ImageAffinTrans, HomMat2D, 'nearest_neighbor', 'false')
    copy_image (ImageAffinTrans, Image)
endfor
```

可以將vector_angle_to_rigid理解為同時執行旋轉變換和平移變換。最難弄明白的是旋轉中心是什麼？下面的程序可以說明如果先旋轉后平移，那麼旋轉中心是(Row1, Column1)，而不是 (Row2, Column2)。(如果先平移后旋轉，那麼結論剛好相反，大家可以試試)

```csharp
read_image (Image, 'hogn-1.jpg')
Row1 := 100
Column1 := 100

Row2 := 100
Column2 := 200
dev_display (Image)
//用vector_angle_to_rigid實現縮放、平移
vector_angle_to_rigid (Row1, Column1, 0, Row2, Column2, rad(10), HomMat2D)
affine_trans_image (Image, ImageAffinTrans, HomMat2D, 'nearest_neighbor', 'false')

//分兩步依次執行縮放、平移
hom_mat2d_identity (HomMat2DIdentity)
hom_mat2d_rotate (HomMat2DIdentity, rad(10) - 0, Row1, Column1, HomMat2DRotate)
hom_mat2d_translate (HomMat2DRotate,Row2 - Row1, Column2 - Column1, HomMat2DTranslate)
//觀察圖像ImageAffinTrans和ImageAffinTrans_2能夠完全重合
affine_trans_image (Image, ImageAffinTrans_2, HomMat2DTranslate, 'nearest_neighbor', 'false')

disp_cross (3600, Row1, Column1, 10, 40)
```

vector_angle_to_rigid最常用到的場合一般是模板匹配之類的算法場合，通常用在find_shape_model等算子后面。


下面用一個例子說明一下仿射變換的綜合應用，**即當圖片旋轉90°時，想辦法變換Region使之能夠翻轉到對應的位置**。

![Origin Demo](https://github.com/meme755144/HalconFunctionDescription/blob/master/hom_mat2d%20%E7%B3%BB%E5%88%97%E8%A7%A3%E9%87%8B/Image/OriginDemo.png)

將圖片順時針翻轉90°的方法可以是：`rotate_image (image, ImageRotate, -90, 'constant')`。

但其實它不僅經過了旋轉變換、還進行了平移變換，最明顯的證據就是：**翻轉前后的圖像，他們的中心點坐標不一樣**。完整程序如下：

```csharp
read_image (image, 'C:/Users/happy xia/Desktop/dynPic.png')
binary_threshold (image, Region, 'max_separability', 'dark', UsedThreshold)
dev_set_draw ('margin')
connection (Region, ConnectedRegions)
select_shape_std (ConnectedRegions, SelectedReg, 'max_area', 70)
area_center (image, Area, Row, Column)

rotate_image (image, ImageRotate, -90, 'constant')
area_center (ImageRotate, Area2, Row2, Column2)

hom_mat2d_identity (HomMat2DIdentity)
hom_mat2d_rotate (HomMat2DIdentity, -rad(90), Row, Column, HomMat2DRotate)
hom_mat2d_translate (HomMat2DRotate,Row2 - Row, Column2 - Column, HomMat2DTranslate)

affine_trans_region (SelectedReg, RegionAffineTrans, HomMat2DTranslate, 'constant')
```

![Spin Demo](https://github.com/meme755144/HalconFunctionDescription/blob/master/hom_mat2d%20%E7%B3%BB%E5%88%97%E8%A7%A3%E9%87%8B/Image/SpinDemo.png)


該算法順利達到了目的——圖像翻轉以後，原先生成的Region也翻轉到了對應的位置。

注意：用rotate_image 算子旋轉圖像時，如果旋轉角度不是0°、90°、180°、270°等角度，那麼圖像其實只做了旋轉變換，而沒有進行平移變換。




