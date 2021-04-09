---
title: Custom widgets in Unreal
tags: ue4 c++
---
There are two classes involved in UMG widgets:
* A UWidget-derived class, that you interact with in the editor
* An SCompoundWidget class, that handles the low-level functionality and rendering

Let's take a look at a simple widget that renders a slice, with a given start angle and arc size.
We can control the color and opacity, and optionally use an image for rendering it.

![Slice](/images/CustomWidget/Slice.png)

Note that if you just want to look at the final code, you can get it from the GitHub repo:

[https://github.com/snorristurluson/CustomWidget](https://github.com/snorristurluson/CustomWidget)

### Minimal UMG class
First, we'll add the classes without any functionality and make sure we see the new widget show
up in the editor palette.

Here is the header file for a minimal UMG widget:
```cpp
UCLASS()
class CUSTOMWIDGET_API USlice : public UWidget
{
  GENERATED_BODY()
public:
  virtual void ReleaseSlateResources(bool bReleaseChildren) override;

protected:
  virtual TSharedRef<SWidget> RebuildWidget() override;
  TSharedPtr<SSlateSlice> MySlice;
};
```
And the .cpp file:
```cpp
void USlice::ReleaseSlateResources(bool bReleaseChildren)
{
  MySlice.Reset();
}

TSharedRef<SWidget> USlice::RebuildWidget()
{
  MySlice = SNew(SSlateSlice);
  return MySlice.ToSharedRef();
}
```
Note that the UMG widget class holds a Slate widget instance. Those two functions always have to
be implemented when creating a new widget - *RebuildWidget* constructs the Slate widget, and
*ReleaseSlateResources* clears them up again.

### Minimal Slate class
Here is the Slate class that the UMG widget class references:
```cpp
class CUSTOMWIDGET_API SSlateSlice : public SCompoundWidget
{
public:
  SLATE_BEGIN_ARGS(SSlateSlice)
  {}
  SLATE_END_ARGS()

  void Construct(const FArguments& InArgs);
};
```
And the .cpp file:
```cpp
BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION
void SSlateSlice::Construct(const FArguments& InArgs)
{
}
END_SLATE_FUNCTION_BUILD_OPTIMIZATION
```
The *BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION* and *END_SLATE_FUNCTION_BUILD_OPTIMIZATION* are often
used around the *Construct* function as it can end up having a very complex expression that the
compiler may spend a lot of time trying to optimize. These macros turn optimizations off and on again.

### Passing properties
In order to have the widget draw a slice, we need to add some properties to determine its appearance.
First, we add *Brush*, *Angle* and *ArcSize* to UMG widget:
```cpp
  UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Appearance")
  FSlateBrush Brush;

  UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Appearance")
  float Angle;

  UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Appearance")
  float ArcSize;
```
We also need to add similar properties to the Slate class. They are both added as arguments inside the
*SLATE_BEGIN_ARGS*/*SLATE_END_ARGS* macros, as well as regular member variables in the class:
```cpp
  SLATE_BEGIN_ARGS(SSlateSlice)
  {}
  SLATE_ARGUMENT(FSlateBrush*, Brush)
  SLATE_ARGUMENT(float, Angle)
  SLATE_ARGUMENT(float, ArcSize)
  SLATE_END_ARGS()
  
  ...
  
protected:
  FInvalidatableBrushAttribute Brush;
  float Angle;
  float ArcSize;  
```
The *RebuildWidget* method needs to pass these properties from the UMG widget to the Slate widget:
```cpp
TSharedRef<SWidget> USlice::RebuildWidget()
{
  MySlice = SNew(SSlateSlice)
    .Brush(&Brush)
    .Angle(Angle)
    .ArcSize(ArcSize);
  return MySlice.ToSharedRef();
}
```
The Slate widget now gets these properties as arguments to the *Construct* method:
```cpp
void SSlateSlice::Construct(const FArguments& InArgs)
{
  Brush = FInvalidatableBrushAttribute(InArgs._Brush);
  Angle = InArgs._Angle;
  ArcSize = InArgs._ArcSize;
}
```
Each *SLATE_ARGUMENT* that is defined in the class header is added as a member in the *FArguments* structure,
with an underscore prefixed to its name. The *Construct* method simply copies these values to the class member
variables, allowing them to be used later when rendering the widget.

### Paint the slice
Now we can finally add the *OnPaint* method, which handles the actual rendering of the widget.

I won't go into details of all the arguments at this point - the following are the ones we refer to in the method below:
* *AllottedGeometry* gives us the position and size of the widget
* *OutDrawElements* receives draw elements used to render the widget
* *InWidgetStyle* gives us color and opacity that should be applied

[FSlateDrawElement](https://docs.unrealengine.com/en-US/API/Runtime/SlateCore/Rendering/FSlateDrawElement/index.html)
is the workhorse for rendering Slate widgets. It has methods for drawing boxes, lines, text, splines and more. The
one we will use here allows us to render an arbitrary mesh by sending in vertices (in 2D space) and indices.

We need to fill a TArray of 
[FSlateVertex](https://docs.unrealengine.com/en-US/API/Runtime/SlateCore/Rendering/FSlateVertex/index.html)
structures for the vertices, and a TArray of SlateIndex (uint32) index values defining the triangles built up
from the vertices.

To render the slice, we add one vertex at the center, then add edge vertices along the arc. The index buffer
then defines the triangles using the center and a consecutive pair of edge vertices. The color of each vertex
is calculated by modulating the brush color with color and opacity passed in via the widget style as well as the
overall color and opacity of the widget.

If the brush uses an image, we can set up UV coordinates for the vertex as well - I'm leaving that out for now.
```cpp
int32 SSlateSlice::OnPaint(const FPaintArgs& Args, const FGeometry& AllottedGeometry, const FSlateRect& MyCullingRect,
  FSlateWindowElementList& OutDrawElements, int32 LayerId, const FWidgetStyle& InWidgetStyle,
  bool bParentEnabled) const
{
  const FVector2D Pos = AllottedGeometry.GetAbsolutePosition();
  const FVector2D Size = AllottedGeometry.GetAbsoluteSize();
  const FVector2D Center = Pos + 0.5 * Size;
  const float Radius = FMath::Min(Size.X, Size.Y) * 0.5f;

  const FSlateBrush* SlateBrush = Brush.GetImage().Get();
  FLinearColor LinearColor = ColorAndOpacity.Get() * InWidgetStyle.GetColorAndOpacityTint() * SlateBrush->GetTint(InWidgetStyle);
  FColor FinalColorAndOpacity = LinearColor.ToFColor(true);

  const int NumSegments = FMath::RoundToInt(ArcSize / 10.0f);
  TArray<FSlateVertex> Vertices;
  Vertices.Reserve(NumSegments + 3);

  // Add center vertex
  Vertices.AddZeroed();
  FSlateVertex& CenterVertex = Vertices.Last();

  CenterVertex.Position = Center;
  CenterVertex.Color = FinalColorAndOpacity;

  // Add edge vertices
  for (int i = 0; i < NumSegments + 2; ++i)
  {
    const float CurrentAngle = FMath::DegreesToRadians(ArcSize * i / NumSegments + Angle);
    const FVector2D EdgeDirection(FMath::Cos(CurrentAngle), FMath::Sin(CurrentAngle));
    const FVector2D OuterEdge(Radius*EdgeDirection);

    Vertices.AddZeroed();
    FSlateVertex& OuterVert = Vertices.Last();

    OuterVert.Position = Center + OuterEdge;
    OuterVert.Color = FinalColorAndOpacity;
  }

  TArray<SlateIndex> Indices;
  for (int i = 0; i < NumSegments; ++i)
  {
    Indices.Add(0);
    Indices.Add(i);
    Indices.Add(i + 1);
  }

  const FSlateResourceHandle Handle = FSlateApplication::Get().GetRenderer()->GetResourceHandle( *SlateBrush );
  FSlateDrawElement::MakeCustomVerts(
        OutDrawElements,
        LayerId,
        Handle,
        Vertices,
        Indices,
        nullptr,
        0,
        0
    );
  return LayerId;
}
```

This now allows us to render slices in varying sizes and colors:

![SeveralSlices](/images/CustomWidget/SeveralSlices.png)

### Updating properties
For the editor to work properly we need to update the properties in the Slate widget whenever they change in
the UMG widget. We do that by overriding the *SynchronizeProperties* function:

```cpp
void USlice::SynchronizeProperties()
{
  Super::SynchronizeProperties();
  MySlice->SetBrush(&Brush);
  MySlice->SetAngle(Angle);
  MySlice->SetArcSize(ArcSize);
}
```
The Slate widget needs the following setter functions:
```cpp
void SSlateSlice::SetBrush(FSlateBrush* InBrush)
{
  Brush.SetImage(*this, InBrush);
}

void SSlateSlice::SetAngle(float InAngle)
{
  Angle = InAngle;
}

void SSlateSlice::SetArcSize(float InArcSize)
{
  ArcSize = InArcSize;
}
```
The slice should now update immediately whenever any property is changed in the editor.

### Set the category in the widget palette
Until we specify otherwise, the widget shows up as *Uncategorized* in the widget palette in the UMG Designer.
To set the category, add an override of the *GetPaletteCategory* function:
```cpp
#if WITH_EDITOR
const FText USlice::GetPaletteCategory()
{
  return LOCTEXT("CustomPaletteCategory", "My custom category!");
}
#endif
```

![PaletteCategory](/images/CustomWidget/PaletteCategory.png)

### What next?
There is still work to be done on this widget before it is on par with regular UMG widgets in Unreal. For example,
the properties do not support animations, nor property bindings. I've also not discussed mouse events - all
widgets in UMG (and Slate) are rectangular, so some trickery is needed for proper mouse handling on irregularly
shaped widgets.

One potential reason for implementing custom widgets is for optimization - for example, it may be faster to
implement a single widget that combines rendering of several elements - I'll explore that in near future.

Widgets that act as parents for other widgets are another story as well, so I have several topics for future blog
posts lined up.

### Get the code
The for this project lives on GitHub:

[https://github.com/snorristurluson/CustomWidget](https://github.com/snorristurluson/CustomWidget)
