# 编译安装+PMML模型文件
## 一、依赖环境
### 1、gcc 5.0+

https://blog.csdn.net/furzoom/article/details/53322510

（预计3h）

我装的是9.1.0，包太大无法上传GitHub，上边CSDN有下载链接

如果是环境比较新的虚拟机可以先装下4.8，yum -y install gcc-c++ ，可以装一下gcc需要的环境，后面用9.1.0代替旧版本就好，不然那些依赖自己装要好一会

安装好gcc可以先将虚拟机备份，cmake安装过程出问题可能会导致机子黑屏无法启动，再重新安装gcc耗时太久

### 2、cmake

我装的 version 3.17.0

tar -zxf cmake-3.17.0.tar.gz
cd ~/cmake-3.17.0/
./bootstrap
gmake && gmake install
./bootstrap可能会遇到bug：“/usr/lib/libstdc++.so.6: version `GLIBCXX_3.4.14' not found”

https://www.cnblogs.com/cthon/p/12722055.html

我用过的步骤：

（1） 查看 libstdc++.so 位置和版本
sudo find / -name "libstdc++.so*"  
（2） 查看当前的libstdc++.so.6的使用版本
ls -al /usr/lib64/libstdc++.so.6  
（3） 把新版本拷贝到系统目录下
cp /apps/cthon/lib64/libstdc++.so.6.0.26 /usr/lib64  
（4） 修改libstdc++.so.6的软连接并删除旧版本
sudo rm /usr/lib64/libstdc++.so.6  
sudo rm /usr/lib64/libstdc++.so.6.0.19  
（5） 建立新的软连接
sudo ln -s /usr/lib64/libstdc++.so.6.26 libstdc++.so.6  
（6） 检查结果
strings /usr/lib64/libstdc++.so.6 |grep GLIBCXX  

## 二、编译安装+建模demo
官网：https://xgboost.readthedocs.io/en/release_0.90/index.html

可参考文档：

http://deepspark.cn/2017/05/05/xgboost-install/

https://blog.csdn.net/u010306433/article/details/51403894

https://zhuanlan.zhihu.com/p/97192300

参考官网时右下角有版本切换，不同版本编译方式有区别，下载包后若要切换版本也需在官网切换版本安装该版本编译方式编译，否则可能会编译失败

### 1、下载

git clone --recursive https://github.com/dmlc/xgboost
切换版本

git checkout release_0.90 
自己想要哪个版本就切到哪个版本，spark的版本与xgboost版本有对应关系，要适合自己生产环境，官网这里https://xgboost.readthedocs.io/en/release_0.90/jvm/xgboost4j_spark_tutorial.html有讲版本对应：XGBoost4J-Spark now requires Apache Spark 2.4+，也即xgb0.90版本对应spark2.4+

更新子模块

 git submodule update --init --recursive
### 2、xgboost编译

mkdir build
cd build
cmake ..
make -j4
### 3、mvn打包
cd jvm-packages;  
mvn clean && mvn -Dmaven.test.skip=true -Dspark.version=2.4.3 package   
（mvn clean && mvn -Dmaven.test.skip=true -Dspark.version=2.3.2 package，版本与之前对应）  
### 4、本地mvn引入
cd /opt/soft/xgboost/  
cd jvm-packages/  
将jar引入maven中：  
mvn install:install-file -Dfile=/opt/soft/xgboost/jvm-packages/xgboost4j-spark/target/xgboost4j-spark-0.90.jar -DgroupId=ml.dmlc -DartifactId=xgboost4j-spark -Dversion=0.90 -Dpackaging=jar  
mvn install:install-file -Dfile=/opt/soft/xgboost/jvm-packages/xgboost4j-example/target/xgboost4j-example-0.90.jar -DgroupId=ml.dmlc -DartifactId=xgboost4j-example -Dversion=0.90 -Dpackaging=jar (example这个jar可不引)  
mvn install:install-file -Dfile=/opt/soft/xgboost/jvm-packages/xgboost4j/target/xgboost4j-0.90.jar -DgroupId=ml.dmlc -DartifactId=xgboost4j -Dversion=0.90 -Dpackaging=jar  
### 5、PMML
spark与jpmml版本依赖关系https://github.com/jpmml/jpmml-sparkml  
生成pmml模型：jpmml-sparkml，jpmml-sparkml-xgboost  
Java加载pmml预测：pmml-evaluator  
将jpmml-sparkml-xgboost-1.0-SNAPSHOT.jar引入maven  
mvn install:install-file -Dfile=/opt/temp/jpmml-sparkml-xgboost-1.0-SNAPSHOT.jar -DgroupId=org.jpmml -DartifactId=jpmml-sparkml-xgboost -Dversion=1.0-SNAPSHOT -Dpackaging=jar  
项目包含:jar依赖（编译好的jar附到该路径下了，不想编译的也可用这两运行：xgb-model/src/main/resources）、二/多分类建模demo、pmml生成、Java调取pmml预测  


### 6、xgb无法运行bug之一

pipeline.fit时报错，可能是因为虚拟机IP被自己改了，自己备份虚拟机后重启可能发生连不上网络状态，重设IP后虽然可以连网，但用spark的一些配置需要跟着新IP走：

确定自己可连网的某个IP

sudo vi /etc/hosts

IP （空格） machineName 【多数时候是修改旧IP地址】

退出后 /etc/init.d/network restart
