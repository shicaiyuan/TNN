# How to Create a TNN Model
## Overview

<div align=left><img src="https://raw.githubusercontent.com/darrenyao87/tnn-models/master/doc/cn/user/resource/convert.png"/>

TNN currently supports the industry's mainstream model file formats, including ONNX, Pytorch, Tensorflow and Caffe. As shown in the figure above, TNN utilises ONNX as the intermediate port to support multiple model file formats. 
To convert model file formats such as Pytorch, Tensorflow, and Caffe to TNN, you need to use corresponding tool to convert from the original format to ONNX model first, which then will be transferred into a TNN model.

| Source Model   | Convertor        | Target Model |
|------------|-----------------|----------|
| Pytorch    | pytorch export directly | ONNX     |
| Tensorflow | tensorflow-onnx | ONNX     |
| Caffe      | caffe2onnx      | ONNX     |
| ONNX       | onnx2tnn        | TNN      |

At present, TNN only supports common network structures such as CNN. Networks like RNN and GAN are under development.

## TNN Model Converter

Through the general introduction above, you can find that it takes at least two steps to convert a Tensorflow model into a TNN model. So we provide an integrated tool, convert2tnn, to simplify. which can convert Tensorflow, Caffe and ONNX models into TNN models in one single operation. Since Pytorch can directly export ONNX models, this tool no longer provides support for the Pytorch model.

You can use the convert2tnn tool to directly convert the models to TNN, or you can first convert the corresponding model into ONNX format basedand then convert the ONNX to the TNN model based on the documents.

This article provides two ways to help you use the convert2tnn tool:
- Use covnert2tnn via docker image;
- Manually install dependent tools and compiler tools using convert2tnn conversion tool;

### Convert2tnn Docker (Recommend)

In order to simplify the installation and compilation steps of the convert2tnn conversion tool, TNN currently provides a Dockerfile and a Docker image method. You can build the docker image yourself based on the Dockerfile file, or you can directly pull the built Docker image from the company docker . You can choose the way you like to get the docker image.

#### Build a docker image
``` shell script
cd <path-to-tnn>/tools/
docker build -t tnn-convert:latest.
```
Docker will build based on the Dockerfile file, which needs to wait for a while. After the construction is completed, you can verify whether the construction is completed by the following command.
``` shell script
docker images
```
In the output list, there will be similar output below, which indicates that the docker image has been built.
``` text
REPOSITORY TAG IMAGE ID CREATED SIZE
tnn-convert latest 9e2a73fbfb3b 18 hours ago 2.53GB
```

#### Use convert2tnn to convert

First verify that the docker image can be used normally. First, let's look at the help information of convert2tnn by the following command:

``` shell script
docker run -it tnn-convert:latest python3 ./converter.py -h
```
If the docker image is correct, you will get the following output:
```text
usage: convert [-h] {onnx2tnn,caffe2tnn,tf2tnn} ...

convert ONNX/Tensorflow/Caffe model to TNN model

positional arguments:
  {onnx2tnn,caffe2tnn,tf2tnn}
    onnx2tnn            convert onnx model to tnn model
    caffe2tnn           convert caffe model to tnn model
    tf2tnn              convert tensorflow model to tnn model

optional arguments:
  -h, --help            show this help message and exit
```
From above，we can know that currently convert2tnn provides conversion support for 3 model formats. Suppose we want to convert the Tensorflow model to a TNN model here, we enter the following command to continue to get help information:

``` shell script
docker run  -it tnn-convert:latest  python3 ./converter.py tf2tnn -h
```
The output shows below：
``` text
usage: convert tf2tnn [-h] -tp TF_PATH -in input_name -on output_name
                      [-o OUTPUT_DIR] [-v v1.0] [-optimize] [-half]

optional arguments:
  -h, --help       show this help message and exit
  -tp TF_PATH      the path for tensorflow graphdef file
  -in input_name   the tensorflow model's input names
  -on output_name  the tensorflow model's output name
  -o OUTPUT_DIR    the output tnn directory
  -v v1.0          the version for model
  -optimize        optimize the model
  -half            optimize the model
```
Here is the explanations for each parameters:

- tp parameter (required)
    Use the "-tp" parameter to specify the path of the model to be converted. Currently only supports the conversion of a single TF model, does not support the conversion of multiple TF models together.
- in parameter (required)
    Specify the name of the model input through the "-in" parameter. If the model has multiple inputs, use "," to split
- on parameter (required)
    Specify the name of the model input through the "-on" parameter. If the model has multiple outputs, use "," to split
- output_dir parameter:
    You can specify the output path through the "-o <path>" parameter, but we generally do not apply this parameter in docker. By default, the generated TNN model will be placed in the same path as the TF model.
- optimize parameter (optional)
    You can optimize the model with the "-optimize" parameter. **We strongly recommend that you enable this option. Only when the model conversion fails while you enable this option, you could remove the "-opti mize" parameter and try again**.
- v parameter (optional)
    You can use -v to specify the version number of the model to facilitate later tracking and differentiation of the model.
- half parameter (optional)
   The model data will be stored in FP16 to reduce the size of the model by setting this parameter. By default, the model data is stored in FP32.


**Current convert2tnn input model only supports graphdef format，does not support checkpoint or saved_model format. Refer to [tf2tnn](./tf2tnn_en.md) to transfer checkpoint or saved_model models.

Here is an example of converting a TF model in a TNN model

``` shell script
docker run --volume=$(pwd):/workspace -it tnn-convert:latest  python3 ./converter.py tf2tnn -tp=/workspace/test.pb -in=input0,input2 -on=output0 -v=v2.0 -optimize 
```

Since the convert2tnn tool is deployed in the docker image, if you want to convert the model, you need to first push the model into the docker container. We can use the docker run parameter --volume to copy the model to certain path in docker container. In the above example, the current directory (pwd) for executing the shell is under the "/workspace" folder in the docker container. The test.pb used in the test therefore **must be executed under the current path of the shell command**. After executing the above command, the convert2tnn tool will store the generated TNN model in the same level directory of the test.pb file.The generated file is also in the current directory.

The above information only introduces the conversion for Tensorflow's models. It is similar for other models. You can use the conversion tool's note to remind yourself. These conversion commands are listed below:

``` shell script
# convert onnx
docker run --volume=$(pwd):/workspace -it tnn-convert:latest python3 ./converter.py onnx2tnn /workspace/mobilenetv3-small-c7eb32fe.onnx -optimize -v=v3.0
# convert caffe
docker run --volume=$(pwd):/workspace -it tnn-convert:latest python3 ./converter.py caffe2tnn /workspace/squeezenet.prototxt /workspace/squeezenet.caffemodel -optimize -v=v1.0

```

### Manual Convert2tnn Installation
You can also install the dependent tool of convert2tnn on your development machine manually and compile it according to the relevant instructions. 

If you only want to convert the models of certain types, you just need to install the corresponding dependent tools. For example, if you only want to convert the caffe model, you do not need to install the tools that the Tensorflow model depends on. Similarly, if you need to convert Tensorflow's model, you don't need to install Caffe model conversion dependent tools. However, the ONNX model depends on tools and installation and compilation are required.

The example runs on Centos 7.2. It can also be applied to Ubuntu, as long as the corresponding installation command is modified to the corresponding command on Ubuntu.

#### Build the environment
##### 1. Build ONNX model conversion tool (Required)

- Install protobuf(version >= 3.4.0) 
 
Macos:
```shell script
brew install protobuf
```
## Set proxy (optional)
export http_proxy=http://{addr}:{port}
export https_proxy=http://{addr}:{port}
## Compile
cd <path-to-tnn>/tools/onnx2tnn/onnx-converter
./build.sh 
```

- install python (version >=3.6)  

Macos
```shell script
brew install python3
```
centos:
```shell script
yum install  python3 python3-devel
```

- install python dependencies
onnx=1.6.0  
onnxruntime>=1.1.0   
numpy>=1.17.0  
onnx-simplifier>=0.2.4  
```shell script
pip3 install onnx==1.6.0 onnxruntime numpy onnx-simplifier
```

- cmake （version >= 3.0）
Download the latest cmake，and follow the instructions in official documents to install.
It is recommended to use the latest version.

###### Compile
The onnx2tnn tool can run directly on Mac and Linux with automatic compilation scripts.
 ```shell script
cd <path-to-tnn>/tools/onnx2tnn/onnx-converter
./build.sh
 ```

##### 2. Convert the TensorFlow Model (Optional)

-tensorflow (version == 1.15.0)
It is recommended to use tensorflow version 1.15.0. The current version of tensorflow 2.+ is not compatible and is not recommended.
```shell script
pip3 install tensorflow==1.15.0
```

-tf2onnx (version>= 1.5.5)
```shell script
pip3 install tf2onnx
```
-onnxruntime(version>=1.1.0)
```shell script
pip3 install onnxruntime
```
##### 3. Convert the Caffe model (Optional)

-Install protobuf (version >= 3.4.0)

Macos:
```shell script
brew install protobuf
```

Linux:

For linux system, we suggest to refer to the official [README](https://github.com/protocolbuffers/protobuf/blob/master/src/README.md) document of protobuf and install directly from the source code.

If you are using Ubuntu system, you can use the following instructions to install:

```shell script
sudo apt-get install libprotobuf-dev protobuf-compiler
```

- Install python (version >=3.6)  

Macos
```shell script
brew install python3
```
centos:
```shell script
yum install  python3 python3-devel
```

- onnx(version == 1.6.0)
```shell script
pip3 install onnx==1.6.0
```

- numpy(version >= 1.17.0)
```shell script
pip3 install numpy
```

#### convert2tnn Tool Usage
After meeting the requirements, convert2tnn could convert the models.

```shell script
cd <path_to_tnn_root>/tools/convert2tnn/
python3 converter.py -h
```
Execute the command above will output information below. There 3 options at present.

```text
usage: convert [-h] {onnx2tnn,caffe2tnn,tf2tnn} ...

convert ONNX/Tensorflow/Caffe model to TNN model

positional arguments:
  {onnx2tnn,caffe2tnn,tf2tnn}
    onnx2tnn            convert onnx model to tnn model
    caffe2tnn           convert caffe model to tnn model
    tf2tnn              convert tensorflow model to tnn model

optional arguments:
  -h, --help            show this help message and exit
```
- ONNX model conversion
If you want to convert ONNX models, you can directly choose the onnx2tnn option to view the help information.

```shell script
python3 converter.py onnx2tnn -h
```
usage information：
```text
usage: convert onnx2tnn [-h] [-optimize] [-half] [-v v1.0.0] [-o OUTPUT_DIR]
                        onnx_path

positional arguments:
  onnx_path      the path for onnx file

optional arguments:
  -h, --help     show this help message and exit
  -optimize      optimize the model
  -half          save model using half
  -v v1.0.0      the version for model
  -o OUTPUT_DIR  the output tnn directory
```
Example:
```shell script
python3 converter.py onnx2tnn ~/mobilenetv3/mobilenetv3-small-c7eb32fe.onnx.opt.onnx -optimize -v=v3.0 -o ~/mobilenetv3/ 
```

- caffe2tnn

caffe format conversion

The convert2tnn currently only supports the latest version of Caffe file format, so if you want to convert Caffe model to TNN model. You need to convert the old version caffe network and model into new version first. Caffe comes with such tools.

The caffe network and model are converted to the new version format. The specific usage is as follows:

```shell script
upgrade_net_proto_text [old prototxt] [new prototxt]
upgrade_net_proto_binary [old caffemodel] [new caffemodel]
```

The format after modification:

```text
layer {
  name: "data"
  type: "input"
  top: "data"
  input_param { shape: { dim: 1 dim: 3 dim: 224 dim: 224 } }
}
```


```shell script
python3 converter.py caffe2tnn -h
```
usage information：
```text
usage: convert caffe2tnn [-h] [-o OUTPUT_DIR] [-v v1.0] [-optimize] [-half]
                         prototxt_file_path caffemodel_file_path

positional arguments:
  prototxt_file_path    the path for prototxt file
  caffemodel_file_path  the path for caffemodel file

optional arguments:
  -h, --help            show this help message and exit
  -o OUTPUT_DIR         the output tnn directory
  -v v1.0               the version for model, default v1.0
  -optimize             optimize the model
  -half                 save model using half
```
Example：
```shell script
python3 converter.py caffe2tnn ~/squeezenet/squeezenet.prototxt ~/squeezenet/squeezenet.caffemodel -optimize -v=v1.0 -o ~/squeezenet/ 
```
- tensorflow2tnn

The current convert2tnn model only supports the graphdef model, and does not support checkpoint and saved_model format files. If you want to convert the checkpoint or saved_model model, you can refer to the tf2onnx section below to convert it yourself.

``` shell script
python3 converter.py tf2tnn -h
```
usage information：
```
usage: convert tf2tnn [-h] -tp TF_PATH -in input_name -on output_name
                      [-o OUTPUT_DIR] [-v v1.0] [-optimize] [-half]

optional arguments:
  -h, --help       show this help message and exit
  -tp TF_PATH      the path for tensorflow graphdef file
  -in input_name   the tensorflow model's input names
  -on output_name  the tensorflow model's output name
  -o OUTPUT_DIR    the output tnn directory
  -v v1.0          the version for model
  -optimize        optimize the model
  -half            optimize the model
```
Example：
```shell script
python3 converter.py tf2tnn -tp ~/tf-model/test.pb -in=input0,input2 -on=output0 -v=v2.0 -optimize -o ~/tf-model/
```

## Model Conversion Details
convert2tnn is just an encapsulation of a variety of tools for model converting. According to the principles explained in the first part "Introduction to model conversion", you can also convert the original model into ONNX first, and then convert the ONNX model into a TNN model. We provide documentation on how to manually convert Caffe, Pytorch, TensorFlow models into ONNX models, and then convert ONNX models into TNN models. If you encounter problems when using the convert2tnn conversion tool, we recommend that you understand the relevant content, which may help you to use the tool more smoothly.

- [onnx2tnn](onnx2tnn_en.md)
- [pytorch2tnn](onnx2tnn_en.md)
- [tf2tnn](tf2tnn_en.md)
- [caffe2tnn](caffe2tnn_en.md)
