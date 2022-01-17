# 基于集成学习的城市房价分析及预测
## 一、课题意义及目的
课题意义：
        随着经济发展与城镇化进程的不断推进，房产交易量日益增加，房价评估作为房产交易的重要环节受到广泛重视，研究如何有效地评估房屋房价对售房者与购房者均有重要意义。

课题目的：基于集成学习类回归算法，搭建广州二手房房价分析及预测系统
所做工作：
        1.数据采集
        2.特征工程
        3.模型构建
        4.模型部署

## 二、数据采集（分布式总爬虫+每日定时爬虫）
（1）Scrapy-Redis框架简介	
        Scrapy-Redis框架采用主从结构，其设置了一个主服务器与多个子服务器，主服务器主要管理本地的Redis数据库进行下载任务的分发（也能进行爬虫），子服务器则进行爬虫解析网页并提取数据，转存至主服务器中的Redis数据库。

（2）爬虫模块所做工作	
        1.Xpath进行静态网页解析，提取与房价相关的有效特征。
        2.Selenuim进行动态网页解析，提取与房价相关的有效特征。
        3.对爬取的url进行去重处理。

（3）下载中间件设置（防屏蔽措施）	
        1.模拟不同浏览器标识（构建user-agent池，进行随机抽取）
        2.禁用cookie
        3.构建ip代理池（免费ip的效果较差）
        4.爬虫异常状态判断处理

（4）数据存储	
        Scrapy-Redis框架所爬取的数据默认存储至主服务器中的Redis数据库中，为了持久化存储帮助建模使用，将数据转存至Mysql数据库中。

（5）爬虫效率改进	
        1.框架默认线程数设置（默认为16，根据多次测试进行调整）
        2.多进程设置（使用multiprocessing库开启多个爬虫）
        3.分布式爬取（配置多台子机环境，后连接主机服务器实现多机爬取）

（6）每日定时爬虫
        链家二手房网每日会有房源数据更新，为添加这些新数据于数据库中，使用分布式总爬虫效率较低，因此设计针对每日更新数据的新爬虫脚本，基于linux的crontab进行每日定时运行，从而实现每日数据更新。

![image](https://user-images.githubusercontent.com/59312863/149720810-af997d5b-438b-4aef-b9ea-66fa8ddcbab9.png)

## 三、特征工程
（1）各特征转化为数值型特征	
        对各采集特征进行分析，转化为有效数值特征，常用策略如下：
        1.数据清洗（去除文本、数据提取）
        2.特征编码（对于某些类别型特征，进行合适地编码处理，如标签编码，独热编码）
        3.衍生特征（对于复杂且蕴含着房价相关信息的特征，常在原特征中依靠房产相关知识进行特征分割与特征组合，衍生新的有效数值特征）
（2）异常检测
        对于检测到的异常值，统一以空值替换。
        1.负值检测
        2.与实际认知不相符特征的特征检测
        3.IQR箱型图检测

（3）缺失值填充	
        1.部分特征根据房屋常识结合其余完整特征进行缺失值填充：
        如套内面积、公摊面积：根据套内面积+公摊面积=建筑面积的相互关系填充
        2.其余无法根据常识填充的缺失值，则先根据市辖区进行条件均值与条件众数填充，后再根据全数据集均值与众数填充。

（4）特征变换
        对于偏度高于0.75的连续型特征，进行Box-cox变换，使特征数据服从正态分布，以解决某些模型中需要数据服从正态分布的假设。
        
（5）特征选择	
        特征选择能一定程度上降低数据复杂度，减少过拟合
        使用斯皮尔曼相关系数法进行特征过滤，对相关性过低的特征给予剔除。
        
（6）特征缩放
        使用z-score标准化将各特征数值缩放至0附近，减少各特征之间的量纲差异。
## 四、模型构建
（1）确定评估指标：可决系数R方
（2）划分训练、验证与测试集：
        按6:4比例分割训练集与测试集，后各基于训练集进行五折交叉验证训练模型，最终以测试集的评估得分为模型评价标准。
（3）模型选择与参数调整：（对于各集成类算法的常用算法，选择主要的参数与合适的参数调整范围，进行网格搜索调参，从而实现模型的自动更新）
        1.Bagging类算法：随机森林
        2.Boosting类算法：XGBoost回归
        3.Stacking类算法：先从十种数据挖掘比赛常用的成熟回归算法中选择五个表现较强的学习器（按默认参数进行交叉验证选取），对这五个强学习器设计调参策略进行网格搜索调参作为第一层学习器，第二层则使用较简单的学习器KernelRidge模型以减少过拟合。

## 五、模型部署
（1）网站设计
        基于html与css进行静态网页布局设置，部分模块使用ajax进行渲染，设计广州二手房房价分析及预测系统，包括以下三个模块：
        1.项目简介（作为网站首页）：简单介绍本项目的主要内容、使用工具与结果，并提供建模所使用的最新完整数据集下载。
        2.可视化模块：利用echart对近7天的链家二手房数据概况进行可视化展示。
        3.房价评估模块：用户填写相应特征即可进行房价评估，包含如下设计
        ·表单填写：依据最终建模特征集进行填写表单设计
        ·空值处理：若输入值为空，则依旧依据特征工程里的缺失值处理方式进行处理
        ·输入判断：不可为负且不可过大
        ·特征填写帮助说明：各特征数值的填写说明，帮助用户理解如何输入
        ·评估结果：对预测结果进行逆标准化与逆正态化展示于页面

（2）模型部署
        使用Flask轻量框架对所设计的html与css进行网站搭建，将各模型打包成pkl文件，于Flask中实现前后端对接，从而实现模型应用。
        
![image](https://user-images.githubusercontent.com/59312863/149793590-e9b78660-25a9-4c58-aa58-eaa034f307f7.png)

![image](https://user-images.githubusercontent.com/59312863/149793608-0ebd74dd-bd99-4a13-bf0c-57bdabb1f299.png)

![image](https://user-images.githubusercontent.com/59312863/149793627-33601257-d62b-4fb9-9ee6-84e00bb013b9.png)

## 六、总结
优点：
        1.基于Scrapy-Redis框架的多进程+多线程+分布式爬虫，大幅度提高了爬取效率
        2.每日定时爬虫、特征工程脚本、网格搜索调参等一系列自动化操作，便于该房价分析及预测系统自动运作
        3.房价分析及预测系统的搭建，能让用户下载完整数据集、查看近七日房价房源数据、根据房屋特征评估房价，提高用户了解房价的便捷性。
缺点：
        1.当前系统依旧有许多未顾及用户需求的缺陷，还需进一步调整系统的便捷性与有效性。
        2.对于房价的预测模型构建中，使用的均为当前成熟的回归模型无作创新改进等，且调参策略的依据较模糊，为多次测试与各方面资料所设定。
        3.特征工程中还可进行更细致的处理，提高数据质量，如使用不同的特征选择方法等。
