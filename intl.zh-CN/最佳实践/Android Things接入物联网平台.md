# Android Things接入物联网平台 {#concept_ntf_mgq_p2b .concept}

本文档以室内空气检测项目为例，介绍如何将谷歌Android Things物联网硬件接入阿里云物联网平台。

## 硬件设备 {#section_udw_ctq_p2b .section}

-   项目设备列表

    下表中为室内空气检测所需的硬件设备：

    |设备|设备图片|备注|建议购买渠道|
    |--|----|--|------|
    | NXP Pico i.MX7D

 开发板

 | ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16822/15331136467864_zh-CN.png)

 |Android things系统1.0| 闲鱼网

 **说明：** 该硬件可以用树莓派替代。

 |
    | DHT12

 温湿度传感器

 | ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16822/15331136477865_zh-CN.png)

 |I2C数据通信方式|淘宝|
    | ZE08-CH2O

 甲醛检测传感器

 |![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16822/15331136477866_zh-CN.png)

|UART数据通信方式|淘宝|

-   NXP i.MX7D开发板引脚示意图

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16822/15331136477888_zh-CN.png)

    如需更多帮助，请参见NXP Pico i.MX7D I/O官网接口文档：[https://developer.android.com/things/hardware/imx7d-pico-io](https://developer.android.com/things/hardware/imx7d-pico-io)。

-   设备接线示意图

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16822/15331136477908_zh-CN.png)

    -   将温湿度传感器（DHT12）的时钟信号线引脚SCL和数据线引脚SDA分别与开发板的I2C总线的SCL和SDA引脚相接。
    -   将甲醛检测传感器（ZE08-CH2O）的发送数据引脚TXD与开发板的接收数据引脚RXD相接；将ZE08-CH2O的接收数据引脚RXD与开发板的发送数据引脚TXD相接。

## 创建阿里云物联网平台产品和设备 {#section_pqg_qrk_q2b .section}

如果您还未开通阿里云物联网平台服务，请先了解、开通[物联网平台](https://www.aliyun.com/product/iot)。

1.  登录[阿里云物联网平台控制台](https://iot.console.aliyun.com)。
2.  创建高级版产品。

    在产品管理页面，单击**创建产品**进入产品创建流程。创建产品时，产品版本选择为**高级版**。如需帮助，请参见[创建产品](../../../../intl.zh-CN/用户指南/创建产品与设备/高级版/创建产品.md#)。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16822/15331136477969_zh-CN.png)

3.  定义产品功能。

    进入产品详情页面，单击**功能定义** \> **新增**，为产品添加属性定义。如需帮助，请参见[定义物模型](../../../../intl.zh-CN/用户指南/创建产品与设备/高级版/定义物模型.md#)。

    |属性名称|标识符|数据类型|取值范围|描述|
    |:---|:--|:---|:---|:-|
    |温度|temperature|float|-50~100|DHT12温湿度传感器采集|
    |湿度|humidity|float|0~100|DHT12温湿度传感器采集|
    |甲醛浓度|ch2o|double|0~3|ZE08甲醛检测传感器采集|

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16822/15331136477991_zh-CN.png)

4.  创建设备。

    在设备管理页面，选择刚创建的产品名称，然后单击**添加设备**，在产品下创建设备。如需帮助，请参见[创建设备](../../../../intl.zh-CN/用户指南/创建产品与设备/高级版/创建设备.md#)。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16822/15331136477994_zh-CN.png)


## 开发Android things设备端 {#section_cd4_d2l_q2b .section}

1.  使用Android Studio创建Android things工程，添加网络权限。

    ```
    <uses-permission android:name="android.permission.INTERNET" />
    ```

2.  在gradle中添加eclipse.paho.mqtt。

    ```
    implementation 'org.eclipse.paho:org.eclipse.paho.client.mqttv3:1.2.0'
    ```

3.  配置通过I2C读取温湿度传感器DHT12的数据。

    ```
    private void readDataFromI2C() {
    
            try {
    
                byte[] data = new byte[5];
                i2cDevice.readRegBuffer(0x00, data, data.length);
    
                // check data
                if ((data[0] + data[1] + data[2] + data[3]) % 256 != data[4]) {
                    humidity = temperature = 0;
                    return;
                }
                // humidity data
                humidity = Double.valueOf(String.valueOf(data[0]) + "." + String.valueOf(data[1]));
                Log.d(TAG, "humidity: " + humidity);
                // temperature data
                if (data[3] < 128) {
                    temperature = Double.valueOf(String.valueOf(data[2]) + "." + String.valueOf(data[3]));
                } else {
                    temperature = Double.valueOf("-" + String.valueOf(data[2]) + "." + String.valueOf(data[3] - 128));
                }
    
                Log.d(TAG, "temperature: " + temperature);
    
            } catch (IOException e) {
                Log.e(TAG, "readDataFromI2C error " + e.getMessage(), e);
            }
    
        }
    ```

4.  配置通过UART获取甲醛检测传感器Ze08-CH2O的数据。

    ```
    try {
                    // data buffer
                    byte[] buffer = new byte[9];
    
                    while (uartDevice.read(buffer, buffer.length) > 0) {
    
                        if (checkSum(buffer)) {
                            ppbCh2o = buffer[4] * 256 + buffer[5];
                            ch2o = ppbCh2o / 66.64 * 0.08;
                        } else {
                            ch2o = ppbCh2o = 0;
                        }
                        Log.d(TAG, "ch2o: " + ch2o);
                    }
    
                } catch (IOException e) {
                    Log.e(TAG, "Ze08CH2O read data error " + e.getMessage(), e);
                }
    ```

5.  创建阿里云物联网平台与设备端的连接，上报数据。

    ```
    /*
    payload格式
    {
      "id": 123243,
      "params": {
        "temperature": 25.6,
        "humidity": 60.3,
        "ch2o": 0.048
      },
      "method": "thing.event.property.post"
    }
    */
    MqttMessage message = new MqttMessage(payload.getBytes("utf-8"));
    message.setQos(1);
    
    String pubTopYourPc = "/sys/${YourProductKey}/${YourDeviceName}/thing/event/property/post";
    
    mqttClient.publish(pubTopic, message);
    ```


## 查看实时数据 {#section_n34_g5l_q2b .section}

设备启动后，您可以在阿里云物联网平台控制台，设备详情页，运行状态中查看设备当前的实时数据。

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16822/15331136478025_zh-CN.png)

