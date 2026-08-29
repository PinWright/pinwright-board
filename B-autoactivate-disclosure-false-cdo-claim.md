---
id: B-autoactivate-disclosure-false-cdo-claim
title: "bAutoActivate disclosure asserts a true CDO default that most component classes do not have"
status: OPEN
severity: Medium
category: bug
tags: [bAutoActivate, disclosure, warnings, cdo, archetype, false-statement, actor.set_component_properties, actor.add_component, property.set]
encounters: 1
lastSeen: 2026-08-29
---

# The disclosure states a persistence fact that is false for most component classes

`Utils/AutoActivateDisclosure.h` (`PinWright::MakeAutoActivateDisabledDisclosure`) emits one shared
warning from `actor.set_component_properties`, `actor.add_component` and `property.set` when a write
leaves a component's `bAutoActivate` false. The shipped text packs three claims into one sentence:

    bAutoActivate is now false on '<name>': the component will not start on its own, in this
    session or on any later load. bAutoActivate defaults to true, so the override is saved into
    the level and outlives the session - nothing renders from this component until it is restored
    or something activates it explicitly.

- **A — "will not start on its own, in this session or on any later load."** True unconditionally,
  given the flag reads false after the write.
- **B — "bAutoActivate defaults to true, so the override is saved into the level."** False for most
  component classes.
- **C — "nothing renders from this component until it is restored."** False for every primitive.

Landed hours ago under `B-niagara-validate-green-while-component-inactive` `#4`; `#5` (a fixture
correction) found the false premise and deliberately left the behaviour change to its own ticket.
This is that ticket.

## Verified in engine source (UE 5.8, `C:\UE_5.8\Engine\Source`)

Base chain — nothing defaults it true:

- `UActorComponent::UActorComponent` (`Runtime/Engine/Private/Components/ActorComponent.cpp:536`)
  never assigns `bAutoActivate`. The bitfield (`Classes/Components/ActorComponent.h:317`) is left at
  the zero-initialised **false**. The only other writes in that file are a save/restore pair
  (`:880`/`:884`/`:894`) and `SetAutoActivate` (`:2860`).
- `USceneComponent::USceneComponent` sets `bAutoActivate = false` **outright**
  (`Components/SceneComponent.cpp:128`).
- `UPrimitiveComponent`, `UMeshComponent`, `UStaticMeshComponent` never touch it — so the whole
  static-mesh / light / shape / spline / ISM / decal / text / billboard family starts **false**.

Classes that opt in to `true`, from an exhaustive `bAutoActivate =` sweep of
`Engine/Source/**/*.{cpp,h}` and `Engine/Plugins/**/*.{cpp,h}` (constructor assignments only;
`Engine/Source/Editor` and `Engine/Source/Developer` have none):

| Class | Site |
|---|---|
| `UCameraComponent` | `Runtime/Engine/Private/Camera/CameraComponent.cpp:95` |
| `UCineCameraComponent` | `Runtime/CinematicCamera/.../CineCameraComponent.cpp:43` (re-asserts) |
| `UAudioComponent` | `Runtime/Engine/Private/Components/AudioComponent.cpp:124` |
| `USkinnedMeshComponent` | `Runtime/Engine/Private/Components/SkinnedMeshComponent.cpp:470` |
| `USkeletalMeshComponent` | `Runtime/Engine/Private/Components/SkeletalMeshComponent.cpp:438` (re-asserts) |
| `UParticleSystemComponent` | `Runtime/Engine/Private/Particles/ParticleSystemComponent.cpp:449` |
| `UMovementComponent` | `Runtime/Engine/Private/Components/MovementComponent.cpp:46` (whole movement family) |
| `USpringArmComponent` | `Runtime/Engine/Private/GameFramework/SpringArmComponent.cpp:24` |
| `USceneCaptureComponent2D` | `Runtime/Engine/Private/Components/SceneCaptureComponent.cpp:649` |
| `USceneCaptureComponentCube` | `Runtime/Engine/Private/Components/SceneCaptureComponent.cpp:1305` |
| `UForceFeedbackComponent` | `Runtime/Engine/Private/Components/ForceFeedbackComponent.cpp:164` |
| `ULineBatchComponent` | `Runtime/Engine/Private/Components/LineBatchComponent.cpp:143` |
| `UWindDirectionalSourceComponent` | `Runtime/Engine/Private/WindDirectionalSource.cpp:156` |
| `UPhysicsHandleComponent` | `Runtime/Engine/Private/PhysicsEngine/PhysicsHandleComponent.cpp:22` |
| `UPhysicsSpringComponent` | `Runtime/Engine/Private/PhysicsEngine/PhysicsSpring.cpp:13` |
| `URadialForceComponent` | `Runtime/Engine/Private/PhysicsEngine/RadialForceComponent.cpp:28` |
| `UAsyncPhysicsInputComponent` | `Runtime/Engine/Private/PhysicsEngine/AsyncPhysicsInputComponent.cpp:88` |
| `UNetworkPhysicsComponent` | `Runtime/Engine/Private/PhysicsEngine/NetworkPhysicsComponent.cpp:816` |
| `UNetworkPhysicsSettingsComponent` | `Runtime/Engine/Private/PhysicsEngine/NetworkPhysicsSettingsComponent.cpp:39` |
| `UChaosDestructionListener` | `Runtime/Experimental/GeometryCollectionEngine/.../ChaosBlueprint.cpp:19` |
| `UBehaviorTreeComponent` | `Runtime/AIModule/.../BehaviorTreeComponent.cpp:69` |
| `UMotionControllerComponent` | `Runtime/HeadMountedDisplay/.../MotionControllerComponent.cpp:67` |
| `UMediaSoundComponent` | `Runtime/MediaAssets/.../MediaSoundComponent.cpp:204` |
| `UMediaComponent` | `Runtime/MediaAssets/.../MediaComponent.cpp:14` |
| `UWidgetInteractionComponent` | `Runtime/UMG/.../WidgetInteractionComponent.cpp:38` |
| `UMockDataMeshTrackerComponent` | `Runtime/MRMesh/.../MockDataMeshTrackerComponent.cpp:260` |
| `UNavigationInvokerComponent` | `Runtime/NavigationSystem/.../NavigationInvokerComponent.cpp:15` |
| `UNiagaraComponent` | `Plugins/FX/Niagara/Source/Niagara/Private/NiagaraComponent.cpp:685` |
| `UAudioGameplayVolumeComponent` / `UAudioGameplayVolumeComponentBase` | `Plugins/AudioGameplayVolume/.../AudioGameplayVolumeComponent.cpp:24`, `:162` |
| `USubmixSendVolumeComponent` / `USubmixOverrideVolumeComponent` / `UReverbVolumeComponent` / `UFilterVolumeComponent` / `UAttenuationVolumeComponent` | same plugin, `:48` / `:83` / `:62` / `:34` / `:34` |
| `UGameplayCameraComponentBase` / `UGameplayControlRotationComponent` / `UControllerGameplayCameraEvaluationComponent` | `Plugins/Cameras/GameplayCameras/.../:46` / `:26` / `:17` |
| `UControlRigComponent` | `Plugins/Animation/ControlRig/.../ControlRigComponent.cpp:70` |
| `UMLDeformerComponent` | `Plugins/Animation/MLDeformer/.../MLDeformerComponent.cpp:29` |
| `UCEClonerComponent` | `Plugins/VirtualProduction/ClonerEffector/.../CEClonerComponent.cpp:76` |
| `UXRDeviceVisualizationComponent` | `Plugins/Runtime/XRBase/.../XRDeviceVisualizationComponent.cpp:27` |
| `UXRCreativePointerComponent` / `UXRCreativeITFComponent` | `Plugins/Experimental/XRCreativeFramework/.../:15` / `:433` |
| `UComposurePostProcessPass` / `UComposureCompositingTargetComponent` | `Plugins/Compositing/Composure/.../:37` / `:137` |
| `UCompositeViewProjectionComponent` | `Plugins/Compositing/Composite/.../CompositeViewProjectionComponent.cpp:150` |

Not a clean `true`, and worth knowing:

- `UPathFollowingComponent` — `bAutoActivate = !bTickComponentOnlyWhenMoving`
  (`Runtime/AIModule/.../PathFollowingComponent.cpp:134`), so it is config-dependent.
- `UHoldoutCompositeComponent` — `true` only inside a cvar branch
  (`Plugins/Compositing/CompositeCore/.../HoldoutCompositeComponent.cpp:79`).
- `UScriptPluginComponent` / `UScriptContextComponent` set `true` then clear to `false` later in the
  same constructor (`Plugins/ScriptPlugin/.../:13`+`:30`, `:13`+`:32`).
- Engine classes that explicitly default it **false**: `USynthComponent`
  (`Runtime/AudioMixer/.../SynthComponent.cpp:91`), `UPawnSensingComponent`
  (`Runtime/AIModule/.../PawnSensingComponent.cpp:37`).
- `UTimelineComponent` instances are set true per-instance at
  `Runtime/Engine/Private/BlueprintGeneratedClass.cpp:1833`, not in a constructor.

Claim C, separately verified: `USceneComponent::ShouldRender()`
(`Components/SceneComponent.cpp:3485-3518`) reads visibility, `bHiddenEdTemporary`, owner-hidden and
the parent `UChildActorComponent` — it never consults `IsActive()`. `bAutoActivate` is read in
exactly one place on the register path, `UActorComponent::OnRegister` → `Activate(true)`
(`ActorComponent.cpp:1606`). Activation gates ticking and each component's own update path — which
is why Niagara, Cascade, audio and the movement components care — not `UPrimitiveComponent`
scene-proxy creation. A `UStaticMeshComponent` with `bAutoActivate:false` renders normally.

So on the modal target of these three verbs — a scene/primitive component — writing `false`
**matches the archetype, is therefore never serialised as an override**, and the mesh keeps
rendering, while the response asserts that an override was saved into the level and that nothing
renders. The disclosure generalises a Niagara-true fact to every component.

## Why this is a defect and not a wording nit

The disclosure exists to satisfy `B-niagara-validate-green-while-component-inactive` suggested fix 4
— a write that outlives the session must say so. Its entire value is that the caller can trust it.
Claim B sends a caller after a `.umap` delta that does not exist: that ticket's own cheap
discriminator, `grep -ac bAutoActivate Content/Maps/<Level>.umap`, returns **0** in exactly this
case, so following the disclosure's own advice produces evidence that contradicts it. It also
teaches the generalisation forward — an agent that reads "bAutoActivate defaults to true" will carry
that into unrelated decisions about components that never did.

## Archetype, not the class CDO

`UObject::SerializeScriptProperties` (`Runtime/CoreUObject/Private/UObject/Obj.cpp:2031-2057`) sets
`DiffObject = GetArchetypeFromLoader(this)`, falling back to `GetArchetype()`, and delta-serialises
tagged properties against it. The **archetype**, not `GetClass()->GetDefaultObject()`, is what
decides whether a value reaches the package — so it is the correct comparison for claim B.

`FindArchetypeFromRequiredInfoImpl` (`UObject/UObjectArchetype.cpp:100-190`) resolves the outer's
archetype and looks for a same-named subobject inside it. For a component that means:

- native `CreateDefaultSubobject` component → the owning actor CDO's subobject of that name;
- Blueprint SCS component → the BP's `ComponentTemplate`, or the child BP's
  `UInheritableComponentHandler` override (`RF_InheritableComponentTemplate` branch);
- an instance-added component with no template → falls through to `Class->GetDefaultObject()`.

That last bullet is why no separate CDO branch is needed: `GetArchetype()` already degrades to the
CDO where there is no template. The Blueprint subtlety `#5` flagged is real and cuts **both** ways:

- a BP whose SCS template already carries `bAutoActivate:false` — writing `false` on a placed
  instance produces no delta, yet a class-CDO comparison on a true-defaulting class (a
  `UNiagaraComponent` under a BP that already disabled it) would claim an override that is not
  there;
- a BP whose template sets `bAutoActivate:true` on a false-defaulting class (a
  `UStaticMeshComponent` with the box ticked) — writing `false` **is** a genuine level override,
  which a class-CDO comparison would miss entirely.

The archetype gets both right; the class CDO gets both wrong. It is also the baseline the plugin
already commits to: `Utils/ActorDescribeBuilder.cpp:105` diffs each component against
`Component->GetArchetype()`, and that is the code that produced the
`bAutoActivate: false, is_overridden_locally: true` reading in the parent ticket's evidence. Using
any other baseline lets two verbs disagree about the same component.

One caveat to state rather than hide: because the archive may substitute `GetArchetypeFromLoader` at
save time, no handler-time comparison is an exact prediction of the bytes on disk. The archetype is
the closest available answer and the one the rest of the plugin already uses.

## Which fix, and why not the other two

**Split the claims. Do not gate the whole warning, and do not just reword it.**

Gating everything on the CDO/archetype drops claim A exactly where it still matters.
`USynthComponent` and `UPawnSensingComponent` default `false` in engine code; an instance-added
`UStaticMeshComponent` defaults `false`; a BP template can carry `false`. In every one of those the
component is inert on this and every later load, the write matched the archetype, and a gated
disclosure would say **nothing** — reintroducing the silence this feature was built to end. Claim A
does not depend on a delta existing.

Rewording alone cannot work either: the persistence claim is genuinely class-dependent, so any fixed
string is either false for some class or weakened until it says nothing.

**Fix:**

1. `MakeAutoActivateDisabledDisclosure` keeps emitting the unconditional half whenever the flag reads
   `false` after the write — *the component will not start on its own, in this session or on any
   later load*.
2. Append the override half only when it is **measured**:
   `const UActorComponent* Archetype = Cast<UActorComponent>(Component->GetArchetype());` then
   `Archetype && Archetype->bAutoActivate != 0` → *this is a local override, serialised into the
   level, and it outlives the session*. A null archetype says nothing about serialisation rather
   than guessing.
3. Drop *"nothing renders from this component"*, or narrow it to *nothing this component drives will
   run* — rendering is governed by visibility, not activation, and the clause is false for the whole
   primitive family.
4. Keep the discipline that is already correct: the flag is read back off the component, never taken
   from the request, so a declined write is never disclosed as one that landed and restoring the flag
   stays silent.
5. Reuse `Component->GetArchetype()` specifically, so this disclosure and `actor.describe`'s
   `is_overridden_locally` cannot drift apart.

Test shape: the existing probe in `Tests/Actor/TestComponentAutoActivateDisclosure.cpp` (an
`ACameraActor` / `UCameraComponent`, archetype `true`) already covers the override case. The missing
calibration is the opposite probe — a component whose archetype already has the flag off
(`AStaticMeshActor`'s `UStaticMeshComponent`, or `actor.add_component` with
`componentType: SceneComponent`) — which must still disclose "will not start" and must **not** claim
a level override. That is the fixture `#5` found was passing while measuring nothing.

**Workaround:** trust `actor.describe`'s per-component `is_overridden_locally` on `bAutoActivate` —
that field is archetype-measured and correct — and ignore the disclosure's serialisation clause.

## History
- `#1-shipped-text-generalises-a-niagara-true-default` `OPEN` reporter — Read out of the shipped
  string in `Utils/AutoActivateDisclosure.h` and verified against UE 5.8 source, not inferred.
  `UActorComponent`'s constructor (`ActorComponent.cpp:536`) never assigns the bitfield;
  `USceneComponent` sets it **false** (`SceneComponent.cpp:128`); `UPrimitiveComponent` /
  `UMeshComponent` / `UStaticMeshComponent` never touch it. An exhaustive `bAutoActivate =` sweep of
  `Engine/Source` and `Engine/Plugins` produced the opt-in-true list above (~40 classes, none in
  `Engine/Source/Editor` or `Developer`), plus two engine classes that default it false and three
  that set it conditionally. Third claim also checked: `USceneComponent::ShouldRender()`
  (`SceneComponent.cpp:3485`) never reads `IsActive()`, and `bAutoActivate` is consumed only at
  `ActorComponent.cpp:1606` (`OnRegister` → `Activate`), so a static mesh with the flag off renders
  normally. Serialisation baseline confirmed at `Obj.cpp:2031-2057` (delta vs `GetArchetype()`) and
  the archetype resolution at `UObjectArchetype.cpp:100-190`. Severity Medium: the impact class is
  High — wrong data on a normal path, from the one feature whose purpose is honest disclosure — but
  the reach modifier drops it one band, since the disclosure only fires when a request explicitly
  names `bAutoActivate` and leaves it false, a rare path. Not Low: `Low` is docs / discoverability /
  cosmetic, and this is a false statement of fact in a live response. No plugin source touched, no
  compile and no test run (a suite was live).
