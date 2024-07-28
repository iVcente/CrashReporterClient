Crash Reporter is very useful for allowing players to send to you their crashes logs in a pretty straight forward way. It’s possible to customize its looks and texts to better match them to your game, while also making it a bit more user friendly. I like to use [BugSplat](https://www.bugsplat.com) as the tool/service for sending and analyzing crashes.

First things first, make sure you're using a version of Unreal Engine that's built from source code. Then, let's update the URL Crash Reporter uses to send the reports to. Originally, the reports are sent to Epic -- which doesn't make sense for this case --, but we'll change it so BugSplat can received them. In your Unreal Engine installation directory, go to `Engine/Programs/CrashReportClient/Config/DefaultEngine.ini` and set the `DataRouterUrl` variable as this:
```ini
[CrashReportClient]
CrashReportClientVersion=1.0
DataRouterUrl="https://{database}.bugsplat.com/post/ue4/{appName}/{appVersion}"
```

To start customizing your Crash Reporter, let's provide a custom image to the dialog at runtime. Create the directory `{UnrealSourceVersionPath}/Engine/Content/Slate/CrashReportClient`. In the directory just created, paste a copy of the image you plan to use for the dialog and name it as `Banner.png`. This file should be approximately 728px in width and 264px in height.

After this, we’ll focus in the engine module called `CrashReportClient`, which can be found under `{UnrealSourceVersionPath}/Engine/Source/Programs/CrashReportClient`. We’ll be making changes in three files in its `Private` directory: `CrashReportClientApp.cpp`, `CrashReportClientStyle.cpp`, and `SCrashReportClient.cpp`.

### CrashReportClientApp.cpp

Increase the dialog’s height by setting `InitialWindowDimensions` to 740 in X and 620 in Y like this:
```cpp
/** Default main window size */
const FVector2D InitialWindowDimensions(740, 620);
```
> CrashReportClientApp.cpp | Member Variable | Line [54](https://github.com/EpicGames/UnrealEngine/blob/463443057fb97f1af0d2951705324ce8818d2a55/Engine/Source/Programs/CrashReportClient/Private/CrashReportClientApp.cpp#L54)

### CrashReportClientStyle.cpp

The line of code below defines a new image brush located in the `Slate/CrashReportClient` directory with the file name `Banner.png` and a requested size of width 600px and height 264px. The image brush asset can now be referenced in the style set by using the key `Banner`.
```cpp
Style.Set("Banner", new IMAGE_BRUSH("CrashReportClient/Banner", FVector2D(600, 264)));
```
> CrashReportClientStyle.cpp | FCrashReportClientStyle::Create() | Line [74](https://github.com/EpicGames/UnrealEngine/blob/02dc8dbdd89f749cd5500376e9bb87271bf64848/Engine/Source/Programs/CrashReportClient/Private/CrashReportClientStyle.cpp#L74)

### SCrashReportClient.cpp
The last step is the dialog wording. There are preprocessor directives here to enable the Crash Reporter's default implementation in case you crash in the editor. The custom Crash Reporter is set to appear only in packaged builds. Finally, we're removing the non-symbolicated stack trace as it doesn’t make much sense to display it to users. For most players seeing an unsymbolicated stack trace is likely to confuse them and increase friction between them and sending a crash report.

```cpp
#if UE_BUILD_SHIPPING || UE_BUILD_DEVELOPMENT || UE_BUILD_DEBUG
    FText CrashDetailedMessage = LOCTEXT("CrashDetailed", "We are very sorry that this crash occurred. Our goal is to prevent crashes like this from occurring in the future. Please, help us track down and fix this crash by providing as much information as possible about what you were doing so that we may reproduce the crash and fix it quickly.\n\nThank you for your help in improving {GameName>!");
#else
    FText CrashDetailedMessage = LOCTEXT("CrashDetailed", "We are very sorry that this crash occurred. Our goal is to prevent crashes like this from occurring in the future. Please help us track down and fix this crash by providing detailed information about what you were doing so that we may reproduce the crash and fix it quickly. You can also log a Bug Report with us using the <a id=\"browser\" href=\"https://epicsupport.force.com/unrealengine/s/\" style=\"Hyperlink\">Bug Submission Form</> and work directly with support staff to report this issue.\n\nThanks for your help in improving the Unreal Engine.");
#endif
```
> SCrashReportClient.cpp | > SCrashReportClient::Construct | Line [50](https://github.com/EpicGames/UnrealEngine/blob/463443057fb97f1af0d2951705324ce8818d2a55/Engine/Source/Programs/CrashReportClient/Private/SCrashReportClient.cpp#L50)

```cpp
void SCrashReportClient::ConstructDetailedDialog(const TSharedRef<FCrashReportClient>& Client, const FText& CrashDetailedMessage)
{
    auto CrashedAppName = FPrimaryCrashProperties::Get()->IsValid() ? FPrimaryCrashProperties::Get()->GameName : TEXT("");

    // Set the text displaying the name of the crashed app, if available
    const FText CrashedAppText = CrashedAppName.IsEmpty() ?
        LOCTEXT( "CrashedAppNotFound", "An unknown process has crashed" ) :
        LOCTEXT( "CrashedAppUnreal", "An Unreal process has crashed: " );

    const FText CrashReportDataText = FText::Format( 
        LOCTEXT( "CrashReportData", "Crash reports comprise diagnostics files (<a id=\"browser\" href=\"{0}\" style=\"Richtext.Hyperlink\">click here to view directory</>) and the following summary information: " ),
        FText::FromString( CrashReportClient->GetCrashDirectory()) );

    #if UE_BUILD_SHIPPING || UE_BUILD_DEVELOPMENT || UE_BUILD_DEBUG 
        ChildSlot
        [
            SNew(SBorder)
            .BorderImage(FCrashReportClientStyle::Get().GetBrush("ToolPanel.GroupBorder"))
            [
                SNew(SVerticalBox)

                // Stuff anchored to the top
                +SVerticalBox::Slot()
                .AutoHeight()
                .Padding(FMargin(4, 10))
                [
                    SNew(SImage)
                    .Image(FCrashReportClientStyle::Get().GetBrush("Banner"))
                ]
                +SVerticalBox::Slot()
                .AutoHeight()
                .Padding(4)
                [
                    SNew(STextBlock)
                    .TextStyle(FCrashReportClientStyle::Get(), "Title")
                    .Text(FText::FromString("Oh, no! It seems the game has crashed."))
                ]

                +SVerticalBox::Slot()
                .AutoHeight()
                .Padding( FMargin( 4, 10 ) )
                [
                    SNew( SRichTextBlock )
                    .Text(CrashDetailedMessage)
                    .AutoWrapText(true)
                    + SRichTextBlock::HyperlinkDecorator( TEXT("browser"), FSlateHyperlinkRun::FOnClick::CreateStatic( &OnBrowserLinkClicked ) )
                ]

                +SVerticalBox::Slot()
                .Padding(FMargin(4, 10, 4, 4))
                [
                    SNew(SSplitter)
                    .Orientation(Orient_Vertical)
                    +SSplitter::Slot()
                    .Value(0.3f)
                    [
                        SNew( SOverlay )
                        + SOverlay::Slot()
                        [
                            SAssignNew( CrashDetailsInformation, SMultiLineEditableTextBox )
                            .Style( &FCrashReportClientStyle::Get().GetWidgetStyle<FEditableTextBoxStyle>( "NormalEditableTextBox" ) )
                            .OnTextCommitted( CrashReportClient.ToSharedRef(), &FCrashReportClient::UserCommentChanged )
                            .OnTextChanged( this, &SCrashReportClient::OnUserCommentTextChanged)
                            .Font( FCoreStyle::GetDefaultFontStyle("Regular", 9) )
                            .AutoWrapText( true )
                            .BackgroundColor( FSlateColor( FLinearColor::Black ) )
                            .ForegroundColor( FSlateColor( FLinearColor::White * 0.8f ) )
                        ]

                        // HintText is not implemented in SMultiLineEditableTextBox, so this is a workaround.
                        + SOverlay::Slot()
                        [
                            SNew(STextBlock)
                            .Margin( FMargin(4,2,0,0) )
                            .Font( FCoreStyle::GetDefaultFontStyle("Italic", 9) )
                            .ColorAndOpacity( FSlateColor( FLinearColor::White * 0.5f ) )
                            .Text(LOCTEXT( "CrashProvide", "Please, if applicable, provide any information about what you were doing when the crash occurred." ))
                            .Visibility( this, &SCrashReportClient::IsHintTextVisible )
                        ]    
                    ]
                ]

                +SVerticalBox::Slot()
                .AutoHeight()
                [
                    SNew(SRichTextBlock)
                    .Margin(FMargin(4, 2, 0, 8))
                    .TextStyle(&FCrashReportClientStyle::Get().GetWidgetStyle<FTextBlockStyle>("CrashReportDataStyle"))
                    .Text(CrashReportDataText)
                    .AutoWrapText(true)
                    .DecoratorStyleSet(&FCrashReportClientStyle::Get())

                    +SRichTextBlock::HyperlinkDecorator(TEXT("browser"), FSlateHyperlinkRun::FOnClick::CreateStatic(&OnViewCrashDirectory))
                ]

                +SVerticalBox::Slot()
                .AutoHeight()
                .Padding( FMargin( 4, 12, 4, 4 ) )
                [
                    SNew( SHorizontalBox )
                    .Visibility( FCrashReportCoreConfig::Get().GetHideLogFilesOption() ? EVisibility::Collapsed : EVisibility::Visible )
                    + SHorizontalBox::Slot()
                    .AutoWidth()
                    .VAlign( VAlign_Center )
                    [
                        SNew( SCheckBox )
                        .IsChecked( FCrashReportCoreConfig::Get().GetSendLogFile() ? ECheckBoxState::Checked : ECheckBoxState::Unchecked )
                        .OnCheckStateChanged( CrashReportClient.ToSharedRef(), &FCrashReportClient::SendLogFile_OnCheckStateChanged )
                    ]

                    + SHorizontalBox::Slot()
                    .FillWidth( 1.0f )
                    .VAlign( VAlign_Center )
                    [
                        SNew( STextBlock )
                        .AutoWrapText( true )
                        .Text( LOCTEXT( "IncludeLogs", "Include log files with submission. I understand that logs contain some personal information such as my system and user name." ) )
                    ]
                ]

                // Stuff anchored to the bottom
                +SVerticalBox::Slot()
                .AutoHeight()
                .Padding( FMargin(4, 4+16, 4, 4) )
                [
                    SNew(SHorizontalBox)
        
                    + SHorizontalBox::Slot()
                    .HAlign( HAlign_Center )
                    .VAlign( VAlign_Center )
                    .AutoWidth()
                    .Padding( FMargin( 0 ) )
                    [
                        SNew( SButton )
                        .ContentPadding( FMargin( 8, 2 ) )
                        .Text( LOCTEXT( "CloseWithoutSending", "Close Without Sending" ) )
                        .OnClicked( Client, &FCrashReportClient::CloseWithoutSending )
                        .Visibility(FCrashReportCoreConfig::Get().IsAllowedToCloseWithoutSending() ? EVisibility::Visible : EVisibility::Hidden)
                    ]

                    +SHorizontalBox::Slot()
                    .FillWidth(1.0f)
                    .HAlign(HAlign_Left)
                    .VAlign(VAlign_Center)
                    .Padding(0)
                    [            
                        SNew(SSpacer)
                    ]

                    +SHorizontalBox::Slot()
                    .HAlign(HAlign_Center)
                    .VAlign(VAlign_Center)
                    .AutoWidth()
                    .Padding( FMargin(6) )
                    [
                        SNew(SButton)
                        .ContentPadding( FMargin(8,2) )
                        .Text(LOCTEXT("Send", "Send and Close"))
                        .OnClicked(Client, &FCrashReportClient::Submit)
                        .IsEnabled(this, &SCrashReportClient::IsSendEnabled)
                    ]

                    +SHorizontalBox::Slot()
                    .HAlign(HAlign_Center)
                    .VAlign(VAlign_Center)
                    .AutoWidth()
                    .Padding( FMargin(0) )
                    [
                        SNew(SButton)
                        .ContentPadding( FMargin(8,2) )
                        .Text(LOCTEXT("SendAndRestartEditor", "Send and Restart"))
                        .OnClicked(Client, &FCrashReportClient::SubmitAndRestart)
                        .IsEnabled(this, &SCrashReportClient::IsSendEnabled)
                        .Visibility( bHideSubmitAndRestart || FCrashReportCoreConfig::Get().GetHideRestartOption() ? EVisibility::Collapsed : EVisibility::Visible )
                    ]
                ]
            ]
        ];
    #else
        ChildSlot
        [
            SNew(SBorder)
            .BorderImage(FCrashReportClientStyle::Get().GetBrush("ToolPanel.GroupBorder"))
            [
                SNew(SVerticalBox)

                // Stuff anchored to the top
                +SVerticalBox::Slot()
                .AutoHeight()
                .Padding(4)
                [
                    SNew(SHorizontalBox)
                    +SHorizontalBox::Slot()
                    .AutoWidth()
                    [
                        SNew(STextBlock)
                        .TextStyle(FCrashReportClientStyle::Get(), "Title")
                        .Text(CrashedAppText)
                    ]

                    +SHorizontalBox::Slot()
                    .AutoWidth()
                    [
                        SNew(STextBlock)
                        .TextStyle(FCrashReportClientStyle::Get(), "Title")
                        .Text(FText::FromString(CrashedAppName))
                    ]
                ]

                +SVerticalBox::Slot()
                .AutoHeight()
                .Padding( FMargin( 4, 10 ) )
                [
                    SNew( SRichTextBlock )
                    .Text(CrashDetailedMessage)
                    .AutoWrapText(true)
                    + SRichTextBlock::HyperlinkDecorator( TEXT("browser"), FSlateHyperlinkRun::FOnClick::CreateStatic( &OnBrowserLinkClicked ) )
                ]

                +SVerticalBox::Slot()
                .Padding(FMargin(4, 10, 4, 4))
                [
                    SNew(SSplitter)
                    .Orientation(Orient_Vertical)
                    +SSplitter::Slot()
                    .Value(0.3f)
                    [
                        SNew( SOverlay )
                        + SOverlay::Slot()
                        [
                            SAssignNew( CrashDetailsInformation, SMultiLineEditableTextBox )
                            .Style( &FCrashReportClientStyle::Get().GetWidgetStyle<FEditableTextBoxStyle>( "NormalEditableTextBox" ) )
                            .OnTextCommitted( CrashReportClient.ToSharedRef(), &FCrashReportClient::UserCommentChanged )
                            .OnTextChanged( this, &SCrashReportClient::OnUserCommentTextChanged)
                            .Font( FCoreStyle::GetDefaultFontStyle("Regular", 9) )
                            .AutoWrapText( true )
                            .BackgroundColor( FSlateColor( FLinearColor::Black ) )
                            .ForegroundColor( FSlateColor( FLinearColor::White * 0.8f ) )
                        ]

                        // HintText is not implemented in SMultiLineEditableTextBox, so this is a workaround.
                        + SOverlay::Slot()
                        [
                            SNew(STextBlock)
                            .Margin( FMargin(4,2,0,0) )
                            .Font( FCoreStyle::GetDefaultFontStyle("Italic", 9) )
                            .ColorAndOpacity( FSlateColor( FLinearColor::White * 0.5f ) )
                            .Text( LOCTEXT( "CrashProvide", "Please provide detailed information about what you were doing when the crash occurred." ) )
                            .Visibility( this, &SCrashReportClient::IsHintTextVisible )
                        ]    
                    ]

                    +SSplitter::Slot()
                    .Value(0.7f)
                    [
                        SNew(SVerticalBox)
                        + SVerticalBox::Slot()
                        .AutoHeight()
                        [
                            SNew(SOverlay)

                            + SOverlay::Slot()            
                            [
                                SNew(SColorBlock)
                                .Color(FLinearColor::Black)
                            ]

                            + SOverlay::Slot()
                            [
                                SNew( SRichTextBlock )
                                .Margin( FMargin( 4, 2, 0, 8 ) )
                                .TextStyle( &FCrashReportClientStyle::Get().GetWidgetStyle<FTextBlockStyle>( "CrashReportDataStyle" ) )
                                .Text( CrashReportDataText )
                                .AutoWrapText( true )
                                .DecoratorStyleSet( &FCrashReportClientStyle::Get() )
                                + SRichTextBlock::HyperlinkDecorator( TEXT( "browser" ), FSlateHyperlinkRun::FOnClick::CreateStatic( &OnViewCrashDirectory ) )
                            ]
                        ]

                        + SVerticalBox::Slot()
                        .FillHeight(0.7f)
                        [
                            SNew(SOverlay)
            
                            + SOverlay::Slot()
                            [
                                SNew( SMultiLineEditableTextBox )
                                .Style( &FCrashReportClientStyle::Get().GetWidgetStyle<FEditableTextBoxStyle>( "NormalEditableTextBox" ) )
                                .Font( FCoreStyle::GetDefaultFontStyle("Regular", 8) )
                                .AutoWrapText( false )
                                .IsReadOnly( true )
                                .ReadOnlyForegroundColor( FSlateColor( FLinearColor::White * 0.8f) )
                                .BackgroundColor( FSlateColor( FLinearColor::Black ) )
                                .ForegroundColor( FSlateColor( FLinearColor::White * 0.8f ) )
                                .Text( Client, &FCrashReportClient::GetDiagnosticText )
                            ]


                            + SOverlay::Slot()
                            .HAlign(HAlign_Center)
                            .VAlign(VAlign_Center)
                            [
                                SNew(SThrobber)
                                .Visibility(CrashReportClient.ToSharedRef(), &FCrashReportClient::IsThrobberVisible)
                                .NumPieces(5)
                            ]
                        ]
                    ]
                ]

                +SVerticalBox::Slot()
                .AutoHeight()
                .Padding( FMargin( 4, 12, 4, 4 ) )
                [
                    SNew( SHorizontalBox )
                    .Visibility( FCrashReportCoreConfig::Get().GetHideLogFilesOption() ? EVisibility::Collapsed : EVisibility::Visible )
                    + SHorizontalBox::Slot()
                    .AutoWidth()
                    .VAlign( VAlign_Center )
                    [
                        SNew( SCheckBox )
                        .IsChecked( FCrashReportCoreConfig::Get().GetSendLogFile() ? ECheckBoxState::Checked : ECheckBoxState::Unchecked )
                        .OnCheckStateChanged( CrashReportClient.ToSharedRef(), &FCrashReportClient::SendLogFile_OnCheckStateChanged )
                    ]

                    + SHorizontalBox::Slot()
                    .FillWidth( 1.0f )
                    .VAlign( VAlign_Center )
                    [
                        SNew( STextBlock )
                        .AutoWrapText( true )
                        .Text( LOCTEXT( "IncludeLogs", "Include log files with submission. I understand that logs contain some personal information such as my system and user name." ) )
                    ]
                ]

                +SVerticalBox::Slot()
                .AutoHeight()
                .Padding( FMargin( 4, 4 ) )
                [
                    SNew(SHorizontalBox)

                    +SHorizontalBox::Slot()
                    .AutoWidth()
                    .VAlign(VAlign_Center)
                    [
                        SNew(SCheckBox)
                        .IsChecked( FCrashReportCoreConfig::Get().GetAllowToBeContacted() ? ECheckBoxState::Checked : ECheckBoxState::Unchecked )
                        .IsEnabled( !FEngineBuildSettings::IsInternalBuild() )
                        .OnCheckStateChanged(CrashReportClient.ToSharedRef(), &FCrashReportClient::AllowToBeContacted_OnCheckStateChanged)
                    ]

                    
                    +SHorizontalBox::Slot()
                    .FillWidth(1.0f)
                    .VAlign(VAlign_Center)
                    [
                        SNew(STextBlock)
                        .AutoWrapText(true)
                        .IsEnabled( !FEngineBuildSettings::IsInternalBuild() )
                        .Text(LOCTEXT("IAgree", "I agree to be contacted by Epic Games via email if additional information about this crash would help fix it."))
                    ]
                ]

                // Stuff anchored to the bottom
                +SVerticalBox::Slot()
                .AutoHeight()
                .Padding( FMargin(4, 4+16, 4, 4) )
                [
                    SNew(SHorizontalBox)
        
                    + SHorizontalBox::Slot()
                    .HAlign( HAlign_Center )
                    .VAlign( VAlign_Center )
                    .AutoWidth()
                    .Padding( FMargin( 0 ) )
                    [
                        SNew( SButton )
                        .ContentPadding( FMargin( 8, 2 ) )
                        .Text( LOCTEXT( "CloseWithoutSending", "Close Without Sending" ) )
                        .OnClicked( Client, &FCrashReportClient::CloseWithoutSending )
                        .Visibility(FCrashReportCoreConfig::Get().IsAllowedToCloseWithoutSending() ? EVisibility::Visible : EVisibility::Hidden)
                    ]

                    +SHorizontalBox::Slot()
                    .FillWidth(1.0f)
                    .HAlign(HAlign_Left)
                    .VAlign(VAlign_Center)
                    .Padding(0)
                    [            
                        SNew(SSpacer)
                    ]

                    +SHorizontalBox::Slot()
                    .HAlign(HAlign_Center)
                    .VAlign(VAlign_Center)
                    .AutoWidth()
                    .Padding( FMargin(6) )
                    [
                        SNew(SButton)
                        .ContentPadding( FMargin(8,2) )
                        .Text(LOCTEXT("Send", "Send and Close"))
                        .OnClicked(Client, &FCrashReportClient::Submit)
                        .IsEnabled(this, &SCrashReportClient::IsSendEnabled)
                    ]

                    +SHorizontalBox::Slot()
                    .HAlign(HAlign_Center)
                    .VAlign(VAlign_Center)
                    .AutoWidth()
                    .Padding( FMargin(0) )
                    [
                        SNew(SButton)
                        .ContentPadding( FMargin(8,2) )
                        .Text(LOCTEXT("SendAndRestartEditor", "Send and Restart"))
                        .OnClicked(Client, &FCrashReportClient::SubmitAndRestart)
                        .IsEnabled(this, &SCrashReportClient::IsSendEnabled)
                        .Visibility( bHideSubmitAndRestart || FCrashReportCoreConfig::Get().GetHideRestartOption() ? EVisibility::Collapsed : EVisibility::Visible )
                    ]
                ]
            ]
        ];
    #endif
}
```
> SCrashReportClient.cpp | SCrashReportClient::ConstructDetailedDialog() 

References
---
* [How to customize your Unreal Engine Crash Report Client | BugSplat](https://blog.bugsplat.com/customizing-the-unreal-engine-crash-report-client/)
