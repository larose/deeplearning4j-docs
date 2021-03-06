---
title: 用VGG-16构建识别图像的网页应用
layout: cn-default
---

# 如何用VGG-16构建识别图像的网页应用

神经网络不断刷新着图像识别的准确率纪录。本页主要介绍如何构建一个基于网页的应用程序，用著名的VGG-16网络实现推断，对用户上传的图像进行分类。 

**目录**

* [VGG-16是什么？](#VGG-16)
* [将VGG-16用于网页应用](#UsingVGG-16)
* [加载预训练模型](#LoadingPre-TrainedModels)
* [为测试配置数据加工管道](#ConfigureDataPipelinesforTesting)
* [测试预训练模型](#LoadingPre-TrainedModels)
* [用ModelSerializer保存模型](#SaveModelwithModelSerializer)
* [构建接受输入图像的网页应用](#BuildWebApptoTakeInputImage)
* [将网页应用前端与神经网络后端绑定](#TieWebAppFrontEndtoNeuralNetBackend)
* [直接看代码](#code)
* [预测示例（猫咪！狗狗！）](#example)

## <a name="VGG-16"> VGG-16是什么？</a>

从2010年开始，[ImageNet](http://image-net.org/)每年都会举办一场[挑战赛](http://www.image-net.org/challenges/LSVRC/)，参赛的研究团队用ImageNet数据集作为训练数据，提出各种图像分类解决方案。ImageNet目前有数百万幅已标记的图像，是全世界最大的高质量图像数据集之一。牛津大学的Visual Geometry Group（视觉几何研究组）在2014年的比赛中取得了优异成绩，他们采用的网络架构有两种：16层卷积神经网络VGG-16和19层卷积神经网络VGG-19。 

结果如下：

* [VGG-16 ILSVRC-14](http://www.image-net.org/challenges/LSVRC/2014/results#clsloc) 
* VGG-16在处理VOC和Caltech这两个图像分类基准数据集时的表现同样很出色。相关结果参见[此处](http://www.robots.ox.ac.uk/~vgg/research/very_deep/)。

## <a name="UsingVGG-16"> 将VGG-16用于网页应用</a>

神经网络应用的第一步是训练。在训练过程中，输入数据（此处为图像）进入网络，随后网络的输出或预测结果会与预期结果（即图像的标签）进行比较。每次完成对数据的迭代后，网络的权重会得到调整，以减少预测的误差率——也就是说，权重调整后，网络预测结果与实际图像标签的匹配率可得到改善。（用数百万幅图像对一个大型网络进行训练需要消耗大量计算资源，因此有必要发布此处介绍的预训练网络。）

训练完毕后，网络可以用于推断，即对其读取的数据进行预测。推断恰好是一种不需要这么多计算资源的过程。对于VGG-16和其他架构而言，开发者可以下载预训练的模型直接使用，而无需掌握模型调试与训练的必要技能。预训练模型的加载和预测应用相对比较简单，本页将作详细介绍。 

我们将在另一篇教程中介绍如何加载此类模型并进一步训练。  

## <a name="LoadingPre-TrainedModels"> 加载预训练模型</a>

0.9.0版（0.8.1-SNAPSHOT）的Deeplearning4J配有一个全新的原生模型库。您可以阅读[deeplearning4j-zoo](/model-zoo)模块的内容，了解更多关于使用预训练模型的信息。此处我们需要加载一个预训练的VGG-16模型，用经过ImageNet数据集训练后得到的权重来初始化：

```
ZooModel zooModel = new VGG16();
ComputationGraph vgg16 = zooModel.initPretrained(PretrainedType.IMAGENET);
```

### 导入Keras模型

Deeplearning4J还配有专门面向[Keras](http://keras.io/)的模型导入工具。我们的[模型导入器](/model-import-keras)可将您的Keras网络配置及权重转换为Deeplearning4J的格式。Keras的顺序（Sequential）模型和标准函数式（Functional）模型都可以导入DL4J。导入模型的代码可如此编写：


```
ComputationGraph model = KerasModelImport.importKerasModelAndWeights(modelJsonFilename, weightsHdf5Filename, enforceTrainingConfig);

```

如果导入的预训练模型*仅用于推断*，那么应当设置`enforceTrainingConfig=false`。目前尚不支持的仅用于训练的模型配置会触发警告消息，但模型导入功能会继续运行。

## <a name="ConfigureDataPipelinesforTesting"> 为测试配置数据加工管道</a>

就数据摄取和预处理而言，您可以选择手动流程或者助手功能。VGG-16形式的图像处理助手功能为`TrainedModels.VGG16.getPreProcessor`以及`VGG16ImagePreProcessor()`。（请记住：用于推断的图像的预处理方式必须和训练图像的处理方式相同。） 

### VGG-16图像预处理管道的步骤

1. 缩放至224 * 224、3通道（RGB图像）
2. 均值缩放，每个像素的像素值都减去平均像素值

缩放工作可以由DataVec的原生图像加载器完成。 

均值消减可以手动操作，或者由助手功能完成。 

### 将图像缩放至高224、宽224、3通道的代码示例

如果读取的是图像目录，请使用DataVec的ImageRecordReader

```
ImageRecordReader rr = new ImageRecordReader(224,224,3);
```

如果读取的是单一图像，请使用DataVec的NativeImageLoader

```
NativeImageLoader loader = new NativeImageLoader(224, 224, 3);
INDArray image = loader.asMatrix(file);
```


### 均值消减的代码示例

如果用DataVec的RecordReader加载一个目录

```
 DataSetPreProcessor preProcessor = TrainedModels.VGG16.getPreProcessor();
 dataIter.setPreProcessor(preProcessor);
```

如果采用NativeImageLoader加载图像

```
DataNormalization scaler = new VGG16ImagePreProcessor();
scaler.transform(image);
```

## <a name="TestingPre-TrainedModels"> 测试预训练模型</a>

网络加载完毕后，您应当验证它能按预期的方式工作。请注意，ImageNet并非为人脸识别而设计，所以最好用大象、狗或者猫的图片来测试。 

如果您想将运行结果与Keras的输出作比较，请分别在Keras和DeepLearning4J中加载模型，然后比较两者的输出。最终的结果应该相当接近。如果Keras表示该图像有35.00094%的概率是大象，而DeepLearning4j输出的概率为35.00104%，这很可能只是取整误差，而不是因为模型之间存在实际差异。

### 测试图像目录的代码

```

while (dataIter.hasNext()) {
            //预测数组
            DataSet next = dataIter.next();
            INDArray features = next.getFeatures();
            INDArray[] outputA = vgg16.output(false,features);
            INDArray output = Nd4j.concat(0,outputA);

            //对数据集中的每幅图像，显示排名前五位的预测结果
            List<RecordMetaData> trainMetaData = next.getExampleMetaData(RecordMetaData.class);
            int batch = 0;
            for(RecordMetaData recordMetaData : trainMetaData){
                System.out.println(recordMetaData.getLocation());
                System.out.println(TrainedModels.VGG16.decodePredictions(output.getRow(batch)));
                batch++;
            }

```
 

### 测试由命令行提示输入的图像的代码


```

//缓冲流加载器给出提示，请求指定测试图像的 
// 文件路径
InputStreamReader r = new InputStreamReader(System.in);
BufferedReader br = new BufferedReader(r);
for (; ; ){
    System.out.println("type EXIT to close");
    System.out.println("Enter Image Path to predict with VGG16");
    System.out.print("File Path: ");
    String path = br.readLine();
    if ("EXIT".equals(path))
        break;
    System.out.println("You typed" + path);


	// 此处代码将已提交图像转换为INDArray
	// 进行均值消减并运行推断
    File file = new File(path);
    NativeImageLoader loader = new NativeImageLoader(224, 224, 3);
    INDArray image = loader.asMatrix(file);
    DataNormalization scaler = new VGG16ImagePreProcessor();
    scaler.transform(image);
    INDArray[] output = vgg16.output(false,image);
    System.out.println(TrainedModels.VGG16.decodePredictions(output[0]));

```

## <a name="SaveModelwithModelSerializer"> 用ModelSerializer保存模型</a>

加载模型并测试完毕后，请用DeepLearning4J的`ModelSerializer`保存模型。和用Keras加载模型相比，用ModelSerializer加载模型消耗的资源更少。我们建议您加载一次，然后将模型保存为DeepLearning4J格式，以便之后再次使用。 

### 将模型保存至文件的代码

```
File locationToSave = new File("vgg16.zip");
ModelSerializer.writeModel(model,locationToSave,saveUpdater);
		
```		

### 从文件中加载已保存模型的代码

```
File locationToSave = new File("/Users/tomhanlon/SkyMind/java/Class_Labs/vgg16.zip");
ComputationGraph vgg16 = ModelSerializer.restoreComputationGraph(locationToSave);

```

## <a name="BuildWebApptoTakeInputImage"> 构建接受输入图像的网页应用</a>

下列HTML代码生成的表单元素可以让用户选择一幅图像并上传至我们的服务器。以下示例并未连接至服务器。（尚未完成！） 

<pre>
<form method='post' action='getPredictions' enctype='multipart/form-data'>
    <input type='file' name='uploaded_file'>
    <button>Upload picture</button>
</form>
</pre>

表单元素的动作属性是用户所选图像的上传URL。 

我们选择用[Spark Java](http://sparkjava.com/)编写网页应用，因为它比较简单明了。您只需将Spark Java代码添加至一项已经编写好的类中即可。其他选择还有很多。 

无论您选用哪种Java网页框架，以下步骤都是相同的。 

1. 构建一个表单 
2. 测试表单 
3. 确保文件上传功能运行正常，编写一些Java代码来测试路径字符串，或者测试能否将文件作为Java文件对象访问。 
4. 将网页应用功能作为输入连接至神经网络

## <a name="TieWebAppFrontEndtoNeuralNetBackend"> 将网页应用前端与神经网络后端绑定</a>

可正常工作的最终代码如下所示。 

这个类运行时会在4567端口上启动Jetty网页服务器监听。 

网页应用的启动时间大约和神经网络的加载时间相同。加载VGG-16大约需要四分钟。 

一旦网络开始运行，其RAM使用量大约会以60MB为单位递增，直至达到4G，此时垃圾回收功能会开始清理内存。我们在一个AWS t2-large实例上运行VGG-16，测试了大约一周时间，运行状况很稳定。或许还可以使用更小的AMI。  

## <a name="code"> 完整代码示例</a>


```

package org.deeplearning4j.VGGwebDemo;

import org.datavec.image.loader.NativeImageLoader;
import org.deeplearning4j.nn.graph.ComputationGraph;
import org.deeplearning4j.nn.modelimport.keras.trainedmodels.TrainedModels;
import org.deeplearning4j.util.ModelSerializer;
import org.nd4j.linalg.api.ndarray.INDArray;
import org.nd4j.linalg.dataset.api.preprocessor.DataNormalization;
import org.nd4j.linalg.dataset.api.preprocessor.VGG16ImagePreProcessor;
import javax.servlet.MultipartConfigElement;
import java.io.File;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import static spark.Spark.*;

/**
 * Created by tomhanlon on 1/25/17.
 */
public class VGG16SparkJavaWebApp {
    public static void main(String[] args) throws Exception {

        // 设置HTTPS加密证书的位置
        String keyStoreLocation = "clientkeystore";
        String keyStorePassword = "skymind";
        secure(keyStoreLocation, keyStorePassword, null,null );

        // 加载已训练的模型
        File locationToSave = new File("vgg16.zip");
        ComputationGraph vgg16 = ModelSerializer.restoreComputationGraph(locationToSave);


        // 将上传目录用于用户提交的图像
        // 图像上传、处理后予以删除
        File uploadDir = new File("upload");
        uploadDir.mkdir(); // create the upload directory if it doesn't exist

        // 以下String类将会显示一个用于选择和上传图像的HTML表单
        String form = "<form method='post' action='getPredictions' enctype='multipart/form-data'>\n" +
                "    <input type='file' name='uploaded_file'>\n" +
                "    <button>Upload picture</button>\n" +
                "</form>";


        // 用以处理请求的Spark Java配置
        // 测试请求，URL /hello应返回“hello world”
        get("/hello", (req, res) -> "Hello World");

        // VGGpredict的请求会返回提交图像的表单
        get("VGGpredict", (req, res) -> form);

        // getPredictions（注意表单的动作属性）
        //  Post请求（注意表单采用的是http post发布）
        // 页面显示结果及另一个表单

        post("/getPredictions", (req, res) -> {

            Path tempFile = Files.createTempFile(uploadDir.toPath(), "", "");

            req.attribute("org.eclipse.jetty.multipartConfig", new MultipartConfigElement("/temp"));

            try (InputStream input = req.raw().getPart("uploaded_file").getInputStream()) { // getPart需要使用和表单输入字段相同的“名称”
                Files.copy(input, tempFile, StandardCopyOption.REPLACE_EXISTING);
            }


            // 用户提交的文件为tempFile，转换为Java文件“file”
            File file = tempFile.toFile();

            // 将file转换为INDArray
            NativeImageLoader loader = new NativeImageLoader(224, 224, 3);
            INDArray image = loader.asMatrix(file);

            // 删除实体文件，如果保留则硬盘会逐渐填满
            file.delete();

            // VGG网络的均值消减预处理
            DataNormalization scaler = new VGG16ImagePreProcessor();
            scaler.transform(image);

            //推断返回INDArray数组，index[0]为预测结果
            INDArray[] output = vgg16.output(false,image);

            // 用助手功能VGG16.decodePredictions将标签的概率排序并
            // 返回前五位结果，转换为字符串
            // “predictions”为结果的字符串
            String predictions = TrainedModels.VGG16.decodePredictions(output[0]);

            // 返回结果以及运行另一次推断的表单
            return "<h4> '" + predictions  + "' </h4>" +
                    "Would you like to try another" +
                    form;
            return "<h1>Your image is: '" + tempFile.getName(1).toString() + "' </h1>";

        });


    }

}

```

## <a name="example"> 预测示例</a>

Skymind公司养了几只猫，以下是VGG-16对其中之一的照片的预测结果，网络可能从未见过这只猫。（因为他很害羞……）

![a cat for inference](./../img/cat.jpeg)

	16.694832%, tabby 7.550286%, tiger_cat 0.065847%, cleaver 0.000000%, cleaver 0.000000%, cleaver

VGG-16对下面这只来自互联网的狗给出了相当准确的预测结果，网络在训练过程中有可能见过这幅图片。

![a dog for inference](./../img/dog_320x240.png)

	53.441956%, bluetick 17.103373%, English_setter 5.808368%, kelpie 3.517581%, Greater_Swiss_Mountain_dog 2.263778%, German_short-haired_pointer'
