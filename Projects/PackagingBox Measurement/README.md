# PackagingBox Measurement

## Dimensioning and Weighing Solutions using Depth AI

In the world of warehousing, `space = money` and `high labour cost = lesser profits.` Warehouse problems can affect the speed, efficiency, and productivity of either one particular warehouse operation or the entire chain of processes that are linked with it. Packaging operations have always been a real time-thief in warehouses and often almost completely manual. Often it is a real bottleneck in the warehouse flow. A half-hour before the trucks leave the warehouse, there is usually chaos in the packaging area with staff running into each other, looking for the right boxes and packing material to finish in time. This also affects the quality with the mixing of the goods from different orders as a result. Choosing the right package dimension size hep you to prevent costs of unncessary filling materials. 
<pre>
<img src="https://fwlogistics.com/wp-content/uploads/2018/11/20500.jpg" height="280" width="350"> | <img src="https://st4.depositphotos.com/3922387/19990/i/450/depositphotos_199907606-stock-photo-young-asian-man-using-tape.jpg" height="280" width="350">
</pre>

## Operation packaging efficiency is direct result of:
- Correct product packaging dimension measurement, combined with
- Right packaging materials and
- Operational procedures,

These factors, when implemented diligently, can help you manage packages by reducing damages and satisfy your customers through high operational standards. 
From small boxes to large pallets, our solution enables logistics companies to better manage their inventory and shipments.

## Scope of the project:
- Using OAK-D 3D Stereo Vision camera for capturing measurements of package, box or parcel. 
- Measure how much storage space the object would need or determine what size of the carton to use for shipment, estimate shipping costs.
- Use AI-enabled add-ons such as open, leaking, and broken box detection and packaging optimization to improve workforce efficiency and safety.
<pre> 
<img src="https://cdn.shopify.com/s/files/1/0106/8325/2802/products/OAK-D_1024x1024@2x.jpg?v=1612573255" height="220" width="220"> | <img src="https://cdn.shopify.com/s/files/1/0106/8325/2802/products/IMG_2984_1024x1024@2x.jpg?v=1612573255" height="220" width="220"> | <img src="https://cdn.shopify.com/s/files/1/0106/8325/2802/products/ASDFAWEFAGDVASVqdawfaff2353_1024x1024@2x.jpg?v=1612573255" height="220" width="220">
</pre>

## Technology stacks used <img src="https://luxonis.com/img/brand/depthai-logo-with-text.png" width="10%" height="5%" /> | <img src="https://blog.streamlit.io/content/images/size/w1000/2021/03/logomark-color.png" width="10%" height="5%" /> | <img src="https://opencv.org/wp-content/uploads/2020/07/cropped-OpenCV_logo_white_600x.png" width="5%" />

- DepthAI SDK for OAK-D vision camera inferencing and Spatial Data.
- Streamlit for web based dashboard for one button click dimension measuremnt and all metrics calculations.
- OpenCV for all image processing and feature extaction.

## DepthAI brings realtime Spatial AI to your product
DepthAI is a platform built around the Myriad X to combine depth perception, object detection (neural inference), and object tracking that gives you this power in a simple, easy-to-use Python API and a plug/play System on Module (SoM) with Open-Source hardware.

## Project Instructions:
The project has mainly four parts:
1. DepthAI Pipeline configuration for depth and RGB image data
2. Base Depth Estimation
3. Package Contour Detection
4. Package dimension and volume estimation

# 1. DepthAI Pipeline configuration for depth and RGB image data

```Python
# create depthai pipeline
def createPipeline():
    print('Creating Pipeline')

    # Start defining a pipeline
    pipeline = dai.Pipeline()

    # Define color camera
    camRgb = pipeline.createColorCamera()
    camRgb.setPreviewSize(640, 480)
    camRgb.setBoardSocket(dai.CameraBoardSocket.RGB)
    camRgb.setColorOrder(dai.ColorCameraProperties.ColorOrder.BGR)

    # Define a source - two mono (grayscale) cameras
    monoLeft = pipeline.createMonoCamera()
    monoRight = pipeline.createMonoCamera()
    stereo = pipeline.createStereoDepth()
    spatialLocationCalculator = pipeline.createSpatialLocationCalculator()

    # MonoCamera
    monoLeft.setResolution(dai.MonoCameraProperties.SensorResolution.THE_400_P)
    monoLeft.setBoardSocket(dai.CameraBoardSocket.LEFT)
    monoRight.setResolution(dai.MonoCameraProperties.SensorResolution.THE_400_P)
    monoRight.setBoardSocket(dai.CameraBoardSocket.RIGHT)

    outputDepth = True
    outputRectified = False
    lrcheck = False
    subpixel = False

    # Create outputs
    xoutPreview = pipeline.createXLinkOut()
    xoutDepth = pipeline.createXLinkOut()
    xoutSpatialData = pipeline.createXLinkOut()
    xinSpatialCalcConfig = pipeline.createXLinkIn()

    xoutPreview.setStreamName("preview")
    xoutDepth.setStreamName("depth")
    xoutSpatialData.setStreamName("spatialData")
    xinSpatialCalcConfig.setStreamName("spatialCalcConfig")

    # StereoDepth
    stereo.setOutputDepth(outputDepth)
    stereo.setOutputRectified(outputRectified)
    stereo.setConfidenceThreshold(255)

    stereo.setLeftRightCheck(lrcheck)
    stereo.setSubpixel(subpixel)

    monoLeft.out.link(stereo.left)
    monoRight.out.link(stereo.right)

    spatialLocationCalculator.passthroughDepth.link(xoutDepth.input)
    stereo.depth.link(spatialLocationCalculator.inputDepth)

    topLeft = dai.Point2f(0.4, 0.4)
    bottomRight = dai.Point2f(0.8, 0.8)

    spatialLocationCalculator.setWaitForConfigInput(False)
    config = dai.SpatialLocationCalculatorConfigData()
    config.depthThresholds.lowerThreshold = 100
    config.depthThresholds.upperThreshold = 10000
    config.roi = dai.Rect(topLeft, bottomRight)
    spatialLocationCalculator.initialConfig.addROI(config)
    spatialLocationCalculator.out.link(xoutSpatialData.input)
    xinSpatialCalcConfig.out.link(spatialLocationCalculator.inputConfig)
    camRgb.preview.link(xoutPreview.input)

    return pipeline, topLeft, bottomRight, config
```

## 2. Base Depth Estimation
In the start of the application the depth of the base from the camera is calculated for further package height calculation
<img src='https://github.com/vilaksh01/DepthAI-LazyProjects/blob/main/Projects/PackagingBox%20Measurement/Images/calibBig.gif'>

```python
# call base depth estimation to set base depth for height calculations
def baseDepthEstimation(pipeline, topLeft, bottomRight, config):
    st.warning('Please make sure the bounding box area is clear of objects.')
    col1, col2 = st.beta_columns(2)
    with col1:
        st.text('Preview window.')
    with col2:
        st.text('Click start to calibrate depth for avg. 25 frames')
        start = st.button('Start Calibration')

    # Pipeline is defined, now we can connect to the device
    with dai.Device(pipeline) as device:
        device.startPipeline()

        # Output queue will be used to get the depth frames from the outputs defined above
        depthQueue = device.getOutputQueue(name="depth", maxSize=4, blocking=False)
        spatialCalcQueue = device.getOutputQueue(name="spatialData", maxSize=4, blocking=False)
        spatialCalcConfigInQueue = device.getInputQueue("spatialCalcConfig")

        color = (255, 255, 255)
        noFrames = 0
        frameST = col1.empty()
        frameST2 = col2.empty()
        baseDepth = 0.0
        DepthValue = 0.0
        count = False
        fontType = cv2.FONT_HERSHEY_TRIPLEX

        while True:
            inDepth = depthQueue.get()  # Blocking call, will wait until a new data has arrived
            inDepthAvg = spatialCalcQueue.get()  # Blocking call, will wait until a new data has arrived
            preview = device.getOutputQueue('preview').get()

            img = preview.getFrame()

            depthFrame = inDepth.getFrame()
            depthFrameColor = cv2.normalize(depthFrame, None, 255, 0, cv2.NORM_INF, cv2.CV_8UC1)
            depthFrameColor = cv2.equalizeHist(depthFrameColor)
            depthFrameColor = cv2.applyColorMap(depthFrameColor, cv2.COLORMAP_JET)

            spatialData = inDepthAvg.getSpatialLocations()
            for depthData in spatialData:
                roi = depthData.config.roi
                roi = roi.denormalize(width=depthFrameColor.shape[ 1 ], height=depthFrameColor.shape[ 0 ])
                xmin = int(roi.topLeft().x)
                ymin = int(roi.topLeft().y)
                xmax = int(roi.bottomRight().x)
                ymax = int(roi.bottomRight().y)

                # preview window info
                cv2.rectangle(img, (xmin, ymin), (xmax, ymax), color, cv2.FONT_HERSHEY_SCRIPT_SIMPLEX)
                cv2.putText(img, f"X: {int(depthData.spatialCoordinates.x)} mm", (xmin + 10, ymin + 20),
                            fontType, 0.5, color)
                cv2.putText(img, f"Y: {int(depthData.spatialCoordinates.y)} mm", (xmin + 10, ymin + 35),
                            fontType, 0.5, color)
                cv2.putText(img, f"Z: {int(depthData.spatialCoordinates.z)} mm", (xmin + 10, ymin + 50),
                            fontType, 0.5, color)

                # depth window info
                cv2.rectangle(depthFrameColor, (xmin, ymin), (xmax, ymax), color, cv2.FONT_HERSHEY_SCRIPT_SIMPLEX)
                cv2.putText(depthFrameColor, f"X: {int(depthData.spatialCoordinates.x)} mm", (xmin + 10, ymin + 20),
                            fontType, 0.5, color)
                cv2.putText(depthFrameColor, f"Y: {int(depthData.spatialCoordinates.y)} mm", (xmin + 10, ymin + 35),
                            fontType, 0.5, color)
                cv2.putText(depthFrameColor, f"Z: {int(depthData.spatialCoordinates.z)} mm", (xmin + 10, ymin + 50),
                            fontType, 0.5, color)

                if start:
                    start = False
                    count = True
                    noFrames = 1

                if count:
                    DepthValue = (depthData.spatialCoordinates.z) / 10
                    baseDepth += DepthValue
                    print(baseDepth)
                    DepthValue = 0.0

                if noFrames == 25 and count == True:
                    # Base depth value calculation, dividing by no. of frames for the average
                    count = False
                    print('Frames count {:d}'.format(noFrames))
                    baseDepth = baseDepth / 25
                    print('base depth {:>2f}'.format(baseDepth))
                    st.text("Base Depth = {:>2f}".format(baseDepth))
                    cv2.destroyAllWindows()
                    return baseDepth

                with col1:
                    frameST.image(img, channels='BGR')
                with col2:
                    frameST2.image(depthFrameColor)

                noFrames += 1

                if cv2.waitKey(1) & 0xFF == ord('q'):
                    break
```

## 3. Package Contour Detection
To detect contours we apply first filters and canny detections
<img src='https://github.com/vilaksh01/DepthAI-LazyProjects/blob/main/Projects/PackagingBox%20Measurement/Images/Screenshot%20from%202021-05-05%2007-02-08.png'>

```python
# perform blurring, edge detection, dilation and erode to find contours
            imgBlur = cv2.GaussianBlur(img, (7, 7), 0)
            imgGray = cv2.cvtColor(imgBlur, cv2.COLOR_BGR2GRAY)
            imgCanny = cv2.Canny(imgGray, 50, 100)
            kernel = np.ones((5, 5))
            edged = cv2.dilate(imgCanny, kernel, iterations=1)
            edged = cv2.erode(edged, None, iterations=1)

            # call find contours to get all contours in image
            cnts = cv2.findContours(edged.copy(), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_NONE)
            cnts = imutils.grab_contours(cnts)
            if len(cnts) > 0:
                cnts = contours.sort_contours(cnts)[ 0 ]
            # color for each edge in case of rectangular bounding box
            colors = ((0, 0, 255), (240, 0, 159), (255, 0, 0), (255, 255, 0))

            # loop over the contours individually
            for (i, c) in enumerate(cnts):

                # if the contour is not sufficiently large, ignore it
                if cv2.contourArea(c) < 1000:
                    continue
```
