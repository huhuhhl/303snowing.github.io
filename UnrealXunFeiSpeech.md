�ؼ���: Unreal ��ͼ ����ʶ�� C++

---
*������Ŀ����ʹ������������Ϸ�������������ܶ�Ŀӣ����ڸ�����ˣ���¼һ�²ȿӾ���*

������demo��https://github.com/303snowing/UnrealXunFeiSpeech
demo������汾��4.17.1��ʹ��Visual Studio 2017 15.3

---
### **׼������**
1. �ȵ��ƴ�Ѷ�ɵ�����ƽ̨ע�ᡢ����һ��Ӧ��(�ⲽ���룬��Ϊֻ��ӵ��appid�������ض�Ӧ��sdk)
2. ��������������дsdk��http://www.xfyun.cn/services/voicedictation����ѹ
3. ����Unreal C++�հ���Ŀ������Ҫ����Start Content��ʾ������Ŀ����Ϊ`UnrealXunFeiSpeech`
4. ������Ŀ��������Ŀʵ��
5. ����һ���̳���Actor��C++�࣬�ɼ���ΪPublic

> Ѷ�ɿ��� Windows_voice_1166_59940824/libs �� Windows_voice_1166_59940824/bin Ŀ¼��,ͷ�ļ��� Windows_voice_1166_59940824/include Ŀ¼��

### **����Ѷ�ɿ�**
1. ����Ŀ��Ŀ¼�½���һ��XunFei�ļ���
2. ��Windows_voice_1166_59940824/libs��Windows_voice_1166_59940824/includeĿ¼������XunFei�ļ�����
3. ��Windows_voice_1166_59940824/bin�µ�`msc_x64.dll`�ļ���������Ŀ���̵�Binaries/Win64Ŀ¼��(��.dll�ļ�����ĿǨ�ƻ��������ж�����ȱ��)

### **����Ѷ�ɿ�����·��**
�༭Source/UnrealXunFeiSpeech/UnrealXunFeiSpeech.Build.cs�ļ�����������
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

		// �˴�ָ���ļ�����·��
        PrivateIncludePaths.Add("UnrealXunFeiSpeech/Private");
        PublicIncludePaths.Add("UnrealXunFeiSpeech/Public");
		//����Ѷ�ɾ�̬��
        PublicLibraryPaths.AddRange(new string[] { "..\\XunFei\\libs" });
        PublicAdditionalLibraries.AddRange(new string[] { "msc_x64.lib" });
		//����ļ�����·��
        PublicIncludePaths.AddRange(new string[] { "..\\XunFei\\include" });
    }
}

```

### ��д����
#### 1. ����`FWinRec`��
FWinRec���ӦѶ�ɹٷ����ӵ�winrec.c�ļ�,�Ƿ�װ��Windows¼�����ܣ���Source/UnrealXunFeiSpeech/Public�´���`WinRec.h`����Source/UnrealXunFeiSpeech/Private�´���`WinRec.cpp`��������Ŀ����ӵ���ӦĿ¼��
> **[WinRec.h][WinRec.h] | [WinRec.cpp][WinRec.cpp]**
> �����������ļ��У���ֱ�ӻ��߼�ӵİ���WinRec.h,������WinRec.h��������Զ����Log��ǩ��
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
//������Ŀ�ļ������ڸ��ļ���ʹ��Unreal�Ŀ�
#include "UnrealXunFeiSpeech.h"

#include <stdlib.h>
#include <windows.h>
#include <mmsystem.h>   
#include <process.h>
#include <errno.h>
// �Զ��嶨�徲̬Log ����WinRec.hʹ��
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
//ÿ�����������Լ���Log��ǩ���������
DECLARE_LOG_CATEGORY_EXTERN(WinRec, Warning, All);

class FWinRec
{
	//���ص���������Ϊ��Ԫ���Է������˽�з���
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
//����һ����̬������C++����ʹ�ã��Է���ȫ�ֱ���
static FWinRec * winrec = new FWinRec(FString("static winrec be created !"));


```
#### 2.����`FSpeechRecoginzer`��
FSpeechRecoginzer���ӦѶ�ɹٷ����ӵ�speechrecoginzer.c�ļ�����װ������������д���ܣ���Source/UnrealXunFeiSpeech/Public�´���`SpeechRecognizer.h`����Source/UnrealXunFeiSpeech/Private�´���`SpeechRecognizer.cpp`��������Ŀ����ӵ���ӦĿ¼��
> **[SpeechRecoginzer.h][SpeechRecoginzer.h] | [SpeechRecoginzer.cpp][SpeechRecoginzer.cpp]**
> ����¼���ӿں�Ѷ��MSC�ӿڷ�װһ��MIC¼��ʶ���ģ��
```C++
// Fill out your copyright notice in the Description page of Project Settings.
/*
@file
@brief ����¼���ӿں�Ѷ��MSC�ӿڷ�װһ��MIC¼��ʶ���ģ��

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

//��������
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

//����һ����̬������C++����ʹ�ã��Է���ȫ�ֱ���
static  FSpeechRecoginzer * speechrecoginzer = new FSpeechRecoginzer("static soeech recoginzer be created !");

```

#### 3. ����`FXunFeiSpeech`��
FXunFeiSpeech���з�װ������ʶ���ִ�з����������������̿�ס���¼����ƣ���Source/UnrealXunFeiSpeech/Public�´���`XunFeiSpeech.h`����Source/UnrealXunFeiSpeech/Private�´���`XunFeiSpeech.cpp`��������Ŀ����ӵ���ӦĿ¼��
> **[XunFeiSpeech.h][XunFeiSpeech.h] | [XunFeiSpeech.cpp][XunFeiSpeech.cpp]**
> ������д(iFly Auto Transform)�����ܹ�ʵʱ�ؽ�����ת���ɶ�Ӧ�����֡�
```C++
#pragma once

#include <conio.h>
#include "msp_cmn.h"
#include "msp_errors.h"
#include "SpeechRecoginzer.h"

/*
* ������д(iFly Auto Transform)�����ܹ�ʵʱ�ؽ�����ת���ɶ�Ӧ�����֡�
*/
#define FRAME_LEN	640
#define BUFFER_SIZE	4096
// ʶ��״̬����
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
	//�¼�����������¼���Ŀ�ʼ���������������˳�
	void SetStart();
	void SetStop();
	void SetQuit();
	//�������̿���
	void speech_mic(const char* session_beging_params);
	//��ʶ��������
	const char* get_result() const;

};
//���徲̬ʵ����������C++����ʹ��
static FXunFeiSpeech * xunfeispeech = new FXunFeiSpeech(FString("static xunfeispeech be created !"));

```

#### 4. ����`FSpeechTask`��
FSpeechTask��̳�`FNonAbandonableTask`������������ʶ����Ϊ�����߳�����������������¼���ʶ��ʱ������Ϸ���̣߳���Source/UnrealXunFeiSpeech/Public�´���`SpeechTask.h`����Source/UnrealXunFeiSpeech/Private�´���`SpeechTask.cpp`��������Ŀ����ӵ���ӦĿ¼��
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

### **��дASpeechActor��**
`ASpeechActor`��Ϊ��ͼ��¶��������������������ʼ������¼����ֹͣ¼�����˳�¼���ͷ���Դ����

* [SpeechActor.h][SpeechActor.h] 
```C++ 
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once
//����SpeechTask���ڳ�ʼ����ʱ��������������
#include "SpeechTask.h"

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "SpeechActor.generated.h"

UCLASS()
class UNREALXUNFEISPEECH_API ASpeechActor : public AActor
{
	GENERATED_BODY()
private:
	//�������ʶ����
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
//����Unreal��Json�⣬��������ʶ����(!!!ע����Build.cs�ļ���Module�м���Json)
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
	//����һ��SpeechTask����ʵ��
	FAutoDeleteAsyncTask<FSpeechTask>* SpeechTask = new FAutoDeleteAsyncTask<FSpeechTask>();

	if (SpeechTask)
	{
		//�첽����SpeechTaskʵ�� ��ȥ�����߳��첽ִ��SpeechTask�е�DoWork����
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
	Sleep(300);//�ӳٵȴ���Դ�ͷ����
	return;
}

FString ASpeechActor::SpeechResult()
{	
	Result = FString(UTF8_TO_TCHAR(xunfeispeech->get_result()));
	//ȥ��Ѷ�����ɽ���еı�����json��
	FString LajiString("{\"sn\":2,\"ls\":true,\"bg\":0,\"ed\":0,\"ws\":[{\"bg\":0,\"cw\":[{\"sc\":0.00,\"w\":\"\"}]}]}");
	int32 LajiIndex = Result.Find(*LajiString);
	if (LajiIndex != -1)
	{
		Result.RemoveFromEnd(LajiString);
	}
	TSharedPtr<FJsonObject> JsonObject;
	TSharedRef< TJsonReader<TCHAR> > Reader = TJsonReaderFactory<TCHAR>::Create(Result);
	//������ƴ�ӽ�� ���ظ�������(��ͼ)
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

### **��д��ͼ�ű�ʾ��**
����ͱȽ�������������ͼ�й���һ��SpeechActor����ʹ�����еķ���
![Blueprint][Blueprint]
ע�⣺
1. **������ʼ���ڹؿ�����ʱֻ��Ҫִ��һ�Σ������ظ���ʼ��**
2. **�ڹؿ�������ʱ�򣬻���SpeechActorʵ��������֮ǰ����Ҫִ��SpeechQuit�������ͷ�������Դ�������ڱ�����Ϸʵ�����޷��ٴγ�ʼ��**
3. **��SpeechResult����֮ǰ����ӳ�����0.3�룬�ȴ�����ʶ������������**
4. **�����ʹ��Custom Event����ʽ����SpeechResult�����ֱ������Ϸ���߳���ֱ��ʹ�ú������ã��������Ϸ��֡**

> **����**
�������ʶ��׼ȷ�ʲ��������߶��Լ��Ĵ��ﲻ̫�Ѻã������ڹٷ�ƽ̨��`Ӧ�ù���>������д>���Ի���д`ҳ���ϴ��ȴ��ļ��������Ż�ʶ���ʡ�

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