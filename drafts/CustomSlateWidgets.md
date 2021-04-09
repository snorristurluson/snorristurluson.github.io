## Custom widgets in Unreal

There are two classes involved in UMG widgets:
* A UWidget-derived class, that you interact with in the editor
* An SCompoundWidget class, that handles the low-level functionality and rendering

Let's take a look at a simple widget that renders a slice, with a given start angle and arc size.
We can control the color and opacity, and optionally use an image for rendering it.

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

```cpp
BEGIN_SLATE_FUNCTION_BUILD_OPTIMIZATION
void SSlateSlice::Construct(const FArguments& InArgs)
{
}
END_SLATE_FUNCTION_BUILD_OPTIMIZATION
```

Add Brush, Angle and ArcSize to UMG widget
```cpp
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Appearance")
	FSlateBrush Brush;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Appearance")
	float Angle;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Appearance")
	float ArcSize;
```

Paint the slice
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
Set the category in the widget palette
```cpp
#if WITH_EDITOR
const FText USlice::GetPaletteCategory()
{
	return LOCTEXT("CustomPaletteCategory", "My custom category!");
}
#endif
```