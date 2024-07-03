# camera2

## camera2过程
1. 从camera manager获取camera device

 1.1 openCamera，传入cameraDevice的stateCallback

       1.1.1 在stateCallback中实现onOpened

        1.1.1.1 onOpened中createCaptureSession

2. 创建cameraCaptureSession（session）

 2.1 session创建时传入surface

       2.1.1 surface用ImageReader来获取原始数据

 2.2 session的stateCallback

       2.2.1 StateCallback内实现：onConfigured

        2.2.1.1 onConfigured回调中build到cameraCaptureRequest

        2.2.1.2 session.setRepeatingRequest，重复获取得到流

       2.2.2 StateCallback内实现：onConfigureDailed（可默认）

3. 将ImageReader拿到的数据使用bitmap处理


## ImageReader

## CameraReader


## 问题记录
添加ImageReader的surface以后预览总是停止？
