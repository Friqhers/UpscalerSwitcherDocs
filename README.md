# UpscalerSwitcher Plugin for Unreal Engine

The `UpscalerSwitcher` is a plugin for Unreal Engine that allows dynamic switching between different upscaling methods such as FSR and DLSS.

## 🔧Features

- Enable/Disable FSR and DLSS upscaling.
- Switch between different upscaling methods at runtime.
- Adjust quality settings for each upscaling method.

## 📦Installation

1. Clone or download the `UpscalerSwitcher` plugin into the `Plugins` directory of your Unreal Engine project.
2. Add the following line to your project's `.uproject` file to include the plugin:

    ```json
    "Plugins": [
        {
            "Name": "UpscalerSwitcher",
            "Enabled": true
        }
    ]
    ```

3. Rebuild your project to include the plugin.
4. Set your GameUserSettings class to UpscalerGameUserSettings in Project Settings->Engine-General Settings->GameUserSettingsClass.
5. To use DLSS and FSR features, make sure to install their respective plugins:
    🔷 DLSS Plugin:
    https://developer.nvidia.com/rtx/dlss
    🔶 FSR Plugin:
    https://gpuopen.com/learn/ue-fsr3/

## Usage

### Enabling/Disabling Upscaling

You can enable or disable upscaling methods using the `UUpscalerSwitcherUtils` class:

```cpp
#include "UpscalerSwitcherUtils.h"

// If another upscaler (e.g., DLSS) is currently enabled, it will be disabled before enabling FSR
// Enable saved FSR mode 
UUpscalerSwitcherUtils::ApplyFSR(FFSRModeInformation(Settings->GetFSRQualityMode(),
                                 Settings->GetFSRFrameGenEnabled()));

// This will first disable any previously enabled upscaling method (e.g., FSR) and then enable DLSS
// Enable saved DLSS mode
UUpscalerSwitcherUtils::ApplyDLSS(FDLSSModeInformation(Settings->GetDLSSQualityMode(),
                                    Settings->GetDLSSOptimalScreenPercentage(), Settings->GetDLSSFrameGenEnabled()));

// Disable all upscaling methods (FSR, DLSS, etc.), reverting to native resolution rendering
UUpscalerSwitcherUtils::DisableUpscaling();
```
### Adjusting Quality Settings

#### Adjusting Quality Setting for DLSS

Example c++ code to adjust the quality setting for the DLSS:
```cpp
#include "UpscalerGameUserSettings.h"
#include "UpscalerSwitcherUtils.h"
#include "DLSSLibrary.h" // PS. Dont forget to add DLSSBlueprint module in to your project.Build.cs like PrivateDependencyModuleNames.AddRange(new string[] {"DLSSBlueprint"});
 
// Get settings class
UUpscalerGameUserSettings* Settings = UUpscalerGameUserSettings::GetUpscalerGameUserSettings();

// Set DLSS quality mode to performance
Settings->SetDLSSQualityMode(UDLSSMode_Custom::Performance);

// Get optimal screen percentage for the quality mode
bool bIsSupported, bIsFixedScreenPercentage;
float OptimalScreenPercentage, MinScreenPercentage, MaxScreenPercentage, OptimalSharpness;
UDLSSLibrary::GetDLSSModeInformation(
    Settings->GetDLSSQualityMode(),
    Settings->GetScreenResolution(),
    bIsSupported,OptimalScreenPercentage,
    bIsFixedScreenPercentage,MinScreenPercentage,
    MaxScreenPercentage, OptimalSharpness
    );

// Save the optimal screen percentage
Settings->SetDLSSOptimalScreenPercentage(OptimalScreenPercentage);

// Apply the current saved upscaler (DLSS) by Util function
UUpscalerSwitcherUtils::ApplySavedUpscaler();

// Also you can call the ApplySettings function of UpscalerGameUserSettings which will also call ApplySavedUpscaler
Settings->ApplySettings(false);
```

Example Blueprint usage:
![dlss_usage](https://github.com/user-attachments/assets/6590ab37-b056-4543-ba21-e2540e1a62a6)

#### Adjusting Quality Setting for FSR

Example C++ code to adjust the quality setting for the FSR:
```cpp
#include "UpscalerGameUserSettings.h"
#include "UpscalerSwitcherUtils.h"
#include "DLSSLibrary.h" // PS. Dont forget to add DLSSBlueprint module in to your project.Build.cs like PrivateDependencyModuleNames.AddRange(new string[] {"DLSSBlueprint"});
 
// Get settings class
UUpscalerGameUserSettings* Settings = UUpscalerGameUserSettings::GetUpscalerGameUserSettings();

// Set FSR quality mode to performance
Settings->SetFSRQualityMode(EFFXFSR3QualityMode_Custom::Performance);

// Set FSR frame gen enabled
Settings->SetFSRFrameGenEnabled(true);

// Apply the current saved upscaler (DLSS)
UUpscalerSwitcherUtils::ApplySavedUpscaler();
// Also you can call the ApplySettings function of UpscalerGameUserSettings which will also call ApplySavedUpscaler
Settings->ApplySettings(false);
```

Example Blueprint usage:
![fsr_usage](https://github.com/user-attachments/assets/3ec4227f-fd51-46fc-a744-6c4a14fa53da)



### Automatically Apply the Saved Upscaling Method on Game Start

To ensure your game uses the player's previously selected upscaling method (e.g., DLSS, FSR, or Native resolution) when starting up, you should apply the saved preferences early in the game's lifecycle. A reliable place to do this is within your custom `GameInstance` class.

This way, the selected upscaling method is restored automatically and seamlessly, without requiring any manual input from the player every time the game launches.

Follow these steps:

1. Open your `GameInstance` header file and define the `Init` and `PostEngineInit` as follows:

```h
UCLASS()
class YOURPROJECT_API UYourGameInstance : public UGameInstance
{
    GENERATED_BODY()

public:
    virtual void Init() override;
protected:
    virtual void PostEngineInit();
};
```

2. In the corresponding `GameInstance` source file, implement the  `Init` and `PostEngineInit` function:

```cpp

void UYourGameInstance::Init()
{
    Super::Init();
    FCoreDelegates::OnPostEngineInit.AddUObject(this, &UYourGameInstance::PostEngineInit);
}

void UYourGameInstance::PostEngineInit()
{
    // Wait one frame
    FTSTicker::GetCoreTicker().AddTicker(FTickerDelegate::CreateLambda([this](float DeltaTime)
    {
       if (UUpscalerGameUserSettings* Settings = UUpscalerGameUserSettings::GetUpscalerGameUserSettings())
	   Settings->ApplySettings();
       }
        return false; // Don't repeat
    }), 0.0f);
}
```
