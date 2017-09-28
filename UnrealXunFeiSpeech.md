关键字: Unreal 蓝图 语音识别 C++

---
*由于项目需求，使用语音进行游戏交互，踩了两周多的坑，终于搞出来了，记录一下踩坑经历*

先贴上demo：https://github.com/303snowing/UnrealXunFeiSpeech
demo的引擎版本是4.17.1，使用Visual Studio 2017 15.3

---
### **准备工作**
1. 先到科大讯飞的语音平台注册、创建一个应用(这步必须，因为只有拥有appid才能下载对应的sdk)
2. 下载在线语音听写sdk：http://www.xfyun.cn/services/voicedictation，解压
3. 建立Unreal C++空白项目，不需要包含Start Content，示例的项目名称为`UnrealXunFeiSpeech`
4. 编译项目并启动项目实例
5. 建立一个继承自Actor的C++类，可见性为Public

> 讯飞库在 Windows_voice_1166_59940824/libs 和 Windows_voice_1166_59940824/bin 目录下,头文件在 Windows_voice_1166_59940824/include 目录下

### **导入讯飞库**
1. 在项目根目录下建立一个XunFei文件夹
2. 将Windows_voice_1166_59940824/libs和Windows_voice_1166_59940824/include目录拷贝到XunFei文件夹中
3. 将Windows_voice_1166_59940824/bin下的`msc_x64.dll`文件拷贝到项目工程的Binaries/Win64目录下(该.dll文件在项目迁移或打包过程中都不可缺少)

### **配置讯飞库搜索路径**
编辑Source/UnrealXunFeiSpeech/UnrealXunFeiSpeech.Build.cs文件，代码如下
```
// Fill out your copyright notice in the Description page of Project Settings.

using UnrealBuildTool;

public class UnrealXunFeiSpeech : ModuleRules
{
	public UnrealXunFeiSpeech(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
	
		PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "Json" });

		PrivateDependencyModuleNames.AddRange(new string[] {  });

        // Uncomment if you are using Slate UI
        // PrivateDependencyModuleNames.AddRange(new string[] { "Slate", "SlateCore" });

        // Uncomment if you are using online features
        // PrivateDependencyModuleNames.Add("OnlineSubsystem");

        // To include OnlineSubsystemSteam, add it to the plugins section in your uproject file with the Enabled attribute set to true

		// 此处指定文件搜索路径
        PrivateIncludePaths.Add("UnrealXunFeiSpeech/Private");
        PublicIncludePaths.Add("UnrealXunFeiSpeech/Public");
		//引入讯飞静态库
        PublicLibraryPaths.AddRange(new string[] { "..\\XunFei\\libs" });
        PublicAdditionalLibraries.AddRange(new string[] { "msc_x64.lib" });
		//添加文件搜索路径
        PublicIncludePaths.AddRange(new string[] { "..\\XunFei\\include" });
    }
}

```

### 编写代码
#### 1. 创建`FWinRec`类
FWinRec类对应讯飞官方例子的winrec.c文件,是封装的Windows录音功能，在Source/UnrealXunFeiSpeech/Public下创建`WinRec.h`，在Source/UnrealXunFeiSpeech/Private下创建`WinRec.cpp`，并在项目中添加到对应目录下
> **[WinRec.h][WinRec.h] | [WinRec.cpp][WinRec.cpp]**
> 由于在其他文件中，均直接或者间接的包含WinRec.h,所以在WinRec.h中添加了自定义的Log标签。
```C++
// Fill out your copyright notice in the Description page of Project Settings.
/*
* @file
* @brief a record interface in windows
*
* it encapsluate the windows API waveInxxx;
* Common steps:
*	create_recorder,
*	open_recorder,
*	start_record,
*	stop_record,
*	close_recorder,
*	destroy_recorder
*
* @author	303snowing
* @date		2017/09/09
*/
#pragma once
//包含项目文件，可在该文件中使用Unreal的库
#include "UnrealXunFeiSpeech.h"

#include <stdlib.h>
#include <windows.h>
#include <mmsystem.h>   
#include <process.h>
#include <errno.h>
// 自定义定义静态Log 包含WinRec.h使用
DEFINE_LOG_CATEGORY_STATIC(SnowingLog, Log, All);
DEFINE_LOG_CATEGORY_STATIC(SnowingWarning, Warning, All);
DEFINE_LOG_CATEGORY_STATIC(SnowingError, Error, All);

/* error code */
enum {
	RECORD_ERR_BASE = 0,
	RECORD_ERR_GENERAL,
	RECORD_ERR_MEMFAIL,
	RECORD_ERR_INVAL,
	RECORD_ERR_NOT_READY
};

/* recorder object. */
struct recorder {
	void(*on_data_ind)(char *data, unsigned long len, void *user_para);
	void * user_cb_para;
	volatile int state;		/* internal record state */

	void * wavein_hdl;
	void * rec_thread_hdl;
	void * bufheader;
	unsigned int bufcount;
};
//每个类中声明自己的Log标签，方便调试
DECLARE_LOG_CATEGORY_EXTERN(WinRec, Warning, All);

class FWinRec
{
	//将回调函数声明为友元，以方便访问私有方法
	friend static unsigned int  __stdcall record_thread_proc(void * para); /* the recording callback thread procedure */

public:
	FWinRec() = default;
	FWinRec(FString);
	~FWinRec();

private:
	void dbg_wave_header(WAVEHDR * buf);
	int create_callback_thread(void *thread_proc_para, HANDLE *thread_hdl_out);
	void close_callback_thread(HANDLE thread);
	int open_rec_device(int dev, WAVEFORMATEX *format, HANDLE thread, HWAVEIN *wave_hdl_out);
	int prepare_rec_buffer(HWAVEIN wi, WAVEHDR ** bufheader_out, unsigned int headercount, unsigned int bufsize);
	void free_rec_buffer(HWAVEIN wi, WAVEHDR *first_header, unsigned headercount);
	void close_rec_device(HWAVEIN wi);
	int start_record_internal(HWAVEIN wi, WAVEHDR *header, unsigned int bufcount);
	int stop_record_internal(HWAVEIN wi);
	void data_proc(struct recorder *rec, MSG *msg);
	int is_stopped_internal(struct recorder *rec);
	int open_recorder_internal(struct recorder * rec, unsigned int dev, WAVEFORMATEX * fmt);
	void close_recorder_internal(struct recorder *rec);

public:
	/**
	* @fn
	* @brief	Get the default input device ID
	*
	* @return	returns WAVE_MAPPER in windows.
	*/
	int get_default_input_dev();

	/**
	* @fn
	* @brief	Get the total number of active input devices.
	* @return	the number. 0 means no active device.
	*/
	unsigned int get_input_dev_num();

	/**
	* @fn
	* @brief	Create a recorder object.
	* @return	int			- Return 0 in success, otherwise return error code.
	* @param	out_rec		- [out] recorder object holder
	* @param	on_data_ind	- [in]	callback. called when data coming.
	* @param	user_cb_para	- [in] user params for the callback.
	* @see
	*/
	int create_recorder(struct recorder ** out_rec,
		void(*on_data_ind)(char *data, unsigned long len, void *user_para),
		void* user_cb_para);

	/**
	* @fn
	* @brief	Destroy recorder object. free memory.
	* @param	rec	- [in]recorder object
	*/
	void destroy_recorder(struct recorder *rec);

	/**
	* @fn
	* @brief	open the device.
	* @return	int			- Return 0 in success, otherwise return error code.
	* @param	rec			- [in] recorder object
	* @param	dev			- [in] device id, from 0.
	* @param	fmt			- [in] record format.
	* @see
	* 	get_default_input_dev()
	*/
	int open_recorder(struct recorder * rec, unsigned int dev, WAVEFORMATEX * fmt);

	/**
	* @fn
	* @brief	close the device.
	* @param	rec			- [in] recorder object
	*/

	void close_recorder(struct recorder *rec);

	/**
	* @fn
	* @brief	start record.
	* @return	int			- Return 0 in success, otherwise return error code.
	* @param	rec			- [in] recorder object
	*/
	int start_record(struct recorder * rec);

	/**
	* @fn
	* @brief	stop record.
	* @return	int			- Return 0 in success, otherwise return error code.
	* @param	rec			- [in] recorder object
	*/
	int stop_record(struct recorder * rec);

	/**
	* @fn
	* @brief	test if the recording has been stopped.
	* @return	int			- 1: stopped. 0 : recording.
	* @param	rec			- [in] recorder object
	*/
	int is_record_stopped(struct recorder *rec);
};
//定义一个静态变量供C++代码使用，以访问全局变量
static FWinRec * winrec = new FWinRec(FString("static winrec be created !"));


```
#### 2.创建`FSpeechRecoginzer`类
FSpeechRecoginzer类对应讯飞官方例子的speechrecoginzer.c文件，封装了语音在线听写功能，在Source/UnrealXunFeiSpeech/Public下创建`SpeechRecognizer.h`，在Source/UnrealXunFeiSpeech/Private下创建`SpeechRecognizer.cpp`，并在项目中添加到对应目录下
> **[SpeechRecoginzer.h][SpeechRecoginzer.h] | [SpeechRecoginzer.cpp][SpeechRecoginzer.cpp]**
> 基于录音接口和讯飞MSC接口封装一个MIC录音识别的模块
```C++
// Fill out your copyright notice in the Description page of Project Settings.
/*
@file
@brief 基于录音接口和讯飞MSC接口封装一个MIC录音识别的模块

@author		taozhang9
@date		2016/05/27
*/
#pragma once

#include <stdlib.h>
#include <windows.h>

#include "qisr.h"
#include "msp_cmn.h"
#include "msp_errors.h"

#include "WinRec.h"
#include "SpeechActor.h"

enum sr_audsrc
{
	SR_MIC,	/* write data from mic */
	SR_USER	/* write data from user by calling API */
};

#define DEFAULT_INPUT_DEVID     (-1)


#define E_SR_NOACTIVEDEVICE		1
#define E_SR_NOMEM				2
#define E_SR_INVAL				3
#define E_SR_RECORDFAIL			4
#define E_SR_ALREADY			5


struct speech_rec_notifier {
	void(*on_result)(const char *result, char is_last);
	void(*on_speech_begin)();
	void(*on_speech_end)(int reason);	/* 0 if VAD.  others, error : see E_SR_xxx and msp_errors.h  */
};

#define END_REASON_VAD_DETECT	0	/* detected speech done  */

struct speech_rec {
	enum sr_audsrc aud_src;  /* from mic or manual  stream write */
	struct speech_rec_notifier notif;
	const char * session_id;
	int ep_stat;
	int rec_stat;
	int audio_status;
	struct recorder *recorder;
	volatile int state;
	char * session_begin_params;
};

DECLARE_LOG_CATEGORY_EXTERN(SpeechRecoginzer, Warning, All);

//声明代理
//DECLARE_DELEGATE_RetVal(FString, OnGetResult)

class FSpeechRecoginzer
{
	friend static void iat_cb(char *data, unsigned long len, void *user_para);

public:
	FSpeechRecoginzer() = default;
	FSpeechRecoginzer(FString);
	virtual ~FSpeechRecoginzer();

	//OnGetResult GettedResult;

private:
	void end_sr_on_error(struct speech_rec *sr, int errcode);
	void end_sr_on_vad(struct speech_rec *sr);
	char * skip_space(char *s);
	int update_format_from_sessionparam(const char * session_para, WAVEFORMATEX *wavefmt);
	void wait_for_rec_stop(struct recorder *rec, unsigned int timeout_ms);

public:
	/* must init before start . devid = -1, then the default device will be used.
	devid will be ignored if the aud_src is not SR_MIC */
	int sr_init(struct speech_rec * sr, const char * session_begin_params, enum sr_audsrc aud_src, int devid, struct speech_rec_notifier * notifier);
	int sr_start_listening(struct speech_rec *sr);
	int sr_stop_listening(struct speech_rec *sr);
	/* only used for the manual write way. */
	int sr_write_audio_data(struct speech_rec *sr, char *data, unsigned int len);
	/* must call uninit after you don't use it */
	void sr_uninit(struct speech_rec * sr);

};

//定义一个静态变量供C++代码使用，以访问全局变量
static  FSpeechRecoginzer * speechrecoginzer = new FSpeechRecoginzer("static soeech recoginzer be created !");

```

#### 3. 创建`FXunFeiSpeech`类
FXunFeiSpeech类中封装了语音识别的执行方法，包含整体流程控住与事件控制，在Source/UnrealXunFeiSpeech/Public下创建`XunFeiSpeech.h`，在Source/UnrealXunFeiSpeech/Private下创建`XunFeiSpeech.cpp`，并在项目中添加到对应目录下
> **[XunFeiSpeech.h][XunFeiSpeech.h] | [XunFeiSpeech.cpp][XunFeiSpeech.cpp]**
> 语音听写(iFly Auto Transform)技术能够实时地将语音转换成对应的文字。
```C++
#pragma once

#include <conio.h>
#include "msp_cmn.h"
#include "msp_errors.h"
#include "SpeechRecoginzer.h"

/*
* 语音听写(iFly Auto Transform)技术能够实时地将语音转换成对应的文字。
*/
#define FRAME_LEN	640
#define BUFFER_SIZE	4096
// 识别状态类型
enum {
	EVT_START = 0,
	EVT_STOP,
	EVT_QUIT,
	EVT_TOTAL
};

DECLARE_LOG_CATEGORY_EXTERN(XunFeiSpeech, Warning, All);

class FXunFeiSpeech
{

public:
	//struct speech_rec iat;

public:

	FXunFeiSpeech();
	FXunFeiSpeech(FString);
	//事件触发，控制录音的开始、结束、与程序的退出
	void SetStart();
	void SetStop();
	void SetQuit();
	//整个流程控制
	void speech_mic(const char* session_beging_params);
	//将识别结果返回
	const char* get_result() const;

};
//定义静态实例方便其他C++代码使用
static FXunFeiSpeech * xunfeispeech = new FXunFeiSpeech(FString("static xunfeispeech be created !"));

```

#### 4. 创建`FSpeechTask`类
FSpeechTask类继承`FNonAbandonableTask`，用来将语音识别作为独立线程启动，避免在语音录入和识别时阻塞游戏主线程，在Source/UnrealXunFeiSpeech/Public下创建`SpeechTask.h`，在Source/UnrealXunFeiSpeech/Private下创建`SpeechTask.cpp`，并在项目中添加到对应目录下
> **[SpeechTask.h][SpeechTask.h] | [SpeechTask.cpp][SpeechTask.cpp]**
```C++
#pragma once

#include "XunFeiSpeech.h"
#include "AsyncWork.h"

class FSpeechTask : public FNonAbandonableTask
{

	friend class FAutoDeleteAsyncTask<FSpeechTask>;

	FSpeechTask()
	{
		UE_LOG(SnowingWarning, Warning, TEXT("Speech Task be Create !"));
	}

	void DoWork();
	FORCEINLINE TStatId GetStatId() const
	{
		RETURN_QUICK_DECLARE_CYCLE_STAT(FSpeechTask, STATGROUP_ThreadPoolAsyncTasks);
	}
};
```

### **编写ASpeechActor类**
`ASpeechActor`类为蓝图暴露操作方法，包含语音初始化、打开录音、停止录音和退出录音释放资源操作

* [SpeechActor.h][SpeechActor.h] 
```C++ 
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once
//包含SpeechTask，在初始化的时候，启动语音功能
#include "SpeechTask.h"

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "SpeechActor.generated.h"

UCLASS()
class UNREALXUNFEISPEECH_API ASpeechActor : public AActor
{
	GENERATED_BODY()
private:
	//存放语音识别结果
	FString Result;

public:	
	// Sets default values for this actor's properties
	ASpeechActor();

protected:
	// Called when the game starts or when spawned
	virtual void BeginPlay() override;

public:	
	// Called every frame
	virtual void Tick(float DeltaTime) override;

	UFUNCTION(BlueprintCallable, Category = "XunFei", meta = (DisplayName = "SpeechInit", Keywords = "Speech Recognition Initialization"))
		void SpeechInit();

	UFUNCTION(BlueprintCallable, Category = "XunFei", meta = (DisplayName = "SpeechOpen", Keywords = "Speech Recognition Open"))
		void SpeechOpen();

	UFUNCTION(BlueprintCallable, Category = "XunFei", meta = (DisplayName = "SpeechStop", Keywords = "Speech Recognition Stop"))
		void SpeechStop();

	UFUNCTION(BlueprintCallable, Category = "XunFei", meta = (DisplayName = "SpeechQuit", Keywords = "Speech Recognition Quit"))
		void SpeechQuit();

	UFUNCTION(BlueprintCallable, Category = "XunFei", meta = (DisplayName = "SpeechResult", Keywords = "Speech Recognition GetResult"))
		FString SpeechResult();
};

```

* [SpeechActor.cpp][SpeechActor.cpp] 
```C++
// Fill out your copyright notice in the Description page of Project Settings.
#pragma once

#include "SpeechActor.h"
//引入Unreal的Json库，用来解析识别结果(!!!注意在Build.cs文件的Module中加载Json)
/*
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "Json" });
*/
#include "Serialization/JsonReader.h"
#include "Dom/JsonObject.h"
#include "Serialization/JsonSerializer.h"



// Sets default values
ASpeechActor::ASpeechActor() :
	Result{}
{
 	// Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
	PrimaryActorTick.bCanEverTick = false;

}

// Called when the game starts or when spawned
void ASpeechActor::BeginPlay()
{
	Super::BeginPlay();
}

// Called every frame
void ASpeechActor::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

}

void ASpeechActor::SpeechInit()
{
	//创建一个SpeechTask任务实例
	FAutoDeleteAsyncTask<FSpeechTask>* SpeechTask = new FAutoDeleteAsyncTask<FSpeechTask>();

	if (SpeechTask)
	{
		//异步启动SpeechTask实例 会去单开线程异步执行SpeechTask中的DoWork方法
		SpeechTask->StartBackgroundTask();
	}
	else
	{
		UE_LOG(SnowingError, Error, TEXT("XunFei task object could not be create !"));
		return;
	}

	UE_LOG(SnowingWarning, Warning, TEXT("XunFei Task Stopped !"));
	return;
}

void ASpeechActor::SpeechOpen()
{
	xunfeispeech->SetStart();
	return;
}

void ASpeechActor::SpeechStop()
{
	xunfeispeech->SetStop();
	return;
}

void ASpeechActor::SpeechQuit()
{
	xunfeispeech->SetQuit();
	Sleep(300);//延迟等待资源释放完成
	return;
}

FString ASpeechActor::SpeechResult()
{	
	Result = FString(UTF8_TO_TCHAR(xunfeispeech->get_result()));
	//去掉讯飞生成结果中的标点符号json串
	FString LajiString("{\"sn\":2,\"ls\":true,\"bg\":0,\"ed\":0,\"ws\":[{\"bg\":0,\"cw\":[{\"sc\":0.00,\"w\":\"\"}]}]}");
	int32 LajiIndex = Result.Find(*LajiString);
	if (LajiIndex != -1)
	{
		Result.RemoveFromEnd(LajiString);
	}
	TSharedPtr<FJsonObject> JsonObject;
	TSharedRef< TJsonReader<TCHAR> > Reader = TJsonReaderFactory<TCHAR>::Create(Result);
	//解析并拼接结果 返回给调用者(蓝图)
	if (FJsonSerializer::Deserialize(Reader, JsonObject))
	{
		Result.Reset();
		TArray< TSharedPtr<FJsonValue> > TempArray = JsonObject->GetArrayField("ws");
		for (auto rs : TempArray)
		{
			Result.Append((rs->AsObject()->GetArrayField("cw"))[0]->AsObject()->GetStringField("w"));
		}
	}
	UE_LOG(SnowingError, Error, TEXT("%s"), *Result);
	
	return Result;
	
}


```

### **编写蓝图脚本示例**
这里就比较随意啦，在蓝图中构建一个SpeechActor即可使用其中的方法
![Blueprint][Blueprint]
注意：
1. **语音初始化在关卡运行时只需要执行一次，避免重复初始化**
2. **在关卡结束的时候，或者SpeechActor实例被销毁之前，需要执行SpeechQuit方法，释放语音资源，否则在本次游戏实例中无法再次初始化**
3. **在SpeechResult调用之前务必延迟至少0.3秒，等待语音识别结果完整返回**
4. **请务必使用Custom Event的形式调用SpeechResult，如果直接在游戏主线程中直接使用函数调用，会造成游戏卡帧**

> **补充**
如果出现识别准确率不够，或者对自己的词语不太友好，可以在官方平台的`应用管理>语音听写>个性化听写`页面上传热词文件，可以优化识别率。

[WinRec.h]: https://github.com/303snowing/UnrealXunFeiSpeech/blob/master/Source/UnrealXunFeiSpeech/Public/WinRec.h
[WinRec.cpp]: https://github.com/303snowing/UnrealXunFeiSpeech/blob/master/Source/UnrealXunFeiSpeech/Private/WinRec.cpp
[SpeechRecoginzer.h]: https://github.com/303snowing/UnrealXunFeiSpeech/blob/master/Source/UnrealXunFeiSpeech/Public/SpeechRecoginzer.h
[SpeechRecoginzer.cpp]: https://github.com/303snowing/UnrealXunFeiSpeech/blob/master/Source/UnrealXunFeiSpeech/Private/SpeechRecoginzer.cpp
[XunFeiSpeech.h]: https://github.com/303snowing/UnrealXunFeiSpeech/blob/master/Source/UnrealXunFeiSpeech/Public/XunFeiSpeech.h
[XunFeiSpeech.cpp]: https://github.com/303snowing/UnrealXunFeiSpeech/blob/master/Source/UnrealXunFeiSpeech/Private/XunFeiSpeech.cpp
[SpeechTask.h]: https://github.com/303snowing/UnrealXunFeiSpeech/blob/master/Source/UnrealXunFeiSpeech/Private/SpeechTask.h
[SpeechTask.cpp]: https://github.com/303snowing/UnrealXunFeiSpeech/blob/master/Source/UnrealXunFeiSpeech/Private/SpeechTask.cpp
[SpeechActor.h]: https://github.com/303snowing/UnrealXunFeiSpeech/blob/master/Source/UnrealXunFeiSpeech/Private/SpeechActor.h
[SpeechActor.cpp]: https://github.com/303snowing/UnrealXunFeiSpeech/blob/master/Source/UnrealXunFeiSpeech/Private/SpeechActor.cpp

[Blueprint]: http://img.blog.csdn.net/20170928161213935?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjc5MzEwNA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast