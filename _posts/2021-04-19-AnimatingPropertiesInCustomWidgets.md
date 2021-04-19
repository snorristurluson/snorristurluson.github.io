---
title: Animating Properties In Custom Widgets
tags: ue4 c++ widget
---
In an [earlier blog](/CustomSlateWidgets) I showed how to create a custom widget for drawing a slice. It has
properties for the start angle and arc size, but unlike most numeric properties on widgets they can't be animated
as I left the widget last time. This is easy to fix, though!

If we look at the position properties of the slice (under the Canvas Panel Slot) we see that there is an icon
in front of the input field:

![Position](/images/AnimatingPropertiesInCustomWidgets/Position.png)

This icon is missing from the Angle and ArcSize properties:

![Before](/images/AnimatingPropertiesInCustomWidgets/Before.png)

Clicking this icon adds a keyframe to an animation, so the absence of this icon means the property cannot be
animated. All that is needed for the icon to show up is add a setter for the property:

```cpp
  UFUNCTION(BlueprintCallable, Category="Appearance")
  void SetAngle(float InAngle);

  UFUNCTION(BlueprintCallable, Category="Appearance")
  void SetArcSize(float InArcSize);
```

The setter should also forward the new value to the underlying Slate widget:

```cpp
void USlice::SetAngle(float InAngle)
{
  Angle = InAngle;
  if (MySlice)
  {
    MySlice->SetAngle(Angle);
  }
}

void USlice::SetArcSize(float InArcSize)
{
  ArcSize = InArcSize;
  if (MySlice)
  {
    MySlice->SetArcSize(ArcSize);
  }
}
```

After this change the keyframe animation shows up in the editor:

![After](/images/AnimatingPropertiesInCustomWidgets/After.png)

Now I can set up an animation that animates the angle and/or the arc size of the slice:

![Animation](/images/AnimatingPropertiesInCustomWidgets/Animation.png)

![Result](/images/AnimatingPropertiesInCustomWidgets/Result.png)

## The code
The code for this project lives on GitHub:
[https://github.com/snorristurluson/CustomWidget](https://github.com/snorristurluson/CustomWidget)