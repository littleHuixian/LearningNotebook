# OpenCV with Qt学习笔记

# 环境配置

linux/mac环境配置

```c++
INCLUDEPATH += /usr/local/include \
               /usr/local/include/opencv4 \
               /usr/local/include/opencv4/opencv2

LIBS += /usr/local/lib/libopencv_*.so
```

windows环境配置

```C++
INCLUDEPATH += D:\OpenCV451\buildForQt\install\include\
               D:\OpenCV451\buildForQt\install\include\opencv2

LIBS += -L D:\OpenCV451\buildForQt\install\x64\mingw\lib\libopencv_*.a
```

# 编译方法

```C++
g++ showEagle.cpp -o showEagle `pkg-config --cflags --libs opencv4`

g++ showImg.cpp -o showImg `pkg-config --libs --cflags opencv4`
```



# 使用OpenCV的注意事项

## 1. 读取一张图片并显示

```C++
int main(){
    //load images
    Mat img = imread("/home/huixian/Documents/images/test.jpg");
    if (!img.data) {
        printf("could not load image...\n");
        return;
    }
    //Mat的图片是BGR格式，需要先转换为RGB格式
    cvtColor(img, img, COLOR_BGR2RGB);
    //在窗口控件中显示图片
    myQImg = QImage((const unsigned char*)(img.data), img.cols, img.rows, img.step, QImage::Format_RGB888);
	//显示图片的几种调用方式
    imageShow();
    resizeEvent(QResizeEvent *event);
}

void MainWindow::imageShow()
{
    ui->labelShow->setPixmap(QPixmap::fromImage(myQImg.scaled(ui->labelShow->size(), Qt::KeepAspectRatio, Qt::SmoothTransformation)));
}

void DialogTwo::resizeEvent(QResizeEvent *event)
{
    //在窗口控件中显示图片
    originalQImg = QImage((const unsigned char*)(myImg.data), myImg.cols, myImg.rows, myImg.step, QImage::Format_RGB888);

    ui->labelImage1->setPixmap(QPixmap::fromImage(originalQImg.scaled(ui->labelImage1->size(), Qt::KeepAspectRatio, Qt::SmoothTransformation)));

}
```

## 2. 图片的操作

​		图片格式拷贝：图片的强制拷贝使用clone函数，避免对拷贝图片处理时影响原图片。

```C++
processedimg = out.clone();
```

​		图片格式转换：OpenCV库处理的图片格式是BGR格式的，当Qt窗口需要显示时要转换为RGB格式，否则会有运行错误。

```C++
Mat img2;
std::string path = filename.toLocal8Bit().toStdString();  //解决中文路径问题
img = cv::imread(path);   //以OpenCV方式加载图片
img2 = img;
cvtColor(img2, img2, COLOR_BGR2RGB);//OpenCV中Mat是BGR格式，需先转换为RGB格式
// cvtColor(img2, img2, COLOR_BGR2RGB);
```

​		彩色图片一般选用RGB888格式，灰色图片一般选用indexd8格式。

```C++
//在窗口控件中显示图片
QImage Qtemp = QImage((const unsigned char*)(img2.data), img2.cols, img2.rows, img2.step, QImage::Format_RGB888);

QImage Qtemp2 = QImage((const unsigned char*)(biImage.data), biImage.cols, biImage.rows, biImage.cols*biImage.channels() , QImage::Format_RGB888));
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
```

​		同理，当需要使用OpenCV库处理图像时，需要将原本是RGB格式的图片转换为BGR格式。

```C++
processedimg = out.clone();
cvtColor(processedimg, processedimg, COLOR_RGB2BGR);  //Mat图片是BGR格式，需先转换为RGB格式
```



## 3. 在Qt窗口中显示处理后的图片

​		使用QLabel控件作为图片显示的载体，再将Mat格式的图片转换为QImage格式，最后将QImage转换为QPixmap进行缩放显示。

```C++

QImage Qtemp2 = QImage((const unsigned char*)(out.data), out.cols, out.rows, out.cols()*out.channels(), QImage::Format_RGB888);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
//按比例缩放
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

​		注意：Mat格式的图片转换为QImage格式时，对于灰度图和彩色图是不同的。通过channels()函数进行区分。

彩色通道：

```C++
QImage Qtemp2 = QImage((const unsigned char*)(out.data), out.cols, out.rows, out.cols()*out.channels(), QImage::Format_RGB888);
```

单通道：

```C++
QImage Qtemp2 = QImage((const unsigned char*)(biImage.data), biImage.rows, biImage.cols, biImage.cols*biImage.channels() , QImage::Format_Indexed8));
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
```



```C++
void MainWindow::importFrame()
{
    if(isCamera==0){
        capture >> frame;
        if (!frame.empty()) {
            //only RGB of Qt
            cvtColor(frame, frame, COLOR_BGR2RGB);
            QImage srcQImage = QImage((uchar*)(frame.data), frame.cols, frame.rows, QImage::Format_RGB888);
            ui->labelShow->setPixmap(QPixmap::fromImage(srcQImage));
            ui->labelShow->resize(srcQImage.size());
            ui->labelShow->show();
        }

    }
}
```

## 答题卡线的检测

```C++
void MainWindow::on_btnProcess_clicked()
{
    Mat grayImg, binaryImg, morhpImg;
    cvtColor(srcImg, grayImg, COLOR_BGR2GRAY);

    threshold(grayImg, binaryImg, 0, 255, THRESH_BINARY_INV|THRESH_OTSU);

    Mat kernel = getStructuringElement(MORPH_RECT, Size(20, 1), Point(-1, -1));
    morphologyEx(binaryImg, morhpImg, MORPH_OPEN, kernel, Point(-1, -1));

    kernel = getStructuringElement(MORPH_RECT, Size(3, 3), Point(-1, -1));
    dilate(morhpImg, morhpImg, kernel);

    //hough lines
    vector<Vec4i>lines;
    HoughLinesP(morhpImg, lines, 1, CV_PI/180.0, 5, 10, 5);
    Mat resultImg = srcImg.clone();
    cvtColor(resultImg, resultImg, COLOR_BGR2RGB);

    for (size_t t = 0; t<lines.size(); t++) {
        Vec4i ln = lines[t];
        line(resultImg, Point(ln[0], ln[1]), Point(ln[2], ln[3]), Scalar(0, 255, 255), 2, LINE_AA);
    }

    showImage(resultImg);
}
```



# 读取视频

## 读取视频文件并采用高斯背景消元法和二帧差法检测移动目标

示例1：

videoplay.h

```C++
#pragma once

#include <QtWidgets/QMainWindow>
#include <opencv2/opencv.hpp>
#include <QTimer>

using namespace cv;
using namespace std;

class videodisplayinqt : public QMainWindow
{
    Q_OBJECT

public:
    videodisplayinqt(QWidget *parent = Q_NULLPTR);

private:
    //Ui::videodisplayinqtClass ui;
    Ui::videodisplayinqt ui;
    VideoCapture capture;
    QTimer *timer;
    Mat frame;

private slots:
    void importFrame();
    void on_displayButton_clicked();
    void on_stopButton_clicked();
};
```

videoplay.cpp

```C++
#include "videodisplayinqt.h"

videodisplayinqt::videodisplayinqt(QWidget *parent) : QMainWindow(parent)
{
    ui.setupUi(this);

    this->setWindowTitle("video play with Qt");
    timer = new QTimer(this);
    //import frame when timeout
    connect(timer, SIGNAL(timeout()), this, SLOT(importFrame()));
}

void videodisplayinqt::importFrame()
{
    capture >> frame;
    if (!frame.empty()) {
        //only RGB of Qt
        cvtColor(frame, frame, COLOR_BGR2RGB);
        QImage showImg = QImage((const unsigned char*)(frame.data), frame.cols, frame.rows, frame.step,QImage::Format_RGB888);
        ui->labelPlay->setPixmap(QPixmap::fromImage(showImg.scaled(ui->labelPlay->size(), Qt::KeepAspectRatio, Qt::SmoothTransformation)));
    }
}

//帧间差分法目标检测、背景消去法目标检测、光流法目标检测
void videodisplayinqt::on_displayButton_clicked()
{
    if (isCamera==0) {
        capture.open("/home/huixian/Documents/images/bike.avi");
        //获取当前视频帧率
        double rate = capture.get(CAP_PROP_FPS);
        //每一帧之间的延时与视频的帧率相对应
        double m_delay = 1000 / rate;
        //开启定时器
        timer->start(m_delay);
	//自适应高斯混合背景建模（MOG2）的帧差法
    } else if (isCamera==1) {
        // 打开视频文件
        cv::VideoCapture capture("bike.avi");

        // 当前视频帧
        cv::Mat frame;
        // 前景的二值图像
        cv::Mat foreground;
        // 背景图像
        cv::Mat background;
        cv::namedWindow("Extracted Foreground");
        // 混合高斯模型类的对象，全部采用默认参数
        cv::Ptr<cv::BackgroundSubtractorMOG2> ptrMOG = cv::createBackgroundSubtractorMOG2(500, 16, false);
        // cv::bgsegm::createBackgroundSubtractorMOG();
        // cv::BackgroundSubtractor::apply
        bool stop(false);
        // 遍历视频中的所有帧
        while (!stop) {
            // 读取下一帧（如果有）
            if (!capture.read(frame))
                break;

            // 更新背景并返回前景
            ptrMOG->apply(frame, foreground, 0.05);

            // 改进图像效果
            cv::threshold(foreground, foreground, 30, 255, cv::THRESH_BINARY);
            //4.腐蚀
            //Mat kernel_erode = getStructuringElement(MORPH_RECT, Size(7, 7));
            //Mat kernel_dilate = getStructuringElement(MORPH_RECT, Size(21, 21));
            //erode(foreground, foreground, kernel_erode);
            //imshow("erode", diff_thresh);
            //5.膨胀
            //dilate(foreground, foreground, kernel_dilate);
            //均值滤波，去除椒盐噪声
            //medianBlur(foreground, foreground,11);
            // 显示前景和背景
            cv::imshow("Extracted Foreground", foreground);

            // 产生延时，或者按键结束
            if (cv::waitKey(10) >= 0)
                stop = true;
        }

    } else {
        // 打开视频文件
        cv::VideoCapture capture("bike.avi");
        // 检查是否成功打开视频
        //if (!capture.isOpened())
        //    return 0;
        // 当前视频帧
        cv::Mat frame;
        // 前景的二值图像
        cv::Mat foreground;
        // 背景图像
        cv::Mat background;
        //cv::namedWindow("Extracted Foreground");
        // 混合高斯模型类的对象，全部采用默认参数
        cv::Ptr<cv::BackgroundSubtractorMOG2> ptrMOG = cv::createBackgroundSubtractorMOG2(500, 16, false);
        // cv::bgsegm::createBackgroundSubtractorMOG();
        // cv::BackgroundSubtractor::apply
        bool stop(false);
        // 遍历视频中的所有帧
        while(!stop) {
            // 读取下一帧（如果有）
            if (!capture.read(frame))
                break;
            Mat result = frame.clone();
            // 更新背景并返回前景
            ptrMOG->apply(frame, foreground, 0.03);
            // 改进图像效果
            cv::threshold(foreground, foreground, 25, 255, cv::THRESH_BINARY);
            //4.腐蚀
            Mat kernel_erode = getStructuringElement(MORPH_RECT, Size(5, 5));
            Mat kernel_dilate = getStructuringElement(MORPH_RECT, Size(11, 11));
            erode(foreground, foreground, kernel_erode);
            //imshow("erode", diff_thresh);
            //5.膨胀
            dilate(foreground, foreground, kernel_dilate);
            //均值滤波，去除椒盐噪声
            //medianBlur(foreground, foreground,11);
            //6.查找轮廓并绘制轮廓
            vector<vector<Point> > contours;
            findContours(foreground, contours, RETR_EXTERNAL, CHAIN_APPROX_NONE);
            for (int i = 0; i < contours.size(); i++) {
                //在result上绘制正外接矩形
                rectangle(result, boundingRect(contours[i]), Scalar(0, 255, 0), 1);
            }
            // 显示前景和背景
            cv::imshow("Extracted Foreground", foreground);
            // imshow("result", result);
            // 产生延时，或者按键结束
            if (cv::waitKey(10) >= 0)
                stop = true;
        }
    } 
    //    timer->start(20);// Start timing, Signal out when timeout
}

void videodisplayinqt::on_stopButton_clicked()
{
    timer->stop();
    capture.release();
}
```

示例2

```C++
#include <iostream>
#include <opencv2/opencv.hpp>
using namespace std;
using namespace cv;
//帧差法检测车辆
Mat moveCheck(Mat &forntFrame,Mat &afterFrame)
{
    Mat frontGray,afterGray,diff;
    Mat resFrame=afterFrame.clone();
    //灰度处理
    cvtColor(forntFrame,frontGray,CV_BGR2GRAY);
    cvtColor(afterFrame,afterGray,CV_BGR2GRAY);
    //帧差处理 找到帧与帧之间运动物体差异
    absdiff(frontGray,afterGray,diff);
    //imshow("diff",diff);
 
    //二值化
    threshold(diff,diff,25,255,CV_THRESH_BINARY);
    //imshow("threashold",diff);
    //腐蚀处理：
    Mat element=cv::getStructuringElement(MORPH_RECT,Size(3,3));
    erode(diff,diff,element);
    //imshow("erode",diff);
    //膨胀处理
    Mat element2=cv::getStructuringElement(MORPH_RECT,Size(20,20));
    dilate(diff,diff,element2);
    //imshow("dilate",diff);
 
    //动态物体标记
    vector<vector<Point>>contours;//保存关键点
    findContours(diff,contours,CV_RETR_EXTERNAL,CV_CHAIN_APPROX_SIMPLE,Point(0,0));
 
    //提取关键点
    vector<vector<Point>>contour_poly(contours.size());
    vector<Rect>boundRect(contours.size());
 
    int x,y,w,h;
    int num=contours.size();
 
    for(int i=0;i<num;i++) {
        approxPolyDP(Mat(contours[i]),contour_poly[i],3,true);
        boundRect[i]=boundingRect(Mat(contour_poly[i]));
 
        x=boundRect[i].x;
        y=boundRect[i].y;
        w=boundRect[i].width;
        h=boundRect[i].height;
        //绘制
        rectangle(resFrame,Point(x,y),Point(x+w,y+h),Scalar(0,255,0),2);
    }
    return resFrame;
}
 
int main(int argc, char *argv[])
{
    Mat frame;
    Mat temp;
    Mat res;
    int count=0;
    VideoCapture cap("C:/Users/15123/Pictures/Camera Roll/carMove.mp4");
    while(cap.read(frame)){
          count++;
          if(count==1)
          {
              res=moveCheck(frame,frame);
 
          }
          else
          {
              res=moveCheck(temp,frame);
 
          }
          temp=frame.clone();
          imshow("frame",frame);
          imshow("res",res);
          waitKey(25);
    }
    return 0;
}
```

## 三帧差法

为了提高帧差法的鲁棒性和稳定性，又出现了三帧差分法这种改进方法。

三帧差分法是将连续的三帧图像，分别进行转灰度图、高斯模糊消除噪声干扰，然后进行逐帧相减，也就是后一帧图像减去当前帧图像、当前帧图像减去前一帧图像，从而得到两张差异图像。再将得到的两个差值图像进行与操作，得到共同的差异区域，最后通过开运算操作消除微小干扰。这样就得到了三帧图像间的明显差异区域，也就是运动的目标物体。

而且二帧差分法对于微小运动物体的检测能力比较差，因为如果在两帧图像之间变化太小，就很难被检测出来。而三帧差分法利用连续三帧图像的差异结果，能够提高对微小运动物体的检测能力，同时增强对噪声、光照等因素的抗干扰能力。

```C++
VideoCapture capture;
capture.open("D:\\opencv_c++\\opencv_tutorial\\data\\images\\bike.avi");
//capture.open("D:\\OpenCV\\opencv\\sources\\samples\\data\\vtest.avi");
if (!capture.isOpened())
{
    return 0;
}
Mat pre_frame1, pre_frame2, current_frame;
capture.read(pre_frame1);
capture.read(pre_frame2);
Mat pre_gray1, pre_gray2, current_gray;
cvtColor(pre_frame1, pre_gray1, COLOR_BGR2GRAY);
cvtColor(pre_frame2, pre_gray2, COLOR_BGR2GRAY);
Mat pre_gaus1, pre_gaus2, current_gaus;
GaussianBlur(pre_gray1, pre_gaus1, Size(), 10, 0);
GaussianBlur(pre_gray2, pre_gaus2, Size(), 10, 0);

while (capture.read(current_frame)){
    cvtColor(current_frame, current_gray, COLOR_BGR2GRAY);
    GaussianBlur(current_gray, current_gaus, Size(), 10, 0);
    Mat diff1, diff2, diff;
    subtract(pre_gaus2, pre_gaus1, diff1);
    subtract(current_gaus, pre_gaus2, diff2);
    Mat diff1_binary, diff2_binary;
    threshold(diff1, diff1_binary, 0, 255, THRESH_BINARY | THRESH_OTSU);
    threshold(diff2, diff2_binary, 0, 255, THRESH_BINARY | THRESH_OTSU);
    bitwise_and(diff1_binary, diff2_binary, diff);
    Mat kernel = getStructuringElement(MORPH_RECT, Size(3, 3));
    morphologyEx(diff, diff, MORPH_OPEN, kernel, Point(-1, -1), 1, 0);
    imshow("diff", diff);
    imshow("current_frame", current_frame);
    char ch = cv::waitKey(20);
    if (ch == 27){
        break;
    }
    pre_gaus1 = pre_gaus2.clone();
    pre_gaus2 = current_gaus.clone();
}
```

从视频可以看出，对同一视频进行运动检测，三帧差法的检测效果要优于二帧差法，对于运动目标的检测更稳定，而且没有出现误检的情况。

可见三帧差分法的性能是要优于二帧差分法的，而且由于帧差法本身的运算规模并不算特别庞大，所以运行速度还是比较理想的，对于二帧差和三帧差之间的运行速度虽然有差距，但是在视频播放过程中均没有出现明显的卡顿现象。

# OpenCV实现最小二乘法

```C++
#include<iostream>
#include<opencv2\opencv.hpp>

using namespace std;
using namespace cv;

int main(){
    vector<Point>points;
    //(27 39) (8 5) (8 9) (16 22) (44 71) (35 44) (43 57) (19 24) (27 39) (37 52)
    points.push_back(Point(27, 39));
    points.push_back(Point(8, 5));
    points.push_back(Point(8, 9));
    points.push_back(Point(16, 22));
    points.push_back(Point(44, 71));
    points.push_back(Point(35, 44));
    points.push_back(Point(43, 57));
    points.push_back(Point(19, 24));
    points.push_back(Point(27, 39));
    points.push_back(Point(37, 52));
    Mat src = Mat::zeros(400, 400, CV_8UC3);

    for (int i = 0;i < points.size();i++) {
        //在原图上画出点
        circle(src, points[i], 3, Scalar(0, 0, 255), 1, 8);
    }
    //构建A矩阵
    int N = 2;
    Mat A = Mat::zeros(N, N, CV_64FC1);

    for (int row = 0;row < A.rows;row++) {
        for (int col = 0; col < A.cols;col++) {
            for (int k = 0;k < points.size();k++){
                A.at<double>(row, col) = A.at<double>(row, col) + pow(points[k].x, row + col);
            }
        }
    }
    //构建B矩阵
    Mat B = Mat::zeros(N, 1, CV_64FC1);
    for (int row = 0;row < B.rows;row++){
        for (int k = 0;k < points.size();k++) {
            B.at<double>(row, 0) = B.at<double>(row, 0) + pow(points[k].x, row)*points[k].y;
        }
    }
    //A*X=B
    Mat X;
    //cout << A << endl << B << endl;
    solve(A, B, X,DECOMP_LU);
    cout << X << endl;
    vector<Point>lines;
    for (int x = 0;x < src.size().width;x++){
        // y = b + ax;
        double y = X.at<double>(0, 0) + X.at<double>(1, 0)*x;
        printf("(%d,%lf)\n", x, y);
        lines.push_back(Point(x, y));
    }
    polylines(src, lines, false, Scalar(255, 0, 0), 1, 8);
    imshow("src", src);

    //imshow("src", A);
    waitKey(0);
    return 0;
}
```



# 图像的遍历操作

​		对图像的遍历操作（主要分为指针遍历和[数组](https://so.csdn.net/so/search?q=数组&spm=1001.2101.3001.7020)遍历）和对像素的算术操作。
1、首先是使用数组来遍历图像

```C++
/********************数组遍历像素点********************/
int height = image.rows;
int width = image.cols;
int ch = image.channels();
for (int row = 0; row < height; row++){
    for (int col = 0; col < width; col++){
        if (ch == 3) {
            //定义一个包含bgr三通道的向量，存放在图像该点的值
            Vec3b bgr = image.at<Vec3b>(row, col);
            bgr[0] = 255 - bgr[0];
            bgr[1] = 255 - bgr[1];
            bgr[2] = 255 - bgr[2];
            m4.at<Vec3b>(row, col) = bgr;
            
        } else if (ch == 1) {
            int gray;
            gray = image.at<uchar>(row, col);
            m4.at<uchar>(row, col) = 255 - gray;
        }
    }
}
```

​		首先获取了图像的高度、宽度和通道数，分别使用height、width和channels三个变量来保存，然后使用两个for循环分别对图像的高和宽进行遍历。
​		当遍历到一个像素点时，先判断这幅图像是三通道还是单通道的，如果是三通道图，就用Vec3b定义一个可以存放B、G、R三个数据的向量，用来存放该点的像素值。然后分别对三通道去取反，也就是用255减去该像素值，再将相减后的值赋给图像的该点。在这里使用到了Mat.at<_type>(row, col)，意思是把一个Mat对象在(row, col)这个点处的像素值，通过<_type>指定的格式返回，例如使用Vec3b,所以返回的是一个三维向量。
​		在单通道图的操作也是大同小异，其中

```C++
gray = image.at<uchar>(row, col);
```

​		表示将该点处的像素值以uchar类型返回，并赋给变量gray。

2、接下来是指针遍历图像像素点

```C++
/********************指针遍历像素点********************/
int height = image.rows;
int width = image.cols;
int ch = image.channels();
for (int row = 0; row < height; row++){
    uchar* currentRow = image.ptr<uchar>(row);			//获取当前行的头指针
    uchar* resultRow = m4.ptr<uchar>(row);			//保存结果图像的当前行的头指针
    for (int col = 0; col < width; col++) {
        if (ch == 3) {
            int blue = *currentRow;			//每个元素为b、g、r三通道
            currentRow++;
            int green = *currentRow;
            currentRow++;
            int red = *currentRow;
            currentRow++;						//遍历完三通道指针指向下一个像素点

            *resultRow = 255 - blue;
            resultRow++;
            *resultRow = 255 - green;
            resultRow++;
            *resultRow = 255 - red;
            resultRow++;
            
        } else if (ch == 1) {
            int gray = *currentRow;
            currentRow++;
            *resultRow = gray;
            resultRow++;
        }
    }
}
```

​		同样是先获取图像的高、宽和通道数，再使用两个for循环对图像的高和宽进行遍历。不同的是，在对每一行循环时，会使用uchar* currentRow = image.ptr<uchar>(row); uchar* resultRow = m4.ptr<uchar>(row);这两行代码来分别获取原图像和结果图像中的当前行指针。行指针指向的是每一行的头部。
​		在获取到当前行指针后，再对图像的列数进行遍历，通过指针的移动来逐个获取当前行中的每一列的数据。对于三通道图，一个像素有B、G、R三个值，所以要获取一个像素的值，需要遍历这三个通道。

​		总结：这两种遍历方式都能对图像的每个像素点进行遍历，使用数组遍历方式较容易理解，也比较容易实现，但是没有使用指针时运行那么快；而使用指针遍历方式需要比较好地理解指针所指向的是什么，代码上来实现也比较麻烦，但是优势是执行速度快。

3、对像素的算术操作

```C++
/********************像素算术操作********************/
Mat image1;
image1 = imread("D:/opencv_c++/LinuxLogo.jpg");
Mat image2 = imread("D:/opencv_c++/WindowsLogo.jpg");

int height1 = image1.rows;
int width1 = image1.cols;
int ch1 = image1.channels();
int height2 = image2.rows;
int width2 = image2.cols;
int ch2 = image2.channels();
if (height1 != height2 || width1 != width2 || ch1 != ch2){
    return -1;
}

Mat dst = Mat::zeros(image1.size(), image1.type());
for (int row = 0; row < height1; row++){
    uchar* currentRow1 = image1.ptr<uchar>(row);
    uchar* currentRow2 = image2.ptr<uchar>(row);
    uchar* resultRow = dst.ptr<uchar>(row);
    for (int col = 0; col < width1; col++){
        if (ch1 == 3) {
            int b1 = *currentRow1++;
            int g1 = *currentRow1++;
            int r1 = *currentRow1++;

            int b2 = *currentRow2++;
            int g2 = *currentRow2++;
            int r2 = *currentRow2++;

            /*Vec3b bgr_result = dst.at<Vec3b>(row, col);
				bgr_result[0] = saturate_cast<uchar>(b1 + b2);
				bgr_result[1] = saturate_cast<uchar>(g1 + g2);
				bgr_result[2] = saturate_cast<uchar>(r1 + r2);
				dst.at<uchar>(row, col) = bgr_result;*/
            *resultRow = saturate_cast<uchar>(b1 + b2);
            resultRow++;
            *resultRow = saturate_cast<uchar>(g1 + g2);
            resultRow++;
            *resultRow = saturate_cast<uchar>(r1 + r2);
            resultRow++;
            
        } else if (ch1 == 1) {
            int gray1 = *currentRow1++;
            int gray2 = *currentRow2++;

            *resultRow = saturate_cast<uchar>(gray1 + gray2);
            resultRow++;
        }
    }
}
```

​		首先读取两张图像，然后获取这两张图像的宽高和通道数并进行比对，只有两张图像的宽高和通道数都相同才可以进行像素的算术操作。
​		对像素进行算术操作，同样需要对图像遍历像素点，也可以选择数组或者指针的方式来遍历。在上述代码中两种方式都有，其中被注释掉的是用数组方式的遍历。
​		当访问某个坐标时，获取两张图像在该坐标的像素值，并进行算术操作，例如加法运算：*resultRow = saturate_cast<uchar>(gray1 + gray2);
​		进行算术操作时，要注意结果值的范围，两个[0, 255]的值相加，其结果可能会超过[0, 255]，所以我们不能直接将计算结果赋给像素点。可以用saturate_cast<uchar>这个函数来对计算结果映射为uchar类型，也就是映射到[0, 255]之间，再赋值给像素点，这样子可以防止像素值溢出，显示的时候才能正常显示。
​		上面的代码是通过遍历像素点来实现算术操作，实际上OpenCV提供了一些简单的API来实现这些功能，如以下代码：

```C++
add(image1, image2, dst);
subtract(image1, image2, dst);
multiply(image1, image2, dst);
divide(image1, image2, dst);
```

​		这四个函数分别实现了像素的加、减、乘、除四种基本的算术操作，其参数也非常简单，两个输入图像变量和一个输出图像变量即可。



# 照片素描处理

1.图像灰度化、2.负片处理、3.高斯模糊、4.淡化过程、5.加深过程

```C++
Mat srcImg = imread("E:\\CppFiles\\CppWithOpencv\\test.jpg");
if (!srcImg.data) {
    qDebug()<<"could not load image...\n";
    return ;
}
//完成照片素描
Mat grayImg, bitImg, gaussionImg;
cvtColor( srcImg,  grayImg, COLOR_BGR2GRAY);
//反色处理
bitwise_not(grayImg, bitImg);
//高斯模糊
GaussianBlur(bitImg,  gaussionImg, Size(11, 11), 0, 0);
Mat lightImg, deepImg;
//淡化处理
divide(grayImg, gaussionImg, lightImg, 256);
//加深处理
divide(255-lightImg, 255-gaussionImg, deepImg, 256);
finalImg = 255 - deepImg;

showImg = finalImg.clone();
//    cvtColor( showImage,  showImage, COLOR_BGR2RGB);
QImage originalQImg = QImage((const unsigned char*)(showImg.data), showImg.cols, showImg.rows, showImg.step, QImage::Format_Indexed8);

ui->labelShow->setPixmap(QPixmap::fromImage(originalQImg.scaled(ui->labelShow->size(), Qt::KeepAspectRatio, Qt::SmoothTransformation)));

```



# 1. 使用OpenCV库实现人脸识别

​		这里的人脸识别使用的是级联分类器haarcascades，在机器学习算法中，通过对多个弱分类器的叠加可以实现一个强分类器。使用级联分类器，识别速度快，识别准确率也较高，非常适合于物体识别。本文用到的级联分类器是由opencv官方训练好的一个用于人脸识别的级联分类器，可以分别识别人的眼睛和鼻子以及其他人脸部位，分类器由一个xml文件保存，使用时导入即可。

​		首先，将图片灰度化，再直方图均衡化，增强图片对比度；

​		然后，载入用到的级联分类器模型；

​		最后，使用载入的分类器进行人脸（或者其他部位）识别，识别到的所有人脸（或者其他部位），都会被一个正方形框住。这些正方形组成一个数组，正方形有位置和面积大小，通过人为设置判断标准，可以将人脸可靠的检测出来。

```C++
Mat faceimage=img.clone();//深层拷贝
Mat image_gray;

cvtColor(faceimage, image_gray, CV_BGR2GRAY);   //转为灰度图
equalizeHist(image_gray, image_gray);       //直发图均化，增强对比度方便处理

CascadeClassifier eye_Classifier;            //载入分类器
CascadeClassifier face_cascade;              //载入分类器

//加载分类训练器，OpenCV官方文档的xml文档，可以直接调用
QString aFile = QDir::currentPath() + "/haarcascade_eye.xml";
QString path = QDir::toNativeSeparators(aFile);
string eyefile = path.toStdString();
//把xml文档复制到了当前项目的路径下
if (!eye_Classifier.load(eyefile)){
    qDebug() << "导入haarcascade_eye.xml时出错 !" << endl;
    return;
}

QString aFile2 = QDir::currentPath() + "/haarcascade_frontalface_alt.xml";
QString path2 = QDir::toNativeSeparators(aFile2);
string facefile = path2.toStdString();
//把xml文档复制到了当前项目的路径下
if (!face_cascade.load(facefile)) {
    qDebug() << "导入haarcascade_frontalface_alt.xml时出错 !" << endl;
    return;
}

vector<Rect> faces, eyes;
face_cascade.detectMultiScale(image_gray, faces, 1.2, 5, 0, Size(30, 30));

if (faces.size()>0) {
    for (size_t i = 0; i<faces.size(); i++) {
        rectangle(faceimage, Point(faces[i].x, faces[i].y), Point(faces[i].x + faces[i].width, faces[i].y + faces[i].height), Scalar(0, 0, 255), 10, 8);
    }
}

processedimg=faceimage.clone();
cvtColor(processedimg, processedimg, CV_RGB2BGR);//Mat的图片是BGR格式，需要先转换为RGB格式
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(faceimage.data), faceimage.cols, faceimage.rows,faceimage.cols*faceimage.channels(), QImage::Format_RGB888);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
// 按比例缩放
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

# 2. 图像的直方图分析

​		一张彩色图像的直方图分析包括：R分量，G分量，B分量和灰色分量的直方图分析

## 2.1 各个分量的直方图分析

​		将一副图像的各个分量通道进行分离，然后进行直方图分析

```C++
Mat histimg;

vector<Mat> rgb_channel;
split(img, rgb_channel);//将一幅多通道的图像的各个通道分离。
Mat R = rgb_channel[2];
histimg = getHistograph(R);

processedimg = histimg;
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(histimg.data), histimg.cols, histimg.rows, histimg.cols*histimg.channels(),QImage::Format_Grayscale8);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

## 2.2 灰度直方图分析

​		灰度直方图分析就是将图片转换为灰度图，然后进行直方图分析

```C++
Mat hsv;
//定义灰度图像，转成灰度图
cvtColor(img,hsv,COLOR_BGR2GRAY);
//直方图图像
Mat hist=getHistograph(hsv);

processedimg=hist;
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(hist.data), hist.cols, hist.rows, hist.cols*hist.channels(),QImage::Format_Grayscale8);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

## 2.3 直方图分析的普遍算法

```C++
//直方图提取算法
Mat getHistograph(const Mat grayImage)
{
  //定义求直方图的通道数目，从0开始索引
  int channels[]={0};
  //定义直方图的在每一维上的大小，例如灰度图直方图的横坐标是图像的灰度值，就一维，bin的个数
  //如果直方图图像横坐标bin个数为x，纵坐标bin个数为y，则channels[]={1,2}其直方图应该为三维的，Z轴是每个bin上统计的数目
  const int histSize[]={256};
  //每一维bin的变化范围
  float range[]={0,256};

  //所有bin的变化范围，个数跟channels应该跟channels一致
  const float* ranges[]={range};

  //定义直方图，这里求的是直方图数据
  Mat hist;
  //opencv中计算直方图的函数，hist大小为256*1，每行存储的统计的该行对应的灰度值的个数
  calcHist(&grayImage,1,channels,Mat(),hist,1,histSize,ranges,true,false);//cv中是cvCalcHist

  //找出直方图统计的个数的最大值，用来作为直方图纵坐标的高
  double maxValue=0;
  //找矩阵中最大最小值及对应索引的函数
  minMaxLoc(hist,0,&maxValue,0,0);
  //最大值取整
  int rows=cvRound(maxValue);
  //定义直方图图像，直方图纵坐标的高作为行数，列数为256(灰度值的个数)
  //因为是直方图的图像，所以以黑白两色为区分，白色为直方图的图像
  Mat histImage=Mat::zeros(rows,256,CV_8UC1);

  //直方图图像表示
  for(int i=0;i<256;i++){
    //取每个bin的数目
    int temp=(int)(hist.at<float>(i,0));
    //如果bin数目为0，则说明图像上没有该灰度值，则整列为黑色
    //如果图像上有该灰度值，则将该列对应个数的像素设为白色
    if(temp){
      //由于图像坐标是以左上角为原点，所以要进行变换，使直方图图像以左下角为坐标原点
      histImage.col(i).rowRange(Range(rows-temp,rows))=255;
    }
  }
  //由于直方图图像列高可能很高，因此进行图像对列要进行对应的缩减，使直方图图像更直观
  Mat resizeImage;
  resize(histImage,resizeImage,Size(256,256));
  return resizeImage;
}
```

# 3. 图像形态学处理-腐蚀和膨胀

​		腐蚀和膨胀操作关键点是：内核参数的选取，这个是人为确定的。

​		腐蚀操作如下：

```C++
//获取内核形状和尺寸
int m_KernelValue=1;
Mat element = getStructuringElement(MORPH_RECT, Size(m_KernelValue * 2 + 1, m_KernelValue * 2 + 1), Point(m_KernelValue, m_KernelValue));

//腐蚀操作
Mat m_dstImage;
erode(img, m_dstImage, element);

processedimg=m_dstImage.clone();
cvtColor(processedimg,processedimg,CV_RGB2BGR);//Mat的图片是BGR格式，需要先转换为RGB格式
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(m_dstImage.data), m_dstImage.cols, m_dstImage.rows, m_dstImage.cols*m_dstImage.channels(),QImage::Format_RGB888);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

膨胀操作如下：

```C++
//获取内核形状和尺寸
int m_KernelValue=1;
Mat element = getStructuringElement(MORPH_RECT, Size(m_KernelValue * 2 + 1, m_KernelValue * 2 + 1), Point(m_KernelValue, m_KernelValue));

//膨胀操作
Mat m_dstImage;
dilate(img, m_dstImage, element);

processedimg=m_dstImage.clone();
cvtColor(processedimg,processedimg,CV_RGB2BGR);//Mat的图片是BGR格式，需要先转换为RGB格式
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(m_dstImage.data), m_dstImage.cols, m_dstImage.rows,m_dstImage.cols*m_dstImage.channels(), QImage::Format_RGB888);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

# 4. 图像的线性滤波和非线性滤波

​		这里介绍图像的线性滤波：方框滤波，均值滤波和高斯滤波。以及图像的非线性滤波：中值滤波和双边滤波

## 4.1 方框滤波

滤波操作非常简单，重要的是滤波函数内的参数人为选取

```C++
//进行滤波操作
Mat out;
boxFilter(img, out, -1, Size(3, 3));

//显示
processedimg=out.clone();
cvtColor(processedimg,processedimg,CV_RGB2BGR);//Mat的图片是BGR格式，需要先转换为RGB格式

//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(out.data), out.cols, out.rows,out.cols*out.channels(), QImage::Format_RGB888);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

## 4.2 均值滤波

```C++
//进行滤波操作
Mat out;
blur(img,out,Size(7,7));

processedimg=out.clone();
cvtColor(processedimg,processedimg,CV_RGB2BGR);//Mat的图片是BGR格式，需要先转换为RGB格式
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(out.data), out.cols, out.rows, out.cols*out.channels(),QImage::Format_RGB888);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

## 4.3 高斯滤波

```C++
//进行滤波操作
Mat out;
GaussianBlur(img,out,Size(3,3),0,0);

processedimg=out.clone();
cvtColor(processedimg,processedimg,CV_RGB2BGR);//Mat的图片是BGR格式，需要先转换为RGB格式
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(out.data), out.cols, out.rows, out.cols*out.channels(),QImage::Format_RGB888);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

## 4.4 中值滤波

```C++
//进行中值滤波操作
Mat out;
medianBlur (img, out, 7);//输入，输出，7通道。其中参数3：孔径的线性尺寸，必须大于1.、必须为奇数，越大，滤布越强。

processedimg=out.clone();
cvtColor(processedimg,processedimg,CV_RGB2BGR);//Mat的图片是BGR格式，需要先转换为RGB格式
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(out.data), out.cols, out.rows,out.cols*out.channels(), QImage::Format_RGB888);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

## 4.5 双边滤波

```C++
//进行中值滤波操作
Mat out;
//双边滤波操作
bilateralFilter(img, out, 25, 25 * 2, 25 / 2);

processedimg=out.clone();
cvtColor(processedimg,processedimg,CV_RGB2BGR);//Mat的图片是BGR格式，需要先转换为RGB格式
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(out.data), out.cols, out.rows,out.cols*out.channels(), QImage::Format_RGB888);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

# 5. 图像的二值化，边缘提取以及轮廓分析

​		图像的二值化需要先将彩色图转换为灰度图，由于灰度范围是0-256，因此二值化处理时的值不能越过这个范围。

​		图像边缘提取算法有：sobel算子，Laplacian算子，Canny算子。

​		轮廓分析使用findContours函数，找到的轮廓是有面积值的，可以据此进行轮廓筛选。

## 5.1 图像的二值化

```C++
Mat biImage;
cvtColor(img, biImage, CV_BGR2GRAY);//彩色图转换为灰度图
cv::threshold(biImage,biImage,threshold,255, CV_THRESH_BINARY);//二值化处理

processedimg=biImage;//方便放大显示二值化处理后的图片
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(biImage.data), biImage.cols, biImage.rows,biImage.cols*biImage.channels(),QImage::Format_Indexed8);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

## 5.2 sobel算子边缘提取

```C++
Mat hsv,edgeImg;
//定义灰度图像，转成灰度图
cvtColor(img,hsv,COLOR_BGR2GRAY);
//Sobel边缘检测
Mat x_edgeImg, y_edgeImg;
Mat abs_x_edgeImg, abs_y_edgeImg;

/*****先对x方向进行边缘检测********/
//因为Sobel求出来的结果有正负，8位无符号表示不全，故用16位有符号表示
Sobel(hsv,x_edgeImg, CV_16S, 1, 0, 3, 1, 1, BORDER_DEFAULT);
convertScaleAbs(x_edgeImg, abs_x_edgeImg);//将16位有符号转化为8位无符号

/*****再对y方向进行边缘检测********/
Sobel(hsv, y_edgeImg, CV_16S, 0, 1, 3, 1, 1, BORDER_DEFAULT);
convertScaleAbs(y_edgeImg, abs_y_edgeImg);

addWeighted(abs_x_edgeImg, 0.5, abs_y_edgeImg, 0.5, 0, edgeImg);

processedimg=edgeImg;
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(edgeImg.data), edgeImg.cols, edgeImg.rows, edgeImg.cols*edgeImg.channels(),QImage::Format_Grayscale8);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

## 5.3 laplacian算子边缘提取

```C++
Mat hsv,edgeImg;
//定义灰度图像，转成灰度图
cvtColor(img,hsv,COLOR_BGR2GRAY);
//Laplacian边缘检测
Mat lapImg;
Laplacian(hsv, lapImg, CV_16S, 5, 1, 0, BORDER_DEFAULT);
convertScaleAbs(lapImg, edgeImg);

processedimg=edgeImg;
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(edgeImg.data), edgeImg.cols, edgeImg.rows, edgeImg.cols*edgeImg.channels(),QImage::Format_Grayscale8);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

## 5.4 canny算子边缘提取

```C++
Mat hsv,edgeImg;
//定义灰度图像，转成灰度图
cvtColor(img,hsv,COLOR_BGR2GRAY);
//Canny边缘检测
Canny(hsv, edgeImg, 30, 80);

processedimg=edgeImg;
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(edgeImg.data), edgeImg.cols, edgeImg.rows, edgeImg.cols*edgeImg.channels(),QImage::Format_Grayscale8);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

## 5.5 轮廓分析

​		contoursize数组中contoursize[i]就是轮廓i的面积。

```C++
Mat hsv;
//定义灰度图像，转成灰度图
cvtColor(img,hsv,COLOR_BGR2GRAY);
Mat contImg = Mat ::zeros(hsv.size(),CV_8UC3);//定义三通道轮廓提取图像

Mat binImg;
cv::threshold(hsv, binImg, 127, 255, THRESH_OTSU);//大津法进行图像二值化

vector<vector<Point>> contours;
vector<Vec4i> hierarchy;
//查找轮廓
findContours(binImg, contours, hierarchy, CV_RETR_CCOMP, CV_CHAIN_APPROX_NONE);
//绘制查找到的轮廓
drawContours(contImg, contours, -1, Scalar(0,255,0));

contoursize.clear();
contoursize.resize(1);//重置轮廓面积变量的大小

//    qDebug()<<contours.size();
for(int i=0;i<contours.size();i++){
  ui->contour->addItem(QString::number(i+1));
  double temp=contourArea(contours[i]);
  contoursize.push_back(temp);
}

ui->num->setText(QString::number(contours.size()));
ui->size->setText(QString::number(contourArea(contours[0])));

processedimg=contImg;
//在窗口控件中显示图片
QImage Qtemp2 = QImage((const unsigned char*)(contImg.data), contImg.cols, contImg.rows, contImg.cols*contImg.channels(),QImage::Format_RGB888);
QPixmap pixmap2 = QPixmap::fromImage(Qtemp2);
int width = ui->showimage->width();
int height = ui->showimage->height();
QPixmap fitpixmap2 = pixmap2.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation);  // 按比例缩放

ui->showimage->setPixmap(fitpixmap2);
ui->showimage->show();
```

# 6. 多线程处理图片

​		将图片从硬盘中读取是一个费时的操作，远远高于图片处理的耗时。因此通过多线程的方式，我们就可以在处理图片的同时，导入后续图片到内存中。需要注意的是，需要采用线程锁的方式，避免多个线程处理同一张图片。

​		第一：获取图片所在文件夹中所有图片的名称以及总的图片数量值；

​		第二：对已经在处理中的图片编号进行计数，使用线程锁避免该计数值被多个线程同时改变。

​		第三，由于线程锁机制，每个线程都会顺序分配一张其可以处理的图片编号，只需对该图片进行处理。

​		第四，加入判断语句，当正在处理的图片编号达到或超过总的图片数量值时，结束当前线程。

```C++
void WorkThread::dataprocessing(void){
  while(1){
    //线程一进入先判断是否图像已经处理完毕，如果完毕记录从开始至该线程结束所用的时间
    if(currentimagenum>=allImageNameList.count()){
      overtime=codetime.elapsed()/1000.0;
      emit time_over();//发送批量处理图片花费的时间到主UI所在的线程进行显示
      //结束当前线程
      this->terminate();
      this->wait();
      break;
    }

    //给需要保护的变量currentimagenum加锁，防止多线程时对同一图片进行处理
    QMutex mutex;
    mutex.lock();

    int num=currentimagenum;
    currentimagenum++;
    mutex.unlock();

    //载入需要处理的图片
    QString filename=imageprocessing_filename+"/"+(allImageNameList[num]);
    Mat img;
    std::string srcpath = filename.toLocal8Bit().toStdString(); //解决中文路径问题
    img = cv::imread(srcpath);//加载并图片，延时2后自动关闭窗口

    Mat biImage;
    //  转换为灰度图
    if (img.channels() == 4) {
      cv::cvtColor(img, biImage, CV_BGRA2GRAY);
      
    } else if (img.channels() == 3) {
      cv::cvtColor(img, biImage, CV_BGR2GRAY);
      
    } else if(img.channels() == 2) {
      cv::cvtColor(img,biImage,CV_BGR5652GRAY);
      
    } else if(img.channels() == 1) {// 单通道的图片直接就不需要处理

    } else { // 负数,说明图有问题 直接返回

    }

    //保存处理后的图片
    QImage Qtemp2 = QImage((const unsigned char*)(biImage.data), biImage.cols, biImage.rows,biImage.cols*biImage.channels(),QImage::Format_Grayscale8);
    QString savefile=imageprocessing_filename;
    savefile.append("/output/");
    QDir *photo = new QDir;
    if(!photo->exists(savefile)){
      //创建output文件夹
      photo->mkdir(savefile);
    }
    savefile.append(QString::number(num)+".jpg");
    QString path = QDir::toNativeSeparators(savefile);//将路径转换为当前系统所定义的路径
    Qtemp2.save(path,"JPG", 100);

    savefile.clear();
    filename.clear();
  }
}
```



# 图像分割

## 基于K-Means聚类算法的证件照背景替换

​		kmeans是非常经典的聚类算法，至今也还保留着较强的生命力，图像处理中经常用到kmeans算法或者其改进算法进行图像分割操作，在数据挖掘中kmeans经常用来做数据预处理。opencv中提供了完整的kmeans算法，其函数原型为：

double kmeans( InputArray data, int K, InputOutputArray bestLabels, TermCriteria criteria, int attempts, int flags, OutputArray centers = noArray() );

​		其中data表示用于聚类的数据，是N维的数组类型（Mat型），必须浮点型；

​		K表示需要聚类的类别数；

​		bestLabels聚类后的标签数组，Mat型；

​		criteria迭代收敛准则（MAX_ITER最大迭代次数，EPS最高精度）；

​		attemps表示尝试的次数，防止陷入局部最优；

​		flags 表示聚类中心的选取方式（KMEANS_RANDOM_CENTERS 随机选取，KMEANS_PP_CENTERS使用Arthur提供的算法，KMEANS_USE_INITIAL_LABELS使用初始标签）；

​		centers 表示聚类后的类别中心。


​		K-Means聚类算法正是一种可以将图像分割成K个聚类的迭代算法，对像素点进行聚类的基本流程如下：
​		（1）首先从一幅图像中任意选取 k 个像素点作为初始K个聚类的中心像素点，K值可以手动选取、随机选取、或其它方式得到；
​		（2）对于所剩下其它像素点，则根据它们与当前聚类中心像素点的相似度（距离）来进行度量，分别将它们分配给与其最相近的聚类中心像素点所代表的聚类。这里的相似度（距离）指某一像素点与聚类中心像素点之间的绝对偏差或偏差的平方，偏差通常用像素颜色、亮度、纹理、位置等属性，或这些属性的加权组合进行计算。
​		（3)然后再计算每个新聚类的聚类中心像素点（该聚类中所有像素点的均值）；
​		（4）重复第2和3步骤，直至收敛（聚类不再发生变化）。

​		在OpenCV中，基于K-Means聚类算法实现图像分割主要包含以下几个步骤：
​		（1）将图像转化为数据集，也就是把图像中每个像素点作为一个数据样本形成一个1列、N行、3通道的Mat对象（N为像素点总数量），其每一行为一个样本数据；
注意：需要将数据集转换为浮点型，因为K-Means聚类算法只适用于连续性数据，否则会出现中心像素点漂移出样本集范围的错误
​		（2）使用K-Means聚类算法对数据集进行像素点聚类。得到的聚类中心结果centers中，每一行是一个类别的中心像素点，总共有K行，也就是K类，每一列是中心像素点的一个通道值，{0，1，2}三列分别对应{B，G，R}三通道，而且centers中所有元素都是float类型，如果使用int或uchar类型返回，会导致数值错误，也即是获得的中心像素值出错；
​		（3）将聚类结果中不同类别的像素点都赋予不同的RGB值，而同一类别的像素点赋予同一RGB值，每一类别的代表RGB值可以是该类别中心像素点的RGB值。
示例1：

```C++
Mat transfImg = srcImage.clone();
Mat transfData = transfImg.reshape(3, transfImg.cols*transfImg.rows);

transfData.convertTo(transfData, CV_32F);
Mat bestLabels, centers;

//聚类的类别数，类别数越大则结果越接近原图像
int K = 3;
TermCriteria criteria = TermCriteria(TermCriteria::Type::COUNT + TermCriteria::Type::EPS, 10, 0.01);
kmeans(transfData, K, bestLabels, criteria, 3, KMEANS_PP_CENTERS, centers);
//将输出结果图中的每一个像素点，按照其所属的类别来改变其像素值
Mat kmeans_image = Mat::zeros(srcImage.size(), srcImage.type());
Mat mask = Mat::zeros(srcImage.size(), CV_8UC1);
int flag_label = bestLabels.at<int>(0, 0);			//获取左上角像素点的标签label，作为背景像素点的标签
for (int row = 0; row < transfImg.rows; row++) {
    for (int col = 0; col < transfImg.cols; col++) {
        int data_index = row * transfImg.cols + col;		//表示该像素点在数据集中所在的行索引，也就是第几个样本像素点
        int label = bestLabels.at<int>(data_index, 0);		//获取该像素点所属的类别标签
        //如果某像素点的label和背景像素点的label相同，则mask中对应位置的像素点置为255
        if (label == flag_label) {
            mask.at<uchar>(row, col) = 255;
        }

        if (label == 0) {
            Vec3b bgr = { 255,0,0 };
            kmeans_image.at<Vec3b>(row, col) = bgr;

        } else if (label == 1) {
            Vec3b bgr = { 0,255,0 };
            kmeans_image.at<Vec3b>(row, col) = bgr;

        } else if (label == 2) {
            Vec3b bgr = { 0,0,255 };
            kmeans_image.at<Vec3b>(row, col) = bgr;

        }
    }
}
//    imshow("Final result", kmeans_image);

//对mask图像进行闭操作，降低背景替换后图像的撕裂感
Mat kernel = getStructuringElement(MORPH_RECT, Size(3, 3));
morphologyEx(mask, mask, MORPH_CLOSE, kernel, Point(-1, -1), 1, 0);
//    imshow("mask", mask);

Mat dst;
dst = srcImage.clone();
for (int row = 0; row < transfImg.rows; row++){
    for (int col = 0; col < transfImg.cols; col++) {
        //以mask中同一位置坐标像素点的值为标志flag，如果flag==255证明该位置为背景，则将该像素值设置为我们需要的颜色
        int flag = mask.at<uchar>(row, col);
        if (flag == 255) {
            Vec3b bgr = { 255,255,255 };
            dst.at<Vec3b>(row, col) = bgr;
        }
    }
}
cvtColor(dst, dst, COLOR_BGR2RGB);
imageShow(dst);
```

示例2

```C++
//读取图像，并获取该图像的宽、高、总像素数
Mat test_image = imread("D:\\opencv_c++\\opencv_tutorial\\data\\images\\me.jpg");
resize(test_image, test_image, Size(500, 600));
imshow("test_image", test_image);
int width = test_image.cols;
int height = test_image.rows;
int data_counts = width * height;
//将图像转化成数据集，是一个列数为1、行数为总像素数的Mat对象，每一行的元素是一个像素点样本，具有三通道
Mat data = test_image.reshape(3, data_counts);
data.convertTo(data, CV_32F);

Mat bestLabels, centers;
int K = 10;			//聚类的类别数，类别数越大则结果越接近原图像
TermCriteria criteria = TermCriteria(TermCriteria::Type::COUNT + TermCriteria::Type::EPS, 10, 0.01);
kmeans(data, K, bestLabels, criteria, K, KMEANS_PP_CENTERS, centers);

//获取每一个类别的中心点的BGR值，形成一个分类别像素值索引向量
//聚类中心结果centers中，每一行是一个类别的中心像素点，总共有K行，也就是K类；
//每一列是中心像素点的一个通道值，{0，1，2}三列分别对应{B，G，R}三通道值；
//centers中所有元素都是float类型，如果使用int类型返回，会导致数值错误，则获得的中心像素值出错；
vector<Vec3b> centers_bgr(K);
for (int i = 0; i < K; i++){
    uchar b = uchar(centers.at<float>(i, 0));
    uchar g = uchar(centers.at<float>(i, 1));
    uchar r = uchar(centers.at<float>(i, 2));
    Vec3b bgr = { b,g,r };
    centers_bgr[i] = bgr;
}

//将输出结果图中的每一个像素点，按照其所属的类别来改变其像素值
Mat dst = Mat::zeros(test_image.size(), test_image.type());
for (int row = 0; row < height; row++){
    for (int col = 0; col < width; col++){
        //表示该像素点在数据集中所在的行索引，也就是第几个样本像素点
        int data_index = row * width + col;
        int label = bestLabels.at<int>(data_index, 0);	//获取该像素点所属的类别标签
        dst.at<Vec3b>(row, col) = centers_bgr[label];	//将该像素点所属类别的代表像素值（类别中心像素值）赋给该像素点
    }
}
imshow("dst", dst);
```



## GMM（高斯混合模型）方法

```C++
bool trainEM(InputArray samples,           // 输入的样本，一个单通道的矩阵。从这个样本中，进行高斯混和模型估计。
			OutputArray logLikelihoods=noArray(), //可选项，输出一个矩阵，里面包含每个样本的似然对数值。
			OutputArray labels=noArray(),  //可选项，输出每个样本对应的标注。
			OutputArray probs=noArray())   //可选项，输出一个矩阵，里面包含每个隐性变量的后验概率
```

补充一下EM::Types

​		COV_MAT_SPHERICAL	表示协方差矩阵是一个标量乘以单位矩阵，即sk×I，所以对协方差矩阵的估计只需要确定sk即可
​		COV_MAT_DIAGONAL	表示协方差矩阵是一个对角线元素为正数的对角矩阵，这时只需要对d个参数进行估计即可，d为样本特征属性的数量
​		COV_MAT_GENERIC	表示协方差矩阵是一个对称的正数矩阵，很显然，此时需要估计d2/2个参数，除非样本数据庞大，否则不建议设置该参数。

示例1

```C++
#include<iostream>
#include<opencv2/opencv.hpp>
 
using namespace std;
using namespace cv;
using namespace cv::ml;
 
int main(int argc, char** argv)
{
	Mat src = imread("E:/技能学习/opencv图像分割/test.jpg");
	if (src.empty()){
		cout << "could not load image!" << endl;
		return -1;
	}
 
	namedWindow("input image", WINDOW_AUTOSIZE);
	imshow("input image", src);
 
	//初始化
	int numCluster = 3;
	Scalar colorTab[] = {
		Scalar(0,0,255),
		Scalar(0,255,0),
		Scalar(255,0,0),
		Scalar(0,255,255), //红+绿 == 黄
	};
 
	int width = src.cols;
	int height = src.rows;
	int dims = src.channels();
 
	int numsamples = width * height; //样本总数
 
	Mat points(numsamples, dims, CV_64FC1); //样本矩阵  选择64位是为了精度更高
	Mat labels; //存放表示每个簇的标签
	Mat result = Mat::zeros(src.size(), CV_8UC3);
 
	//图像RGB像素数据转换为样本数据
	int index = 0;
	for (int row = 0; row < height; row++){
		for (int col = 0; col < width; col++){
			index = row * width + col;
			Vec3b rgb = src.at<Vec3b>(row, col);
			points.at<double>(index, 0) = static_cast<int>(rgb[0]);
			points.at<double>(index, 1) = static_cast<int>(rgb[1]);
			points.at<double>(index, 2) = static_cast<int>(rgb[2]);
		}
	}
 
	int begin_time = getTickCount();
	// EM Cluster Train
	Ptr<EM> em_model = EM::create();
	em_model->setClustersNumber(numCluster); //类别数
	em_model->setCovarianceMatrixType(EM::COV_MAT_SPHERICAL); //协方差矩阵的类型
	em_model->setTermCriteria(TermCriteria(TermCriteria::EPS + TermCriteria::COUNT, 100, 0.1)); //迭代停止的标准
	em_model->trainEM(points, noArray(), labels, noArray()); //训练
 
	//对每个像素标记颜色与显示  
	Mat sample(1, dims, CV_64FC1);
	int end_time = getTickCount();
 
	int r = 0, g = 0, b = 0;
 
	for (int row = 0; row < height; row++){
		for (int col = 0; col < width; col++){
			index = row * width + col;
 
			/*
			//直接显示
			int label = labels.at<int>(index, 0);
			Scalar c = colorTab[label];
			result.at<Vec3b>(row, col)[0] = c[0];
			result.at<Vec3b>(row, col)[1] = c[1];
			result.at<Vec3b>(row, col)[2] = c[2];
			*/
 
			//预言的方式显示
			b = src.at<Vec3b>(row, col)[0];
			g = src.at<Vec3b>(row, col)[1];
			r = src.at<Vec3b>(row, col)[2];
			sample.at<double>(0) = b;
			sample.at<double>(1) = g;
			sample.at<double>(2) = r;
 
			int response = cvRound(em_model->predict2(sample, noArray())[1]);
			Scalar c = colorTab[response];
 
			result.at<Vec3b>(row, col)[0] = c[0];
			result.at<Vec3b>(row, col)[1] = c[1];
			result.at<Vec3b>(row, col)[2] = c[2];
		}
	}
 
	//cout << "execution time:" << (time - getTickCount()) / (getTickFrequency() * 1000) << endl;
	cout << "execution time:" << ((end_time - begin_time)/ getTickFrequency()) << endl;
	imshow("EM-Segmentation:",result);
 
	waitKey(0);
	destroyAllWindows();
	return 0;
 
}
```



## 分水岭分割方法

​		[分水岭算法](https://so.csdn.net/so/search?q=分水岭算法&spm=1001.2101.3001.7020)主要是基于距离变换（distance transform），找到mark一些种子点，从这些种子点出发根据像素梯度变化进行寻找边缘并标记

​		markers中存储了图像的大致轮廓，并且以值1，2，3...分别表示各个components.markers通常由函数findContours() 和 drawContours()结合使用来获得。markers相当于watershed()运行时的种子参数。markers中，不属于轮廓(outlined regions)的点的值应置为0.函数运行后，图像中的像素如果是在由某个轮廓种子生成的区域中，那么其值就置为这个种子的编号，如果像素不在轮廓种子生成的区域中，则置为-1。


```C++
#include<opencv2/opencv.hpp>
 
#include <iostream>
 
using namespace cv;
using namespace std;
 
Vec3b RandomColor(int value);  //生成随机颜色函数
 
int main( int argc, char* argv[] )
{
	Mat image=imread(argv[1]);    //载入RGB彩色图像
	imshow("Source Image",image);
 
	//灰度化，滤波，Canny边缘检测
	Mat imageGray;
	cvtColor(image,imageGray,CV_RGB2GRAY);//灰度转换
	GaussianBlur(imageGray,imageGray,Size(5,5),2);   //高斯滤波
	imshow("Gray Image",imageGray); 
	Canny(imageGray,imageGray,80,150);  
	imshow("Canny Image",imageGray);
 
	//查找轮廓
	vector<vector<Point>> contours;  
	vector<Vec4i> hierarchy;  
	findContours(imageGray,contours,hierarchy,RETR_TREE,CHAIN_APPROX_SIMPLE,Point());  
	Mat imageContours=Mat::zeros(image.size(),CV_8UC1);  //轮廓	
	Mat marks(image.size(),CV_32S);   //Opencv分水岭第二个矩阵参数
	marks=Scalar::all(0);
	int index = 0;
	int compCount = 0;
	for( ; index >= 0; index = hierarchy[index][0], compCount++ ) {
		//对marks进行标记，对不同区域的轮廓进行编号，相当于设置注水点，有多少轮廓，就有多少注水点
		drawContours(marks, contours, index, Scalar::all(compCount+1), 1, 8, hierarchy);
		drawContours(imageContours,contours,index,Scalar(255),1,8,hierarchy);  
	}
 
	//我们来看一下传入的矩阵marks里是什么东西
	Mat marksShows;
	convertScaleAbs(marks,marksShows);
	imshow("marksShow",marksShows);
	imshow("轮廓",imageContours);
	watershed(image,marks);
 
	//我们再来看一下分水岭算法之后的矩阵marks里是什么东西
	Mat afterWatershed;
	convertScaleAbs(marks,afterWatershed);
	imshow("After Watershed",afterWatershed);
 
	//对每一个区域进行颜色填充
	Mat PerspectiveImage=Mat::zeros(image.size(),CV_8UC3);
	for(int i=0;i<marks.rows;i++){
		for(int j=0;j<marks.cols;j++){
			int index=marks.at<int>(i,j);
			if(marks.at<int>(i,j)==-1){
				PerspectiveImage.at<Vec3b>(i,j)=Vec3b(255,255,255);
                
			}else{
				PerspectiveImage.at<Vec3b>(i,j) =RandomColor(index);
                
			}
		}
	}
	imshow("After ColorFill",PerspectiveImage);
 
	//分割并填充颜色的结果跟原始图像融合
	Mat wshed;
	addWeighted(image,0.4,PerspectiveImage,0.6,0,wshed);
	imshow("AddWeighted Image",wshed);
 
	waitKey();
}
 
//生成随机颜色函数
Vec3b RandomColor(int value){
	value=value%255;  //生成0~255的随机数
	RNG rng;
	int aa=rng.uniform(0,value);
	int bb=rng.uniform(0,value);
	int cc=rng.uniform(0,value);
	return Vec3b(aa,bb,cc);
}
```



```C++
#include <opencv2/highgui/highgui.hpp>
#include <opencv2/imgproc/imgproc.hpp>
#include <iostream>
#include <vector>

using namespace cv;
using namespace std;

int main(){
	Mat src = imread("coins.jpg");
	if (!src.data){
		printf("could not load image...\n");
		return -1;
	}
	imshow("src", src);
	Mat Gray, shifted, binary;
	//均值漂移Meanshift
	pyrMeanShiftFiltering(src, shifted, 21, 51);
	imshow("shifted", shifted);
	//转换成灰度图像
	cvtColor(shifted, Gray, CV_BGR2GRAY);
	imshow("Gray", Gray);
	//使用OTSU阈值算法进行二值化
	threshold(Gray, binary, 0, 255, THRESH_BINARY | THRESH_OTSU);
	imshow("binary", binary);
	
	//距离变换
	Mat DisTranMat(binary.rows, binary.cols, CV_32FC1);
	distanceTransform(binary, DisTranMat, DistanceTypes::DIST_L2, 3);
	//归一化
	normalize(DisTranMat, DisTranMat, 0.0, 1.0, NORM_MINMAX);
	imshow("DisTranMat", DisTranMat);
 
	//在进行阈值化分割
	threshold(DisTranMat, DisTranMat, 0.4, 1, THRESH_BINARY);
	imshow("TDisTranMat", DisTranMat);
	//归一化统计图像到0-255
	normalize(DisTranMat, DisTranMat, 0.0, 255.0, NORM_MINMAX);
	DisTranMat.convertTo(DisTranMat, CV_8UC1);
	imshow("NorDisTranMat", DisTranMat);
 
	//进行计算标记的分割块
	vector<vector<Point>> contours;//vector<Point>点的集合，一系列的点组成轮廓的集合
	vector<Vec4i> hierarchy;
	findContours(DisTranMat, contours, hierarchy, CV_RETR_EXTERNAL, CV_CHAIN_APPROX_SIMPLE, Point(0, 0));
	Mat markers = Mat::zeros(src.size(), CV_32SC1);
	for (size_t t = 0; t < contours.size(); t++){
		drawContours(markers, contours, static_cast<int>(t), Scalar::all(static_cast<int>(t) + 1), -1);//这里填充的颜色是从1开始
	}
	circle(markers, Point(5, 5), 3, Scalar(255), -1);
	//imshow("markers", markers*10000);
 
	//形态学腐蚀操作
	Mat k = getStructuringElement(MORPH_RECT, Size(3, 3), Point(-1, -1));
	morphologyEx(src, src, MORPH_ERODE, k);
	//完成分水岭算法
	watershed(src, markers);
	Mat mark = Mat::zeros(markers.size(), CV_8UC1);
	markers.convertTo(mark, CV_8UC1);
	//取反
	bitwise_not(mark, mark);
	imshow("mark", mark);
 
	vector<Vec3b> colors;
	for (size_t i = 0; i < contours.size(); i++) {
		int r = theRNG().uniform(0, 255);
		int g = theRNG().uniform(0, 255);
		int b = theRNG().uniform(0, 255);
		colors.push_back(Vec3b((uchar)b, (uchar)g, (uchar)r));
	}
 
	// 颜色填充与最终显示
	Mat dst = Mat::zeros(markers.size(), CV_8UC3);
	int index = 0;
	for (int row = 0; row < markers.rows; row++){
		for (int col = 0; col < markers.cols; col++){
			index = markers.at<int>(row, col);//读取每一个部分的颜色的值
			if (index > 0 && index <= contours.size()){
				dst.at<Vec3b>(row, col) = colors[index - 1];
                
			}else{
                
				dst.at<Vec3b>(row, col) = Vec3b(0, 0, 0);
			}
		}
	}
 
	imshow("Final Result", dst);
	printf("number of objects : %d\n", contours.size());
	waitKey(0);
	return 0;
}
```

# 肤色检测

## 基于YCrCb颜色空间Cr,Cb范围筛选法

1）将RGB模型转换为YCbCr模型

2）阈值分割：据资料显示，正常黄种人的Cr分量大约在133至173之间，Cb分量大约在77至127之间。

```c++
Mat srcImg = imread("E:\\QtProjectFiles\\workDemo\\hand1.jpg");

Mat resImg, tmpImg;
Mat Y, Cr, Cb;
vector<Mat> channels;

srcImg.copyTo(tmpImg);   //将原图拷贝一份
cvtColor(tmpImg, tmpImg, COLOR_BGR2YCrCb);        //转换到YCrCb空间
split(tmpImg, channels);    //通道分离的图存在channels中
Y = channels.at(0);
Cr = channels.at(1);
Cb = channels.at(2);

resImg = Mat::zeros(srcImg.size(), CV_8UC1);
for (int i = 0; i < resImg.rows; i++) {
    uchar* currentCr = Cr.ptr<uchar>(i);
    uchar* currentCb = Cb.ptr<uchar>(i);
    uchar* current = resImg.ptr<uchar>(i);
    for (int j = 0; j < resImg.cols; j++) {
        /*据资料显示，正常黄种人的Cr分量大约在133至173之间，
               Cb分量大约在77至127之间。大家可以根据自己项目需求放大或缩小这两个分量的范围，会有不同的效果。*/
        if ((currentCr[j] > 133) && (currentCr[j] < 173) && (currentCb[j] > 77) && (currentCb[j] < 127))
            current[j] = 255;
        else
            current[i] = 0;
    }
}
imshow("result", resImg);
```

## 基于椭圆皮肤模型的皮肤检测

皮肤模型中有单高斯，混合高斯，贝叶斯模型和椭圆模型等。经过前人学者大量的皮肤统计信息可以知道，如果将皮肤信息映射到YCrCb空间，则在CrCb二维空间中这些皮肤像素点近似成一个椭圆分布。因此如果我们得到了一个CrCb的椭圆，下次来一个坐标(Cr, Cb)我们只需判断它是否在椭圆内（包括边界），如果是，则可以判断其为皮肤，否则就是非皮肤像素点。
```C++
Mat srcImg = imread("E:\\QtProjectFiles\\workDemo\\hand1.jpg");
//构建椭圆模型
Mat tmp=srcImg.clone();//克隆一张原图

Mat SkinMat = Mat::zeros(Size(256, 256), CV_8UC1);
//利用opencv自带的椭圆生成函数先生成一个肤色椭圆模型
ellipse(SkinMat, Point(113, 115.6), Size(23.4, 15.2), 43.0, 0.0, 360.0, Scalar(255, 255, 255), -1);
Mat YcrcbMat, resultMat;//创建一张ycrcb颜色空间图，与结果图
resultMat = Mat::zeros(tmp.size(), CV_8UC1);
//将颜色空间转换到ycrcb
cvtColor(tmp, YcrcbMat, COLOR_BGR2YCrCb);
for (int i = 0; i < resultMat.rows; i++){
    uchar *presult = resultMat.ptr<uchar>(i);
    Vec3b *ycrcb = YcrcbMat.ptr<Vec3b>(i);

    for (int j = 0; j < resultMat.cols; j++) {
        //颜色判断
        if (SkinMat.at<uchar>(ycrcb[j][1], ycrcb[j][2]) > 0)
            presult[j] = 255;
    }
}
imshow("result", resultMat);
```



# Qt鼠标绘图画板

​		在Qt中实现鼠标绘图画板，可以使用Qwidget类的继承来创建自定义的画板控件。以下是一个简单的实现方法： 创建一个自定义的Qwidget类，例如Mywidget， 并在其中定义一些需要的变量和函数。

```c++
class DrawBoardWidget : public QWidget
{
    Q_OBJECT

public:
    explicit DrawBoardWidget(QWidget *parent = nullptr);

protected:
    void mousePressEvent(QMouseEvent *event) override;
    void mouseMoveEvent(QMouseEvent *event) override;
    void paintEvent(QPaintEvent *event) override;

private:
    bool m_drawing;
    QPoint m_lastPoint;
    QPixmap m_pixmap;
};
```

在构造函数中初始化变量

```c++
DrawBoardWidget::DrawBoardWidget(QWidget *parent) : QWidget(parent)
{
    m_drawing = false;
    setFixedSize(400, 400);//设置画板大小
    m_pixmap = QPixmap(size());
    m_pixmap.fill(Qt::white);
}
```

在鼠标按下事件和鼠标移动事件中实现绘图逻辑

```c++
void DrawBoardWidget::mousePressEvent(QMouseEvent *event)
{
    if (event->button() == Qt::LeftButton) {
        m_drawing = true;
        m_lastPoint = event->pos();
    }
}

void DrawBoardWidget::mouseMoveEvent(QMouseEvent *event)
{
    if ((event->buttons() & Qt::LeftButton) && m_drawing) {
        QPainter painter(&m_pixmap);
        painter.setPen(QPen(Qt::black, 2, Qt::SolidLine, Qt::RoundCap, Qt::RoundJoin));
        painter.drawLine(m_lastPoint, event->pos());
        m_lastPoint = event->pos();
        update();
    }
}
//清空画板
void DrawBoardWidget::clearPaint()
{
    m_pixmap.fill(Qt::black);
    update();
}
```

在绘图事件中将绘图内容绘制到画板中

```c++
void DrawBoardWidget::paintEvent(QPaintEvent *event)
{
    QPainter painter(this);
    painter.drawPixmap(rect(), m_pixmap);
}
```

将自定义的QWidget控件添加到主窗口中

```c++
MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
{
    DrawBoardWidget *drawingBoard = new DrawBoardWidget(ui->widget1);
    //setCentralWidget(widget);
    drawingBoard->setGeometry(ui->widget1->geometry());
        
    connect(ui->btnClear, &QPushButton::clicked, drawingBoard, &DrawBoardWidget::clearPaint);
}
//save picture
void MainWindow::on_btnSave_clicked()
{
    QString fileName = QFileDialog::getSaveFileName(this, tr("Save Image"), ".", tr("Images (*.png *.xpm *.jpg)"));
    if (!fileName.isEmpty()) {
        QPixmap pixmap(ui->widget1->size());
        ui->widget1->render(&pixmap);
        pixmap.save(fileName);
    }
    //获取widget的内容
//    QPixmap pixmap = QPixmap::grabWidget(ui->widget1);
//    QPixmap savePic = pixmap.scaled(ui->widget1->size());
//    QString configFilePath = qApp->applicationDirPath();
//    E:\\QtProjectFiles
//    QString saveName = QString("%1.jpg").arg(1);
//    qDebug()<<saveName;
//    savePic.save(configFilePath+saveName);
}
```



代码示例二：

训练数据

```c++
QString trainImgPath = "E:\\QtProjectFiles\\knndigitDemo\\data\\train-images-idx3-ubyte";
QString trainLabelPath = "E:\\QtProjectFiles\\knndigitDemo\\data\\train-labels-idx1-ubyte";
QString testImgPath = "E:\\QtProjectFiles\\knndigitDemo\\data\\t10k-images-idx3-ubyte";
QString testLabelPath = "E:\\QtProjectFiles\\knndigitDemo\\data\\t10k-labels-idx1-ubyte";
// 1 训练数据准备
qDebug()<<"ready data";
//读取训练标签数据 （60000， 1） int32
Mat trainLabels = readMnistLabels(trainLabelPath);
//读取训练图像数据（60000， 784）float32
Mat trainImages = readMnistImages(trainImgPath);

trainImages = trainImages / 255.0;

//读取训练标签数据 （10000， 1） int32
Mat testLabels = readMnistLabels(testLabelPath);
//读取训练图像数据（10000， 784）float32
Mat testImages = readMnistImages(testImgPath);

testImages = testImages / 255.0;

// 2 构建KNN训练
qDebug()<<"step 2";
cv::Ptr<cv::ml::KNearest>knn=cv::ml::KNearest::create();
knn->setDefaultK(5);
knn->setIsClassifier(true);

cv::Ptr<cv::ml::TrainData>traindata = cv::ml::TrainData::create(trainImages, cv::ml::ROW_SAMPLE, trainLabels);
qDebug()<<"start training...";
knn->train(traindata);
qDebug()<<"train finished";

Mat preOut;
//返回值为第一个图像的预测值 pre_out为整个batch的预测值集合
float ret = knn->predict(testImages, preOut);
qDebug()<<"predict result"<<ret;

//计算准确率必须将两种标签化为同一数据类型
preOut.convertTo(preOut, CV_8UC1);
testLabels.convertTo(testLabels, CV_8UC1);

int equalNums = 0;
for(int i = 0; i < preOut.rows; i++){
    if(preOut.at<uchar>(i, 0) == testLabels.at<uchar>(i, 0)){
        equalNums++;
    }
}
float acc = float(equalNums) / float(preOut.rows);
qDebug()<<"accuary"<<acc;
knn->save("E:\\QtProjectFiles\\knndigitDemo\\mnist_knn.xml");

```



```C++
//存储转换 将一个32位整数（即4个字节）进行字节序的反转
int MainWindow::reverseInt(int i)
{
    unsigned char c1, c2, c3, c4;

    c1 = i & 255;
    c2 = (i >> 8) & 255;
    c3 = (i >> 16) & 255;
    c4 = (i >> 24) & 255;

    return ((int)c1<<24) + ((int)c2 << 16) + ((int)c3 << 8) + c4;
}
```



```C++
Mat MainWindow::readMnistImages(const QString fileName)
{
    int magicNums = 0;
    int imageNums = 0;
    int nRows = 0;
    int nCols = 0;

    Mat dataMat;

    string filePath = fileName.toStdString();
    ifstream file(filePath, ios::binary);
    if(file.is_open()){
        qDebug()<<"open image data success...";
        file.read((char*)&magicNums, sizeof (magicNums));  //文件格式
        file.read((char*)&imageNums, sizeof (imageNums));  //图像总数

        file.read((char*)&nRows, sizeof (nRows));        //每个图象的行数
        file.read((char*)&nCols, sizeof (nCols));            //每个图像的列数

        magicNums = reverseInt(magicNums);
        imageNums = reverseInt(imageNums);
        nRows = reverseInt(nRows);
        nCols = reverseInt(nCols);

        qDebug()<<"starting read image data...";

        dataMat = Mat::zeros(imageNums, nRows * nCols, CV_32FC1);
        for (int i = 0; i < imageNums; i++) {
            for (int j = 0; j < nRows * nCols; j++) {
                unsigned char temp = 0;
                file.read((char*)&temp, sizeof (temp));
                float pixelValue = float(temp);
                dataMat.at<float>(i, j) = pixelValue;

            }
        }

    }
    file.close();
    return dataMat;
}
```



```C++
Mat MainWindow::readMnistLabels(const QString fileName)
{
    int magicNumber;
    int itemNumbers;

    string filePath = fileName.toStdString();
    Mat labelMat;
    ifstream file(filePath, ios::binary);
    if (file.is_open()){
        qDebug()<<"-open label data success...";
        file.read((char*)&magicNumber, sizeof (magicNumber));
        file.read((char*)&itemNumbers, sizeof (itemNumbers));
        magicNumber = reverseInt(magicNumber);
        itemNumbers = reverseInt(itemNumbers);
        qDebug()<<"-magic numbers"<<magicNumber<<"label nums"<<itemNumbers;

        labelMat = Mat::zeros(itemNumbers, 1, CV_32SC1);
        for (int i = 0; i < itemNumbers; i++) {
            unsigned char temp = 0;
            file.read((char*)&temp, sizeof (temp));
            labelMat.at<unsigned int>(i, 0) = (unsigned int)temp;
        }

    }
    file.close();
    return labelMat;

}
```

​		首先获取`widget1`的大小，然后使用`grab()`函数获取绘制的内容，并将其转换为`QImage`类型。然后，创建一个`cv::Mat`类型的图像，并使用`QImage`中的像素值初始化该图像。最后，将图像从彩色图像转换为灰度图像。

​		通过这种方式，可以将绘制的内容保存到`cv::Mat`类型的图像中，并用于模型预测。

```C++
void MainWindow::on_btnPredict_clicked()
{
    QSize size = ui->widget1->size();
    QPixmap pixmap = ui->widget1->grab();
    QImage image = pixmap.toImage();
    Mat mat(size.height(), size.width(), CV_8UC4, image.bits(), image.bytesPerLine());
    Mat preImage;
    cvtColor(mat, preImage, COLOR_BGRA2GRAY);

    // load knn model 可将加载模型全局化
    cv::Ptr<cv::ml::KNearest>knn = cv::ml::StatModel::load<cv::ml::KNearest>("E:\\QtProjectFiles\\knndigitDemo\\mnist_knn.xml");
    //change data type               uchar to float32
    cv::resize(preImage, preImage, Size(28, 28));
    preImage.convertTo(preImage, CV_32F);

    // 将图像转换为一维向量
    Mat imageVector = preImage.reshape(1, 1);

    //predict image
    float result = knn->predict(imageVector);
    qDebug()<<"result "<<result;
}
```

