## UE4 C++ 初探
---
*声明：笔者引擎版本为 Unreal Engine 4.16.1*

从打印 Hello World 和打印基本变量开始：

工程的建立和基本代码的编写就不再赘述了，网上有很多，我看的是这个:
<http://blog.csdn.net/u011326794/article/details/47706959>

可以在新建的AActor中的`Tick`或者`BeginPlay`中使用 ***GEngine*** 的 AddOnScreenDebugMessage 进行输出，GEngine 是 [UEngine][] 类型的一个全局指针，被声明在 Engine.h 中，源代码如下：

<font color=green>Engine.h</font>
```C++
/** Global engine pointer. Can be 0 so don't use without checking. */
extern ENGINE_API class UEngine*			GEngine;
```
在 Engine.cpp 中对 GEngine 进行了初始化：

<font color=green>Engine.cpp</font>
```C++
/**
 * Global engine pointer. Can be 0 so don't use without checking.
 */
ENGINE_API UEngine*	GEngine = NULL;
```

* UE4 在屏幕上打印文字的函数原型如下：
```C++
void UEngine::AddOnScreenDebugMessage(
		uint64 Key,
		float TimeToDisplay,    // 消息在屏幕上显示的时间
		FColor DisplayColor,    // 文字的颜色
		const FString & DebugMessage,    // 显示内容
		bool bNewrOnTop,
		const FVector2D & TextScale
)

void UEngine::AddOnScreenDebugMessage(
		int32 Key,
		float TimeToDisplay,    // 消息在屏幕上显示的时间
		FColor DisplayColor,    // 文字的颜色
		const FString & DebugMessage,    // 显示内容
		bool bNewrOnTop,
		const FVector2D & TextScale
)
```
官方文档的例子在这里：
```C++
void AHelloWorldPrinter::Tick(float DeltaTime)  
{  
    Super::Tick(DeltaTime); //Call parent class Tick  

    static const FString ScrollingMessage(TEXT("Hello World: "));  

    if (GEngine)  
    {  
        const int32 AlwaysAddKey = -1; // Passing -1 means that we will not try and overwrite an   
                                       // existing message, just add a new one  
        GEngine->AddOnScreenDebugMessage(AlwaysAddKey, 0.5f, FColor::Yellow, ScrollingMessage + FString::FromInt(MyNumber));  

        const int32 MyNumberKey = 0; // Not passing -1 so each time through we will update the existing message instead  
                                     // of making a new one  
        GEngine->AddOnScreenDebugMessage(MyNumberKey, 5.f, FColor::Yellow, FString::FromInt(MyNumber));  

        ++MyNumber; // Increase MyNumber so we can see it change on screen  
    }  
}
```

* 贴出一个Demo：
```C++
// Called every frame
void AFloatingActor::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
	
	FVector NewLocation = GetActorLocation();
	if (GEngine)
	{
		GEngine->AddOnScreenDebugMessage(-1, 8.f, FColor::Blue, TEXT("Hello World !"));
		GEngine->AddOnScreenDebugMessage(0, 3.f, FColor::Red, *NewLocation.ToString());
	}
}
```
* *代码说明：*
	* [Tick][] 在类创建的时候由编辑器自动创建
	* [GetActorLocation()][] 用来获取 Actor 当前的 Location，返回的是一个 [FVector][] 结构体类型
	* 使用`FVector::ToString()`将 NewLocation 由 FVector 转为 FString

* *笔者的代码执行结果如下：*
	![笔者的代码执行结果](http://img.blog.csdn.net/20170808145247130?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjc5MzEwNA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
* *补充：*
	* [FString][] 的`FString::SanitizeFloat(double Infloat)`将 double 类型转为 FString 类型，[FString::SanitizeFloat()](https://docs.unrealengine.com/latest/INT/API/Runtime/Core/Containers/FString/SanitizeFloat/index.html "FString::SanitizeFloat()官方文档")

<!-- 引用链接 START -->
[UEngine]: https://docs.unrealengine.com/latest/INT/API/Runtime/Engine/Engine/UEngine/index.html "UEngine官方文档"

[Tick]: https://docs.unrealengine.com/latest/INT/API/Runtime/Engine/GameFramework/AActor/Tick/1/index.html "AActor::Tick官方文档"

[GetActorLocation()]: https://docs.unrealengine.com/latest/INT/API/Runtime/Engine/GameFramework/AActor/GetActorLocation/index.html "AActor::GetActorLocation()官方文档"

[FVector]: https://docs.unrealengine.com/latest/INT/API/Runtime/Core/Math/FVector/index.html "FVector官方文档"

[FString]: https://docs.unrealengine.com/latest/INT/API/Runtime/Core/Containers/FString/index.html "FString官方文档"
[`FString::SanitizeFloat(double Infloat)`]: https://docs.unrealengine.com/latest/INT/API/Runtime/Core/Containers/FString/SanitizeFloat/index.html

<!-- 引用链接 END -->
