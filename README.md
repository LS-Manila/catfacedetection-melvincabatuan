# Cat face detection with OpenCV

catfacedetection-melvincabatuan created by Classroom for GitHub

This project implements detection of cat faces with OpenCV Haar cascade classifier.

## Prerequisite:

1. Native Development Kit (NDK): http://developer.android.com/ndk/index.html 
2. OpenCV: (Refer to https://github.com/DeLaSalleUniversity-Manila/opencvcamerapreviewsample-melvincabatuan for adding OpenCV module dependency to your project.)

## Keypoints (JNI):

### Android.mk
```make
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

OPENCV_INSTALL_MODULES:=on
OPENCV_LIB_TYPE:=SHARED

include /home/cobalt/Android/OpenCV-android-sdk/sdk/native/jni/OpenCV.mk

LOCAL_SRC_FILES  := DetectionBasedTracker.cpp
LOCAL_C_INCLUDES += $(LOCAL_PATH)
LOCAL_LDLIBS     += -llog -ldl

LOCAL_MODULE     := detection_based_tracker

include $(BUILD_SHARED_LIBRARY)
```

### Application.mk
```make
APP_STL := gnustl_static
APP_CPPFLAGS := -frtti -fexceptions
APP_ABI := armeabi-v7a
APP_PLATFORM := android-15
```

### DetectionBasedTracker.cpp (based on OpenCV FaceDetection Sample)
```cpp
#include "DetectionBasedTracker.h"

/* ph_edu_dlsu_catfacedetection_DetectionBasedTracker class */


#include <opencv2/core/core.hpp>
#include <opencv2/objdetect.hpp>

#include <string>
#include <vector>

#include <android/log.h>

#define LOG_TAG "CatFaceDetection/DetectionBasedTracker"
#define LOGD(...) ((void)__android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__))

using namespace std;
using namespace cv;

inline void vector_Rect_to_Mat(vector<Rect>& v_rect, Mat& mat)
{
    mat = Mat(v_rect, true);
}

class CascadeDetectorAdapter: public DetectionBasedTracker::IDetector
{
public:
    CascadeDetectorAdapter(cv::Ptr<cv::CascadeClassifier> detector):
            IDetector(),
            Detector(detector)
    {
        LOGD("CascadeDetectorAdapter::Detect::Detect");
        CV_Assert(detector);
    }

    void detect(const cv::Mat &Image, std::vector<cv::Rect> &objects)
    {
        LOGD("CascadeDetectorAdapter::Detect: begin");
        LOGD("CascadeDetectorAdapter::Detect: scaleFactor=%.2f, minNeighbours=%d, minObjSize=(%dx%d), maxObjSize=(%dx%d)", scaleFactor, minNeighbours, minObjSize.width, minObjSize.height, maxObjSize.width, maxObjSize.height);
        Detector->detectMultiScale(Image, objects, scaleFactor, minNeighbours, 0, minObjSize, maxObjSize);
        LOGD("CascadeDetectorAdapter::Detect: end");
    }

    virtual ~CascadeDetectorAdapter()
    {
        LOGD("CascadeDetectorAdapter::Detect::~Detect");
    }

private:
    CascadeDetectorAdapter();
    cv::Ptr<cv::CascadeClassifier> Detector;
};

struct DetectorAgregator
{
    cv::Ptr<CascadeDetectorAdapter> mainDetector;
    cv::Ptr<CascadeDetectorAdapter> trackingDetector;

    cv::Ptr<DetectionBasedTracker> tracker;
    DetectorAgregator(cv::Ptr<CascadeDetectorAdapter>& _mainDetector, cv::Ptr<CascadeDetectorAdapter>& _trackingDetector):
            mainDetector(_mainDetector),
            trackingDetector(_trackingDetector)
    {
        CV_Assert(_mainDetector);
        CV_Assert(_trackingDetector);

        DetectionBasedTracker::Parameters DetectorParams;
        tracker = makePtr<DetectionBasedTracker>(mainDetector, trackingDetector, DetectorParams);
    }
};



/*
 * Class:     ph_edu_dlsu_catfacedetection_DetectionBasedTracker
 * Method:    nativeCreateObject
 * Signature: (Ljava/lang/String;I)J
 */
JNIEXPORT jlong JNICALL Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeCreateObject
  (JNIEnv * jenv, jclass, jstring jFileName, jint faceSize)
{
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeCreateObject enter");
    const char* jnamestr = jenv->GetStringUTFChars(jFileName, NULL);
    string stdFileName(jnamestr);
    jlong result = 0;

    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeCreateObject");

    try
    {
        cv::Ptr<CascadeDetectorAdapter> mainDetector = makePtr<CascadeDetectorAdapter>(
            makePtr<CascadeClassifier>(stdFileName));
        cv::Ptr<CascadeDetectorAdapter> trackingDetector = makePtr<CascadeDetectorAdapter>(
            makePtr<CascadeClassifier>(stdFileName));
        result = (jlong)new DetectorAgregator(mainDetector, trackingDetector);
        if (faceSize > 0)
        {
            mainDetector->setMinObjectSize(Size(faceSize, faceSize));
            //trackingDetector->setMinObjectSize(Size(faceSize, faceSize));
        }
    }
    catch(cv::Exception& e)
    {
        LOGD("nativeCreateObject caught cv::Exception: %s", e.what());
        jclass je = jenv->FindClass("org/opencv/core/CvException");
        if(!je)
            je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, e.what());
    }
        catch (...)
        {
        LOGD("nativeCreateObject caught unknown exception");
        jclass je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, "Unknown exception in JNI code of DetectionBasedTracker.nativeCreateObject()");
        return 0;
    }

    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeCreateObject exit");
    return result;
}

/*
 * Class:     ph_edu_dlsu_catfacedetection_DetectionBasedTracker
 * Method:    nativeDestroyObject
 * Signature: (J)V
 */
JNIEXPORT void JNICALL Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeDestroyObject
  (JNIEnv * jenv, jclass, jlong thiz)
{
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeDestroyObject");

    try
    {
        if(thiz != 0)
        {
            ((DetectorAgregator*)thiz)->tracker->stop();
            delete (DetectorAgregator*)thiz;
        }
    }
    catch(cv::Exception& e)
    {
        LOGD("nativeestroyObject caught cv::Exception: %s", e.what());
        jclass je = jenv->FindClass("org/opencv/core/CvException");
        if(!je)
            je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, e.what());
    }
    catch (...)
    {
        LOGD("nativeDestroyObject caught unknown exception");
        jclass je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, "Unknown exception in JNI code of DetectionBasedTracker.nativeDestroyObject()");
    }
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeDestroyObject exit");
}



/*
 * Class:     ph_edu_dlsu_catfacedetection_DetectionBasedTracker
 * Method:    nativeStart
 * Signature: (J)V
 */
JNIEXPORT void JNICALL Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeStart
  (JNIEnv * jenv, jclass, jlong thiz)
{
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeStart");

    try
    {
        ((DetectorAgregator*)thiz)->tracker->run();
    }
    catch(cv::Exception& e)
    {
        LOGD("nativeStart caught cv::Exception: %s", e.what());
        jclass je = jenv->FindClass("org/opencv/core/CvException");
        if(!je)
            je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, e.what());
    }
    catch (...)
    {
        LOGD("nativeStart caught unknown exception");
        jclass je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, "Unknown exception in JNI code of DetectionBasedTracker.nativeStart()");
    }
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeStart exit");
}



/*
 * Class:     ph_edu_dlsu_catfacedetection_DetectionBasedTracker
 * Method:    nativeStop
 * Signature: (J)V
 */
JNIEXPORT void JNICALL Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeStop
 (JNIEnv * jenv, jclass, jlong thiz)
{
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeStop");

    try
    {
        ((DetectorAgregator*)thiz)->tracker->stop();
    }
    catch(cv::Exception& e)
    {
        LOGD("nativeStop caught cv::Exception: %s", e.what());
        jclass je = jenv->FindClass("org/opencv/core/CvException");
        if(!je)
            je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, e.what());
    }
    catch (...)
    {
        LOGD("nativeStop caught unknown exception");
        jclass je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, "Unknown exception in JNI code of DetectionBasedTracker.nativeStop()");
    }
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeStop exit");
}


/*
 * Class:     ph_edu_dlsu_catfacedetection_DetectionBasedTracker
 * Method:    nativeSetFaceSize
 * Signature: (JI)V
 */
JNIEXPORT void JNICALL Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeSetFaceSize
  (JNIEnv * jenv, jclass, jlong thiz, jint faceSize)
{
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeSetFaceSize -- BEGIN");

    try
    {
        if (faceSize > 0)
        {
            ((DetectorAgregator*)thiz)->mainDetector->setMinObjectSize(Size(faceSize, faceSize));
            //((DetectorAgregator*)thiz)->trackingDetector->setMinObjectSize(Size(faceSize, faceSize));
        }
    }
    catch(cv::Exception& e)
    {
        LOGD("nativeStop caught cv::Exception: %s", e.what());
        jclass je = jenv->FindClass("org/opencv/core/CvException");
        if(!je)
            je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, e.what());
    }
    catch (...)
    {
        LOGD("nativeSetFaceSize caught unknown exception");
        jclass je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, "Unknown exception in JNI code of DetectionBasedTracker.nativeSetFaceSize()");
    }
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeSetFaceSize -- END");
}


/*
 * Class:     ph_edu_dlsu_catfacedetection_DetectionBasedTracker
 * Method:    nativeDetect
 * Signature: (JJJ)V
 */
JNIEXPORT void JNICALL Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeDetect
 (JNIEnv * jenv, jclass, jlong thiz, jlong imageGray, jlong faces)
{
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeDetect");

    try
    {
        vector<Rect> RectFaces;
        ((DetectorAgregator*)thiz)->tracker->process(*((Mat*)imageGray));
        ((DetectorAgregator*)thiz)->tracker->getObjects(RectFaces);
        *((Mat*)faces) = Mat(RectFaces, true);
    }
    catch(cv::Exception& e)
    {
        LOGD("nativeCreateObject caught cv::Exception: %s", e.what());
        jclass je = jenv->FindClass("org/opencv/core/CvException");
        if(!je)
            je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, e.what());
    }
    catch (...)
    {
        LOGD("nativeDetect caught unknown exception");
        jclass je = jenv->FindClass("java/lang/Exception");
        jenv->ThrowNew(je, "Unknown exception in JNI code DetectionBasedTracker.nativeDetect()");
    }
    LOGD("Java_ph_edu_dlsu_catfacedetection_DetectionBasedTracker_nativeDetect END");
}
```

## Creating the JNI Header
```shell
$ cd ~/AndroidStudioProjects/CatFaceDetection/app/
$ ls
app.iml  build  build.gradle  jni  libs  proguard-rules.pro  src
$ javah -d jni -classpath /home/cobalt/Android/Sdk/platforms/android-22/android.jar:/home/cobalt/Android/Sdk/extras/android/support/v7/appcompat/libs/android-support-v7-appcompat.jar:/home/cobalt/Android/Sdk/extras/android/support/v4/android-support-v4.jar:build/intermediates/classes/debug:../libraries/opencv/build/intermediates/classes/debug ph.edu.dlsu.catfacedetection.DetectionBasedTracker
```


## Native Build
```shell
$ ndk-build
[armeabi-v7a] Compile++ thumb: detection_based_tracker <= DetectionBasedTracker.cpp
[armeabi-v7a] Prebuilt       : libopencv_java3.so <= /home/cobalt/Android/OpenCV-android-sdk/sdk/native/jni/../libs/armeabi-v7a/
[armeabi-v7a] SharedLibrary  : libdetection_based_tracker.so
/home/cobalt/Android/adt-bundle-linux-x86-20131030/android-ndk-r10d/toolchains/arm-linux-androideabi-4.8/prebuilt/linux-x86/bin/../lib/gcc/arm-linux-androideabi/4.8/../../../../arm-linux-androideabi/bin/ld: warning: hidden symbol '__aeabi_atexit' in /home/cobalt/Android/adt-bundle-linux-x86-20131030/android-ndk-r10d/sources/cxx-stl/gnu-libstdc++/4.8/libs/armeabi-v7a/thumb/libgnustl_static.a(atexit_arm.o) is referenced by DSO /home/cobalt/AndroidStudioProjects/CatFaceDetection/app/obj/local/armeabi-v7a/libopencv_java3.so
[armeabi-v7a] Install        : libdetection_based_tracker.so => libs/armeabi-v7a/libdetection_based_tracker.so
[armeabi-v7a] Install        : libopencv_java3.so => libs/armeabi-v7a/libopencv_java3.so
```


## Modify the 'Module:app build.gradle' sourceSets jniLibs.srcDirs into 'libs'
```gradle
apply plugin: 'com.android.application'

android {
    compileSdkVersion 22
    buildToolsVersion "22.0.1"

    defaultConfig {
        applicationId "ph.edu.dlsu.catfacedetection"
        minSdkVersion 15
        targetSdkVersion 22
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

    sourceSets {
        main {
            jni.srcDirs = []
            jniLibs.srcDirs=['libs']
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:22.2.1'
    compile 'com.android.support:design:22.2.1'
    compile project(':libraries:opencv')
}
```


## Accept

To accept the assignment, click the following URL:

https://classroom.github.com/assignment-invitations/70b4edeefa11789443ca80c7935e1b85

## Sample Solution:

https://github.com/DeLaSalleUniversity-Manila/catfacedetection-melvincabatuan

## Submission Procedure with Git: 

```shell
$ cd /path/to/your/android/app/
$ git init
$ git add –all
$ git commit -m "your message, e.x. Assignment 1 submission"
$ git remote add origin <Assignment link copied from assignment github, e.x. https://github.com/DeLaSalleUniversity-Manila/secondactivityassignment-melvincabatuan.git>
$ git push -u origin master
<then Enter Username and Password>
```


## Screenshots:


![alt tag](https://github.com/DeLaSalleUniversity-Manila/catfacedetection-melvincabatuan/blob/master/device-2015-11-06-210811.png)

![alt tag](https://github.com/DeLaSalleUniversity-Manila/catfacedetection-melvincabatuan/blob/master/device-2015-11-06-210918.png)

![alt tag](https://github.com/DeLaSalleUniversity-Manila/catfacedetection-melvincabatuan/blob/master/device-2015-11-06-211012.png)

![alt tag](https://github.com/DeLaSalleUniversity-Manila/catfacedetection-melvincabatuan/blob/master/device-2015-11-06-211822.png)

![alt tag](https://github.com/DeLaSalleUniversity-Manila/catfacedetection-melvincabatuan/blob/master/device-2015-11-06-212138.png)

"*C makes it easy to shoot yourself in the foot; C++ makes it harder, but when you do, it blows away your whole leg.*" - Bjarne Stroustrup (Danish computer scientist, developer of the C++ programming language)
