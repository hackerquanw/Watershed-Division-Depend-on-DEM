# Watershed-Division-Depend-on-DEM
# 基于DEM生成流域
# Running environment: Arcmap
# 编写背景
- 从数字高程模型（DEM）中提取流域在很多行业，例如环境保护、水文水利、农业等行业应用很广泛，也是最基础的应用之一。目前提取流域的方法主要有依赖ArcGIS中的空间分析工具来完成，需要使用14个空间分析工具来完成，费时费力，同时会产生许多中间过程文件。本插件使用Python语言，基于ArcGIS10进行二次开发，使用1个脚本来实现流域的提取，大大提高了工作效率。
# 设计原理
- 本插件原理如下图所示：
![image](https://user-images.githubusercontent.com/44941550/167165332-2ff7f0bf-58a9-486c-a0d2-362fa052a2dc.png)


# 设计原理（Script_basinFill脚本）
- （1）flow direction：流向
输入：dem
结果：flowdir_tif1。
获取目标区DEM后直接使用flow direction进行处理
- 原理：
获取表面的水文特征的关键之一是能够确定从栅格中的每个像元流出的方向。这可通过流向（flow direction）工具来完成。该工具把表面作为输入，然后输出一个显示从每个像元流出方向的栅格。
存在八个有效的输出方向，分别与流量可以流入的八个相邻像元相关。该方法通常被称为八方向 (D8) 流向建模，并且遵循在 Jenson 和 Domingue (1988) 中介绍的方法。
D8 流向法可对每个像元到其最陡下坡邻域的流向进行建模。
以 D8 流向类型运行的流向工具的输出是值范围介于 1 到 255 之间的整型栅格。从中心出发的各个方向值为：

![image](https://user-images.githubusercontent.com/44941550/167165381-f1d82bf8-1658-49cd-8471-848dfcb1e715.png)

- 例如，如果最陡下降方向位于当前处理像元的左侧，则将该处理像元的流向编码将为 16。
- 下图是一个示例：
 ![image](https://user-images.githubusercontent.com/44941550/167165422-e29b3e80-93b3-4d6f-85cf-b37d170ef749.png)

- 计算流向
流向由来自每个像元的最陡下降方向或最大下降方向确定。流向计算如下：

maximum_drop = change_in_z-value / distance * 100
计算像元中心之间的距离。因此，如果像元大小为 1，则两个正交像元之间的距离为 1，两个对角线像元之间的距离为 1.414（2 的平方根）。如果多个像元的最大下降方向都相同，则会扩大相邻像元范围，直到找到最陡下降方向为止。

找到最陡下降方向后，将使用表示该方向的值对输出像元进行编码。

如果所有相邻像元都比待处理像元高，则会将该待处理像元视为噪点并使用其相邻像元的最低值进行填充，该待处理像元具有朝向本身的流向。但是，如果单像元汇点位于栅格的实际边缘附近或至少具有一个 NoData 像元作为相邻像元，则会由于邻域信息不足而不能对其进行填充。要将某个像元视为真实单像元汇点，必须存在所有邻域信息。

如果两个像元彼此流入，则它们都是汇点，且具有未定义的流向。通过数字高程模型 (DEM) 获取流向的这一方法在 Jenson 和 Domingue (1988) 中进行了介绍。

可以使用汇（Sink）工具识别成为汇点的像元。要获取沿表面的流向的精确表示，应在使用流向栅格之前填充汇点。
Greenlee, D. D. 1987. "Raster and Vector Processing for Scanned Linework." Photogrammetric Engineering and Remote Sensing 53 (10): 1383–1387.

Jenson, S. K., and J. O. Domingue. 1988. "Extracting Topographic Structure from Digital Elevation Data for Geographic Information System Analysis." Photogrammetric Engineering and Remote Sensing 54 (11): 1593-1600.

- （2）sink
输入：flowdir_tif1
结果：Sink_watershed
虽然自然界地形中存在坑很正常，保证模拟水流能够按照流域的方向顺利流到出水口，我们需要把坑填上。
- （3）watershed
输入：
    input flow direction raster:flowdir_tif1
    input raster or feature pour point data:Sink_watershed
结果：sinkshed
这一步是通过watershed工具推算出哪些区域的水会流到这些坑里
- （4）zonal statistics
输入：
input raster or feature zone data: sinkshed
zone field: Value
Input value raster: dem
statistics type: minimum
输出：sink_min
通过区域统计得到“坑”的流域内高程的最低值
- （5）zonal fill
输入：
input zone raster:sinkshed
input weight raster:dem
输出：
sink_max
使用区域填充的结果是沿着“坑”流域的边界，也就是分水岭，把整个流域填平。这样就获得了“坑”流域的最高高程
- （6）raster calculator
输入：
sink_max - sink_min
输出：
sink_depth
大值减小值得到“坑”的最大深度

![image](https://user-images.githubusercontent.com/44941550/167165500-d43ff8ae-058b-4fe0-a78e-b4add0d40a01.png)


- （7）fill
输入：
input surface raster: dem
Z limit: sink_depth(max)
输出：Fill_sink_de1(Fill_tif1)
Z limit中要填sink_depth中的上限值，参考第6步的结果，就是143。在该脚本中，我们在上一步结束后自动获取该参数并将其传入fill工具，用下面的代码获取最大值
zlimit = arcpy.GetRasterProperties_management(sink_depth, "MAXIMUM")。
- （8）flow direction
输入：Fill_sink_de1(Fill_tif1)
输出：FlowDir_fill1
- （9）flow accumulation
输入：FlowDir_Fill1
输出：FlowAcc2
系统开始在无瑕疵的dem上下雨，然后计算流经每个栅格的水流量。
- （10）raster calculator(2)
计算公式：con(FlowAcc2>X,1)
输出：stream_r
公式中的X是要根据所在研究区，计划要提取的流域大小等因素确定的。过小会造成提取的河流过密，生成的流域细碎。如果过大会造成流域偏大，研究细节也就被埋没了。在这里，一张ASTGTM的影像提取建议取值2000.
- （11）stream link
输入：
input stream raster:stream_r
input flow direction raster:FlowDir_Fill1
输出：stream_link
通过stream link可以把栅格的值按照河流分别赋值，同一河段的栅格值都一样。方便后续转为矢量线。
- （12）stream to feature
输入：
input stream raster:stream_link
input flow direction raster:FlowDir_Fill1
输出:river_fea
这一步同样可以用raster to polyline完成，效果就是把栅格的河流转化为矢量的线。但据esri官方讲，stream to feature因为考虑的流向的原因，转化效果更好一些。
- （13）watershed(2)
输入：
input flow direction raster: FlowDir_Fill1
input raster or feature pour point data: stream_link
输出：Watersh_Flow1
- （14）Raster to Polygon
将Watersh_Flow1转化为矢量图，输出结果：RasterT_Watersh1以便接下来根据个人需要选择需要的子流域。
# 操作步骤
- 运行脚本 

![image](https://user-images.githubusercontent.com/44941550/167165538-b4537709-505e-44a1-8958-46518638f1d5.png)


Input DEM
- 输入地形图（推荐ASTGTM V2版本的TIF格式地形图）。

 
**X** 
X是汇流累积量阈值，要根据所在研究区，计划要提取的流域大小等因素确定的。过小会造成提取的河流过密，生成的流域细碎。如果过大会造成流域偏大，研究细节也就被埋没了。在这里，一张ASTGTM的影像提取建议取值2000.

**river** 

输出的水系矢量线图层

**watershed** 

输出的子流域矢量多边形图层
最终的运行结果如下图所示

![image](https://user-images.githubusercontent.com/44941550/167165675-f5f3d52c-49a9-4ce8-a8fa-37c484d2eb4d.png)


- 生成了矢量的线图层和面图层，输出的区域完全是输入的DEM数据的区域，这一点和SWAT模型不同，SWAT模型必须设定一个流域出口，我们的软件不需要设定这个流域出口，SWAT模型输出来的是一条河流的流域边界，我们这个工具输出的结果可以自由选择，他完全是子流域的数据集，至于由哪些子流域组成一个大流域，可以自己选择，那么选择的时候判断依据是看河网分支来判断，也可以自行自由组合各种子流域，这个工具比SWAT的河网生成工具更加灵活。如下图所示：

![image](https://user-images.githubusercontent.com/44941550/167165722-4c4cdb56-b055-4806-a15b-c7e01ac28e77.png)


- 最后，如果大家还需要更深入的了解该工具运算结果的使用，如何按照自己的需求，利用该工具所获得的子流域和河网判断目标湖泊或者河流流域界线，可以访问我的网易云课堂看教学视频（第二章的课时6和7）
https://study.163.com/course/introduction/1209321814.htm
