# 🕹️ UE5 Mechanics Showcase: Interaction & Inspection System

This repository showcases a modular, high-performance **First-Person Interaction & 3D Object Inspection System** developed in **Unreal Engine 5**. 

The system follows clean architecture principles, separating heavy math, raycasting, and state management into **C++**, while leveraging **Blueprints** for rapid UI iteration, sound effects, and design-centric puzzle logic.

---

## 🛠️ 1. C++ Architecture & Interfaces

To achieve strict decoupling and eliminate expensive casting between actors, communication is driven entirely by **C++ Interfaces**.

### Architecture Framework Overview
Below is the core class and interface setup used throughout the project:

![Project C++ Architecture](Снимок%20экрана%202026-06-16%20162409.png)

### Player-to-Object Raycast (`PlayerCharacter.cpp`)
Every frame, the character performs a line trace extending `InteractDistance` (250 units) to dynamically detect interactable elements in the environment and cache the active reference:

```cpp
void APlayerCharacter::CheckForInteractables()
{
    FVector Start;
    FRotator Rotation;
    GetWorld()->GetFirstPlayerController()->GetPlayerViewPoint(Start, Rotation);
    FVector End = Start + (Rotation.Vector() * InteractDistance);

    FHitResult Hit;
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(this);

    if (GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_Visibility, Params) && Hit.GetActor())
    {
        ABaseInteractable* FoundInteractable = Cast<ABaseInteractable>(Hit.GetActor());
        if (FoundInteractable) 
        { 
            CurrentInteractable = FoundInteractable; 
        }
    }
}
```

### The Inspection Interface Contract (`InspectableInterface.h`)
Any object in the game world can support 3D inspection seamlessly simply by inheriting this interface, removing hard dependencies on the player class:

```cpp
class SCARYROOM_API IInspectableInterface
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Inspect")
    void OnInspectStart();

    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Inspect")
    void OnInspectEnd();

    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "Inspect")
    UStaticMeshComponent* GetInspectMesh();
};
```

---

## 👁️ 2. Dynamic 3D Object Inspection System

The core gameplay loop allows players to pick up items and rotate them freely in front of the camera. This behavior is encapsulated inside a dedicated actor component: `UInspectComponent`.

### State Guarding & Input Redirection (`InspectComponent.cpp`)
To ensure high visual fidelity and robust state safety, the component stops camera shakes, freezes skeletal mesh animations (fixing head-bob jitter bugs), and safely switches the input mode:

```cpp
void UInspectComponent::StartInspect(AActor* ItemToInspect)
{
    GetWorld()->GetFirstPlayerController()->PlayerCameraManager->StopAllCameraShakes();

    if (!ItemToInspect || bIsInspecting || bIsReturning) return;

    bIsInspecting = true;
    InspectedActor = ItemToInspect;
    OriginalTransform = InspectedActor->GetActorTransform();

    // Freeze skeletal mesh animations to prevent camera head-bob artifacts
    if (ACharacter* PlayerChar = Cast<ACharacter>(GetOwner()))
    {
        PlayerChar->GetMesh()->bPauseAnims = true;
    }

    if (APlayerController* PC = GetWorld()->GetFirstPlayerController())
    {
        PC->bShowMouseCursor = true;
        PC->SetInputMode(FInputModeGameAndUI());
        PC->SetIgnoreMoveInput(true);
    }

    if (InspectedActor->GetClass()->ImplementsInterface(UInspectableInterface::StaticClass()))
    {
        IInspectableInterface::Execute_OnInspectStart(InspectedActor);
    }
}
```

### Frame-Rate Independent Interpolation (`InspectComponent.cpp`)
Inside `TickComponent`, smooth transitions for picking up and returning objects back to their original transform positions are calculated using `VInterpTo` and `RInterpTo`:

```cpp
void UInspectComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

    if (!InspectedActor) return;

    if (bIsInspecting && CachedAnchor)
    {
        FVector TargetLoc = CachedAnchor->GetComponentLocation();
        FRotator TargetRot = CachedAnchor->GetComponentRotation();
        InspectedActor->SetActorLocation(FMath::VInterpTo(InspectedActor->GetActorLocation(), TargetLoc, DeltaTime, 13.f));
        InspectedActor->SetActorRotation(FMath::RInterpTo(InspectedActor->GetActorRotation(), TargetRot, DeltaTime, 13.f));
    }
    else if (bIsReturning)
    {
        FVector CurLoc = InspectedActor->GetActorLocation();
        FVector TargetLoc = OriginalTransform.GetLocation();

        InspectedActor->SetActorLocation(FMath::VInterpTo(CurLoc, TargetLoc, DeltaTime, 13.f));
        InspectedActor->SetActorRotation(FMath::RInterpTo(InspectedActor->GetActorRotation(), OriginalTransform.GetRotation().Rotator(), DeltaTime, 13.f));

        if (FVector::Dist(CurLoc, TargetLoc) < 2.0f)
        {
            InspectedActor->SetActorLocationAndRotation(OriginalTransform.GetLocation(), OriginalTransform.GetRotation().Rotator());
            bIsReturning = false;
            InspectedActor = nullptr;
        }
    }
}
```

---

## 🎨 3. Blueprint Implementation & Gameplay Layer

While C++ establishes the rigid technical foundation, **Blueprints** are utilized as a flexible visual scripting layer for designer-facing mechanics, widgets, and level script integration.

### 🔦 Flashlight Toggle Logic
Manages input action mapping via the Enhanced Input System to smoothly toggle spotlight visibility and trigger context-aware audio effects:

![Flashlight Blueprint Graph](Снимок%20экрана%202026-06-16%20162438.png)### 🔑 Key-Lock Narrative System (Inventory Validation)

Displays real-time screen prompts when a locked actor is approached, handles item validation against the character's inventory arrays, and manages actor lifecycles upon successful entry:

![Door Interaction Blueprint Graph](Снимок%20экрана%202026-06-16%20162511.png)---

## 🚀 Key Takeaways & Optimization Highlights
* **Zero Casting Overhead:** Code structure is entirely decoupled. The player never casts directly to interactable actors, routing all actions through interface contracts.
* **Camera Jitter Resolution:** Fixed standard view bugs during inspection states by programmatically forcing `bPauseAnims = true` on the player character.
* **Input Spam Protection:** Active state flags (`bIsReturning`, `bIsInspecting`) discard premature inputs, preserving physical consistency and preventing collision glitches.

* ---

## 🛠️ DevLog & Technical Bugfixes (19 June 2026)

### Camera Collision & Clipping Fix
* **The Issue:** When standing close to geometry (walls/doors) and looking sharply downward, the first-person camera would glitch, clip through meshes, and warp outside the level bounds.
* **The Culprit:** The `SpringArmComponent` had `Use Pawn Control Rotation` enabled. In tight spaces, this inherited rotation heavily conflicted with the spring arm's built-in collision solver.
* **The Solution:** Disabled the checkbox, allowing the camera to correctly respect line-traces and slide smoothly against wall geometry without breaking immersion.

| Before (Camera Clipping) | After (Smooth Collision Fixed) |
|---|---|
| ![Clipping Bug](Снимок%20экрана%202026-06-19%20141128.jpg) | ![Fixed Camera](Снимок%20экрана%202026-06-19%20164918.jpg) |
