# ProjectW Portfolio - Metehan Goksel Kurtulan 

ProjectW is not a public repository, instead I have written down a breakdown of the systems i have created in this document. I can provide further information if requested.

This document can be better viewed at [Notion](https://helpful-bite-d89.notion.site/ProjectW-Portfolio-Metehan-Goksel-Kurtulan-20a08570f25e80f2b5aacd6a3ed05e72).

## Summary

ProjectW is a round-based multiplayer last-man-standing spell arena inspired by *Warcraft 3’s Warlock.* The game is built from the ground up using Unreal Engine 5 and designed for up to 12 players. The game revolves around the idea of defeating enemy players in an arena that gets smaller over time by either damaging or pushing them off the arena.

Developed entirely solo while learning UE5, this project was built using C++ and Blueprint in combination with Unreal’s Gameplay Ability System (GAS). Every system — from multiplayer logic to UI and combat — was self-researched, tested, and implemented through iterative learning and development.

While the visuals may be a bit rough, this was a deliberate tradeoff: all assets were either free, dirt-cheap, or AI-generated, allowing full focus on gameplay systems, performance, and architecture. The result is a functional, scalable prototype that proves strong mechanics don't need a triple-A budget.

The goal of this document is to showcase key gameplay systems, architectural decisions, and engineering practices behind the project. Due to the a few paid assets, i cannot legally make it public, but I can share any further details about the project upon request. *Paid assets are purely visual/audio and do not affect the engineering work.*

---

## Showcase

- 🎥 [Menu and Lobby](https://youtu.be/Z8bkf3qagkI)  
- 🎮 [Gameplay Showcase](https://youtu.be/b00fYstrt7s)  

---

## Core Technologies

- Unreal Engine 5:  Built with a clear division between C++ and Blueprint. Core logic is written in C++ whereas gameplay behaviors of abilities and UI are done in Blueprint for rapid iteration and testing.
- Gameplay Ability System (GAS): Attributes, abilities, effects, tags, input management anything related to these systems fully integrated and uses the full power of GAS.
- UMG & Niagara: UI and visual effects for spells, feedback and player state
- Steam Sessions: Lobby, hosting, invites etc. all through Advanced Sessions Plugin.

## Systems Built

1. [Round-Based Game Mode](#1-round-based-game-mode):
Modular game phase system built with a custom `GameMode`/`GameState` architecture that is easily extensible for new game modes.
2. [Tag-Based Gameplay Systems](#2-tag-based-gameplay-system):
Spells and items are dynamically bought, sold, upgraded, and assigned through `GameplayTag`s. Input bindings, Niagara FX, and UI updates are all handled via tag-driven event systems.
3. [Inventory System](#3-inventory-system): 
Fully replicated inventory system that is designed to be an equivalent to `AbilitySystemComponent` for items. Handles passive and on-use effects, item binding via input tags, fast array replication.
4. [Extended Ability System Component](#4-extended-ability-system-component):
Custom subclass of `AbilitySystemComponent` that adds support for tag-based ability lookup, dynamic input rebinding, shop and UI integration.
5. [Modular Shop System](#5-modular-shop-system):
Fully replicated, server-authoritative shop using gold validation and inventory checks. Includes tooltips and resell logic with adjustable multipliers.
6. [Unified Damage Model](#6-unified-damage-model):
Central `ExecCalc_Damage` handles multiple damage types in a single effect using `GameplayTag`s. Supports penetration/resistance scaling and a modular `AfterEffects` system for debuffs and buffs.
7. [Projectile Framework](#7-projectile-framework):
Base projectile class that supports knockback, reflection, piercing, destruction, and more — all configurable via data or interfaces for flexible spell behaviors.
8. [Structured Logging System](#8-structured-logging-system):
Custom logging macros and helpers that support JSON-formatted logs and runtime diagnostics across various systems.
9. [Local & Global Timer Management (WTM)](#9-local--global-timer-management-wtm):
Global static timer manager that handles gameplay-safe asynchronous delays using string-based keys for scoped access, cancelation, and debugging.
10. [Extended Sessions](#10-extended-sessions): 
Using Advanced Session System, this new extended version allows easy setups for this game. whether if it’s adding new modes or new filter options for the lobbies.

### Additional Systems

- Scoreboard: Tracks damage, kills, overall round performance.
- Tagged Effected Component: Listens for `GameplayTag` changes to dynamically spawns and removes VFX on the character such as fire effect for burn.
- Input Binding & Spellbar UI: Allows players to rebind inputs of spells via drag-and-drop interface.
- Click-to-Move Component: Custom movement logic replacing default click-to-move behavior.
- Arena Collapse System: Arena tiles collapse dynamically over time during combat phase, shrinking the play area.
- Lobby Settings: Host can define various settings such as starting gold, teams, number of rounds to play, synched to all players who can check them out before the game starts.

---

## Detailed Systems Breakdowns

### 1. Round-Based Game Mode
<details>
## Overview

The game uses a modular round system split into clear phases:

- Loading: Wait for all players to connect
- Warmup: Safe period to test spells/items without damage
- Combat: Players can damage/kill each other
- Intermission: Shop access, round reset

Phase control is driven entirely by a custom `AWBaseGameMode` and replicated via `AMyGameState` using the `EGamePhase` enum for consistent state tracking across clients.

---

## Design Goals

- Easy to extend with new game phases
- Predictable phase transitions with proper authority handling
- Ability to cleanly reset or interrupt phases (e.g. for player disconnection)

---

## Architecture

- `AWBaseGameMode::SetGamePhase(EGamePhase)` manages all transitions
- `AMyGameState` replicates the current phase to clients
- Delegates (`OnGamePhaseChanged`) notify other systems like UI, scoreboard, VFX
- `SET_TIMER()` are used for timed transitions

---

## Example: Phase Transition

```cpp
void AWBaseGameMode::SetGamePhase(EGamePhase NewPhase)
{
    switch (CurrentGamePhase)
    {
        case EGamePhase::Combat:
            HandleEndIntermissiont();
            break;
        // other phase cleanup logic
    }

    CurrentGamePhase = NewPhase;
    PhaseStartTime = GetWorld()->GetTimeSeconds();

    switch (NewPhase)
    {
        case EGamePhase::Warmup:
            HandleCombat();
            break;
        // other phase entry logic
    }

    GetMyGameState()->MulticastCurrentGamePhase(NewPhase);
}
```
</details>

### 2. Tag-Based Gameplay Systems
<details>
## Overview

GameplayTags are the backbone of several systems, enabling clean event dispatching and fully data-driven behavior across abilities, input, effects, UI, and animation.

The system allows spells, inputs, visual effects, and gameplay responses to be defined or triggered purely through tag queries without hard-coded conditions.

---

## Key Systems Using Tags

- Ability Input Mapping
    
    Each spell bar slot is associated with a specific `GameplayTag`, passed into the assigned ability and used for tag-based activation via the `AbilitySystemComponent`.
    
- Gameplay Effects & Conditions
    
    All gameplay logic — from buffs and debuffs to conditions and cooldowns — is driven by tags present on the character, spell, or effects.
    
- Event Broadcasting
    
    Tag-driven delegates are used for dynamic reactions across systems. Example:
    
    When a `Status.Debuff.Burn` tag is applied, the `TaggedEffectsComponent` listens and spawns a fire visual effect on the burning character. On tag removal, it cleans up.
    
- UI & FX Logic
    
    UI elements (status icons, spell cooldowns, spells and items in shops etc.) are conditionally shown based on tag queries. Visual feedback such as cooldowns are also handled via tags.
    
- Animation State Control
    
    Tags like `Status.Stun` inform animation blueprints to switch locomotion states for visual feedback.
    
- Montage Events
    
    Tags such as `Ability.Base.Casting` and `Event.Cast.Complete` drive the interaction between animation and ability logic. This allows character animation to notify systems of key moments like cast completion without hard references.
    

---

## Design Goals

- Avoid logic duplication by centralizing logic in tags
- Allow designers to define spells, effects, and UI behavior using tags only
- Support flexible tag listeners for FX, UI, and state machines

---

## Architecture

- `WTags`: Centralized static class with constants for all common tags
- `TaggedEffectsComponent`: Subscribes to tag change delegates and spawns/removes Niagara FX
- `ASC->AnyGameplayTagChanged`: A server-authoritative delegate that triggers on both server and client, enabling multiple systems to respond to tag changes.

## Example: Tag Declaration

```cpp
void WTags::InitializeNativeGameplayTags()
{
	//Attributes
	GameplayTags.Attribute = UGameplayTagsManager::Get().AddNativeGameplayTag(FName("Attribute"), FString("Attribute"));
	GameplayTags.Attribute_Main = UGameplayTagsManager::Get().AddNativeGameplayTag(FName("Attribute.Main"), FString("Main Attribute"));
	GameplayTags.Attribute_Main_MaxHealth = UGameplayTagsManager::Get().AddNativeGameplayTag(FName("Attribute.Main.MaxHealth"), FString("Max Health"));
	...
}
```

## Example: Input Tag Binding

```cpp
MyInputComponent->BindAbilityActions(InputConfig, this, &ThisClass::AbilityInputTagPressed, &ThisClass::AbilityInputTagReleased, &ThisClass::AbilityInputTagHeld);

...

void AMyPlayerController::InputTagPressed(FGameplayTag InputTag)
{
	if (GetASC() && GetASC()->HasMatchingGameplayTag(WTags::Get().Player_Input_Block_InputPressed)) return;

	if (InputTag.MatchesTagExact(WTags::Get().Input_RMB))
	{
		bHasTarget = ActorAtCursor ? true : false;
		ClickToMoveComponent->InputPressed();
	}
	if (GetASC()) GetASC()->AbilityInputTagPressed(InputTag);
	if (GetInventory()) GetInventory()->ItemInputTagPressed(InputTag);
}
```

## Example: VFX Binding to Tag

```cpp
void UTaggedEffectComponent::OnTagApplied(FGameplayTag& Tag, int32 Count)
{
	const WTags Tags = WTags::Get();

	if (!Tag.MatchesTag(Tags.Status_Buff) && !Tag.MatchesTag(Tags.Status_Debuff) && !Tag.MatchesTag(Tags.Ability_Type_Passive)) return;

	const bool bTryActive = Count > 0;
	const bool bEffectActive = ActiveEffects.Contains(Tag);

	if (bTryActive && bEffectActive) return;

	if (bEffectActive && !bTryActive)
	{
		UNiagaraComponent* Effect = ActiveEffects[Tag];
		if (Effect)
		{
			Effect->SetAutoDestroy(true);
			Effect->Deactivate();
			ActiveEffects.Remove(Tag);
		}
	}

	else if (!bEffectActive && bTryActive)
	{
		const UTaggedNiagaraInfo* Data = UGameData::GetTaggedEffectsNiagaraData(this);
		const FMyTaggedNiagaraInfo Info = Data->GetNiagaraSystemForTag(Tag);
		if (!Info.Niagara)
			return;
		
		const auto NiagaraSystem = UNiagaraFunctionLibrary::SpawnSystemAttached(Info.Niagara, OwnerMeshComp, USocketFunctions::GetSocketName(Info.Socket), FVector::ZeroVector, FRotator::ZeroRotator, EAttachLocation::SnapToTarget, true);
		
		if (NiagaraSystem && Info.bUseAbsoluteRotation)
		{
			NiagaraSystem->SetUsingAbsoluteRotation(true);
			NiagaraSystem->SetWorldRotation(FRotator::ZeroRotator);
			ActiveEffects.Add(Tag, NiagaraSystem);
		}
	}
}
```

![image.png](attachment:8cf5a215-ceb0-471d-a440-5edc8fca37e1:image.png)

## Example: Anim Notify Tag

![image.png](attachment:547f3537-823b-40db-a7b6-1dbc7739ee88:image.png)

![image.png](attachment:699e7586-d339-4365-be01-d0f4e278d1e2:image.png)

![image.png](attachment:47ffddd9-ba93-4851-939a-7a212caf539b:image.png)
</details>

### 3. Inventory System
<details>
## Overview

The inventory system is designed to be a lightweight equivalent to the `AbilitySystemComponent` for items that only have the features required for this game. It manages passive bonuses, on-use effects, item bindings etc. 

It’s completely server-authoritative and data-driven via `FItem` and `UItemInfo` and uses serialized fast array replication to ensure efficient network sync.

---

## Design Goals

- Modular, data-driven inventory that supports both passive and consumable items
- Fully replicated with minimal bandwidth via `FFastArraySerializer`
- UI-agnostic, designed to plug into any layout or input method
- Easy to expand without rewriting logic (e.g., add new item types)

---

## Architecture

- `UInventorySystem` : Actor component that owns and replicates the inventory. Manages input handling, addition/removal logic, RPC calls etc.
- `FInventorySlot`: Represents a single item slot. Designed to be an equivalent to `FGameplayAbilitySpec`. Tracks quantity, assigned tags, effect handles, as well as owning logic for using and clearing the item.
- `FItem` & `UItemInfo`: Defines data for items. Name, description, icon, price etc.
- `FReplicatedInventorySlots : public FFastArraySerializer`: Fast array wrapper used to replicate the inventory state efficiently across networked clients.
- All item logic, including effects and input bindings, is entirely data-driven — no hardcoded behaviors are required.

---

## Example: Add Item ( Inventory )

```cpp
bool UInventorySystem::AddItem(const FString& ItemID, int32 Quantity)
{
	FInventorySlot* Slot = GetSlotOfItem(ItemID);
	if (Slot)
	{
		Slot->Quantity += Quantity;
		Inventory.MarkItemDirty(*Slot);
		return true;
	}

	if (GetInventorySlots().Num() < MaxItemCount)
	{
		FInventorySlot NewSlot = FInventorySlot(GetASC());
		NewSlot.AssignItem(ItemID, Quantity);
		NewSlot.SlotInputTag = GetFirstEmptyInput();
		GetInventorySlots().Add(NewSlot);

		Inventory.MarkItemDirty(NewSlot);
		OnRep_InventorySlots();

		return true;
	}

	return false;
}
```

## Example: Assign Item ( Slot )

```cpp
void FInventorySlot::AssignItem(const FString& InItemID, int32 InQuantity)
{
	ItemID = InItemID;
	Quantity = InQuantity;

	FItem Item = UGameData::GetPrimaryItemInfo(ASC)->GetItemByID(ItemID)->Item;
	
	for (auto Effect : Item.Effects)
	{
		FGameplayEffectContextHandle EffectContext = ASC->MakeEffectContext();
		FActiveGameplayEffectHandle EffectHandle = ASC->ApplyGameplayEffectToSelf(Effect.GetDefaultObject(), 1.0f, EffectContext);
		EffectHandles.Add(EffectHandle);
	}
}
```

![ezgif-71b63e8d3f196d.gif](attachment:fddad835-8bbf-454f-984d-08e39d80bc53:ezgif-71b63e8d3f196d.gif)
</details>

### 4. Extended Ability System Component
<details>
## Overview

`UMyAbilitySystemComponent` is a custom subclass of `UAbilitySystemComponent` tailored to support the tag-based, modular, and rebinding-heavy gameplay of this project.

It allows dynamic ability assignment and rebinds via gameplay tags. Leveling, upgrading, downgrading spells with ease. Input tag based activation and precasting etc.

---

## Design Goals

- Reduce boilerplate for tag-based ability access and input.
- Easy integration with other systems such as UI and shop without direct dependency.
- Expose a simple API to designers with tag-based access.

### Example Key Functions

- `AddAbility(AbilityTag, AutoEquip)` – Adds an ability by tag and optionally assigns it to a free slot
- `EquipAbility(AbilityTag, InputTag)` – Binds an ability to a specific input tag
- `UpgradeAbility` / `DowngradeAbility` – Adjusts ability level and triggers replication
- `AbilityInputTagPressed/Held/Released` – Routes input tags directly to ability specs
- `GetSpecOfInput` / `GetSpecOfAbility` – Tags to internal specs
- `HandleBuyAbility` / `HandleSellAbility` – Handles add/upgrade and sell/downgrade requests from shop.

---

## Example: Ability Input Tag Pressed

```dhall
void UMyAbilitySystemComponent::AbilityInputTagPressed(const FGameplayTag& InputTag)
{
	if (!InputTag.IsValid()) return;

	if (InputTag.MatchesTagExact(WTags::Get().Input_LMB))
	{
		if (const FGameplayAbilitySpec* AbilitySpec = GetPrecastingSpec())
		{
			InvokeReplicatedEvent(EAbilityGenericReplicatedEvent::InputPressed, AbilitySpec->Handle, AbilitySpec->ActivationInfo.GetActivationPredictionKey());
		}
	}
	else if (InputTag.MatchesTagExact(WTags::Get().Input_RMB))
	{
		if (const FGameplayAbilitySpec* AbilitySpec = GetPrecastingSpec())
		{
			CancelAbility(AbilitySpec->Ability);
		}
	}
	else
	{
		FScopedAbilityListLock ActivateScopeLock(*this);
		for (auto& AbilitySpec : GetActivatableAbilities())
			if (AbilitySpec.DynamicAbilityTags.HasTagExact(InputTag))
			{
				if (const FGameplayAbilitySpec* PrecastingSpec = GetPrecastingSpec())
				{
					CancelAbility(PrecastingSpec->Ability);
				}

				AbilitySpecInputPressed(AbilitySpec);

				if (GetGameplayTagCount(WTags::Get().Ability_Base_Casting))
					return;
				if (!AbilitySpec.IsActive())
				{
					TryActivateAbility(AbilitySpec.Handle);
				}
			}
	}
}
```

## Example: Equip Ability

```cpp
void UMyAbilitySystemComponent::EquipAbility(const FGameplayTag& AbilityTag, const FGameplayTag& InputTag)
{
	if (!AbilityTag.IsValid() || !InputTag.IsValid()) return;

	FGameplayTag TargetAbility = GetAbilityOfInput(InputTag);

	if (TargetAbility.IsValid() && TargetAbility.MatchesTagExact(AbilityTag)) return;

	SetOrSwapInputOfAbility(AbilityTag, InputTag);
}

void UMyAbilitySystemComponent::SetOrSwapInputOfAbility(const FGameplayTag& FirstAbilityTag, const FGameplayTag& SecondInputTag)
{
	FGameplayAbilitySpec* FirstSpec = GetSpecOfAbility(FirstAbilityTag);
	if (!FirstSpec) return;

	if (FGameplayAbilitySpec* SecondSpec = GetSpecOfInput(SecondInputTag))
	{
		AssignInputToSpec(*SecondSpec, GetInputFromSpec(*FirstSpec));
		MarkAbilitySpecDirty(*SecondSpec);
	}

	AssignInputToSpec(*FirstSpec, SecondInputTag);
	MarkAbilitySpecDirty(*FirstSpec);

	AbilitiesUpdated.Broadcast();
}
```
</details>

### 5. Modular Shop System
<details>
## Overview

The shop system is fully replicated, server-authoritative framework that allows players to buy, sell, upgrade and downgrade spells and items during intermission phase. All shop interactions are driven by gameplay tags and configurable data assets to ensure the system is extensible and supports different gameplay modes and item/spell pools.

---

## Design Goals

- Ensure all logic is validated server-side to prevent exploits.
- Easy expansion by designers via `DataAsset`s.
- Allow full reconfiguration without hardcoding.

---

## Architecture

- `UPlayerEconomy`: Owned by `PlayerState`, handles currency and transactions.
- `UMyWidgetController`: A middle class base for UI, separates direct dependency between UI blueprint and core game systems.
- `UShopWidgetController`: Extends `UMyWidgetController` for shop specific functionalities

---

## Example Workflow

Buying an Item:

- Player clicks an item in UI
- UI calls `TryBuyItem(const FString ItemId)`
- Client checks: item is valid, have enough gold, have slot to buy etc. to prevent unnecessary network calls.
- Client calls `ServerTryBuyItem_Implementation(const FString& ItemId)`
- Server checks: item is valid, have enough gold, have slot to buy etc.
- If valid, item is added to inventory and gold is deducted

## Example: Buy Spell

```cpp
void UPlayerEconomy::TryBuySpell(const FGameplayTag& AbilityTag)
{
	checkPlayerState();
	checkGamePhase();
	checkSpellTag();
	checkAbility();

	const UPlayerEconomy* PE = PlayerState->GetPlayerEconomy();
	float Price = Ability.Price.GetValueAtLevel(ASC->GetAbilityLevel(AbilityTag) + 1);

	if (!ASC->GetHasEmptyAbilitySlot())
		return;

	if (PE->HasEnoughGold(Price))
	{
		ServerTryBuySpell(AbilityTag);
	}
}

void UPlayerEconomy::ServerTryBuySpell_Implementation(const FGameplayTag& AbilityTag)
{
	checkPlayerState();
	checkGamePhase();
	checkSpellTag();
	checkAbility();

	UPlayerEconomy* PE = PlayerState->GetPlayerEconomy();
	float Price = Ability.Price.GetValueAtLevel(ASC->GetAbilityLevel(AbilityTag) + 1);

	if (!ASC->GetHasEmptyAbilitySlot())
		return;
	
	if (PE->RemoveGold(Price))
	{
		ASC->HandleBuyAbility(AbilityTag);
	}
}
```

![ezgif-7ec8ea79238230.gif](attachment:deb61e85-7966-4318-be4c-470794b38b40:ezgif-7ec8ea79238230.gif)

## 5.1 Description Format

To streamline tooltip creation, all ability descriptions are dynamically generated inside the `PrimaryAbilityInfo : UPrimaryDataAsset`. This system allows for quick, scalable formatting with rich text support, and can be extended easily when new variables are introduced.

### Named Arguments

```cpp
struct FDescriptionNamedArguments
{
	FString _Level0 = FString::Printf(TEXT("_Level0"));
	FString _Level1 = FString::Printf(TEXT("_Level1"));
	FString _CD0 = FString::Printf(TEXT("_CD0"));
	FString _CD1 = FString::Printf(TEXT("_CD1"));
	FString _Cost0 = FString::Printf(TEXT("_Cost0"));
	FString _Cost1 = FString::Printf(TEXT("_Cost1"));
	FString _FireDmg0 = FString::Printf(TEXT("_FireDmg0"));
	FString _FireDmg1 = FString::Printf(TEXT("_FireDmg1"));
	FString _FrostDmg0 = FString::Printf(TEXT("_FrostDmg0"));
	FString _FrostDmg1 = FString::Printf(TEXT("_FrostDmg1"));
	FString _LightDmg0 = FString::Printf(TEXT("_LightDmg0"));
	FString _LightDmg1 = FString::Printf(TEXT("_LightDmg1"));
	FString _ArcDmg0 = FString::Printf(TEXT("_ArcDmg0"));
	FString _ArcDmg1 = FString::Printf(TEXT("_ArcDmg1"));
	FString _RadDmg0 = FString::Printf(TEXT("_RadDmg0"));
	FString _RadDmg1 = FString::Printf(TEXT("_RadDmg1"));
};
```

### Format Logic

When formatting begins, all placeholders are bound to their current and next-level values. If the ability is an instance of `UEffectAbility`, damage values are extracted from tags using a helper function.

```cpp
if (AbilityDefault)
	{
		FDescriptionNamedArguments Args;
		FormattedText = FText::FormatNamed(
			FormattedText,
			Args._Level0, Level,
			Args._Level1, Level + 1,
			Args._Cost0, FText::AsNumber(AbilityDefault->GetCost(Level), &FormatOptions),
			Args._Cost1, FText::AsNumber(AbilityDefault->GetCost(Level + 1), &FormatOptions),
			Args._CD0, FText::AsNumber(AbilityDefault->GetCooldown(Level), &FormatOptions),
			Args._CD1, FText::AsNumber(AbilityDefault->GetCooldown(Level + 1), &FormatOptions)
		);

		if (const UEffectAbility* EA = Cast<UEffectAbility>(AbilityDefault))
		{
			FormattedText = FText::FormatNamed(
				FormattedText,
				Args._FireDmg0, FText::AsNumber(EA->GetDamageAtLevel(Level, Tags.Damage_Type_Fire), &FormatOptions),
				Args._FireDmg1, FText::AsNumber(EA->GetDamageAtLevel(Level + 1, Tags.Damage_Type_Fire), &FormatOptions),
				Args._FrostDmg0, FText::AsNumber(EA->GetDamageAtLevel(Level, Tags.Damage_Type_Frost), &FormatOptions),
				Args._FrostDmg1, FText::AsNumber(EA->GetDamageAtLevel(Level + 1, Tags.Damage_Type_Frost), &FormatOptions),
				Args._LightDmg0, FText::AsNumber(EA->GetDamageAtLevel(Level, Tags.Damage_Type_Lightning), &FormatOptions),
				Args._LightDmg1, FText::AsNumber(EA->GetDamageAtLevel(Level + 1, Tags.Damage_Type_Lightning), &FormatOptions),
				Args._ArcDmg0, FText::AsNumber(EA->GetDamageAtLevel(Level, Tags.Damage_Type_Arcane), &FormatOptions),
				Args._ArcDmg1, FText::AsNumber(EA->GetDamageAtLevel(Level + 1, Tags.Damage_Type_Arcane), &FormatOptions),
				Args._RadDmg0, FText::AsNumber(EA->GetDamageAtLevel(Level, Tags.Damage_Type_Radiant), &FormatOptions),
				Args._RadDmg1, FText::AsNumber(EA->GetDamageAtLevel(Level + 1, Tags.Damage_Type_Radiant), &FormatOptions)
			);
		}	
```

Description Template:`<Body2>Hurls a blazing orb that bursts on impact, searing foes for </><Fire>{_FireDmg0}</><Footer> > </><Fire>{_FireDmg1} fire</><Body2> damage and igniting them with </><Fire>burn</><Body2>.</>`

Result:

![image.png](attachment:1c6eef9d-ad33-4008-b7f3-113f1aed5ea3:image.png)
</details>

### 6. Unified Damage Model
<details>
## Overview

The entire game uses a single centralized `UExecCalc_Damage` class to calculate all damage. Each damage type (e.g. Fire, Frost, Arcane etc.) is handled modularly using `GameplayTag`s, paired with corresponding resistances and penetrations. The final result is a clean, extendable and data-driven system that can easily scale with new damage types or mechanics.

---

## Design Goals

- Centralize and unify all damage logic under a single Execution Calculation.
- Support tag-driven behaviors.
- Easily extendable for new mechanics like critical hits, blocking, or future damage types.

---

## Architecture

- `ExecCalc_Damage`: 
A centralized execution calculation that reads `SetByCallerMagnitudes` for each damage-type tag and applies resistance/penetration scaling.
- `DamageStatics`: Struct that captures relevant attributes for both source and target.

---

## Example: Damage Execution Skeleton

```cpp
#define GetCapAttrValue_Clamped(ResultVar, Attribute, EvalParams, MinVal, MaxVal)  \
	float ResultVar = 0;\
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(Attribute, EvalParams, ResultVar);\
	ResultVar = FMath::Clamp<float>(ResultVar, MinVal, MaxVal);
	
struct MyDamageStatics
{
	DECLARE_ATTRIBUTE_CAPTUREDEF(FirePenetration);
	...
	DECLARE_ATTRIBUTE_CAPTUREDEF(FireResistance);
	...

	MyDamageStatics()
	{
		DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, FirePenetration, Source, false);
		...
		DEFINE_ATTRIBUTE_CAPTUREDEF(UMyAttributeSet, FireResistance, Target, false);
		...
	}
};
```

```cpp
void UExecCalc_Damage::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
	...
	TMap<FGameplayTag, FGameplayEffectAttributeCaptureDefinition> TagsToCaptureDefs;
	TagsToCaptureDefs.Add(Tags.Attribute_Penetration_Fire, DamageStatics().FirePenetrationDef);
	...

	float Damage = 0.f;
  for (const auto& Pair : Tags.DamageTypesToBonuses)
  {
	  ...
    float DamageTypeValue = Spec.GetSetByCallerMagnitude(DamageTypeTag, false);
		GetCapAttrValue_Clamped(TResistance, TagsToCaptureDefs[ResistanceTag], EvalParams, 0, 100);
		GetCapAttrValue_Clamped(SPenetration, TagsToCaptureDefs[PenetrationTag], EvalParams, 0, 100);
		DamageTypeValue *= (100.f - TResistance + SPenetration) / 100;

		Damage += DamageTypeValue;
  }
	
	const FGameplayModifierEvaluatedData EvalData(UMyAttributeSet::GetIncomingDamageAttribute(), EGameplayModOp::Additive, Damage);
	OutExecutionOutput.AddOutputModifier(EvalData);
}
```

## Example: New Logic implementation

```cpp
if (!bIsDebuff)
	{
	 	GetCapAttrValue_Max(TBlockChance, DamageStatics().BlockChanceDef, EvalParams, 0);
	 	GetCapAttrValue_Max(TBlockAmount, DamageStatics().BlockChanceDef, EvalParams, 0);
	 	bool bBlocked = FMath::RandRange(0, 100) < TBlockChance;
	 	if (bBlocked) Damage -= TBlockAmount;
	 	UEffectContextFunctions::SetIsBlockedHit(EffectContextHandle, bBlocked);
	}
```

This logic can be inserted before or after the main damage calculation, depending on whether blocking should reduce base input or the final resolved damage.
</details>

### 7. Projectile Framework
<details>
## Overview

Projectiles in ProjectW are modular actors designed for spells and effects with flexible movement, collision, and damage logic. The system supports both simple and complex behavior, from straight-line projectiles to spline-following missiles that bounce, reflect, or apply effects on overlap, implemented using C++ inheritance, component composition, and seamless integration with the Gameplay Ability System (GAS).

This system allows a single spell definition to instantiate completely different projectile behaviors by changing only the class and some parameters, making the system extremely scalable and data-driven.

---

## Design Goals

- Enable highly reusable and configurable projectile behavior.
- Cleanly separate team relevance, collision rules, and effect application.
- Support both physics-based and spline-based motion.
- Integrate tightly with GAS and `SpellParams` for data-driven behavior.

---

## Architecture

- `AWBaseSpellActor`: Shared base class for all spell actors. Handles replication of `SpellParams`, team relevance checks (`IsEnemy`, `IsAlly`, `IsSelf`), and collision filtering using `TargetCollisionTypes` and `TargetEffectTypes`.
- `ABaseProjectile`: Main projectile class. Implements overlap logic, hit tracking, projectile-vs-projectile interactions and scalable impact effects. Effect application is done through `ApplyDamageEffect()` using GAS.
- `ASplineMovementProjectile`: Subclass of `ABaseProjectile` for curved projectile paths. Follows a spline until a configurable distance, then transitions to standard velocity-based motion or ends. Supports bounce and rotation update.
- `AMyEffectActor`: General-purpose actor to apply one or more `GameplayEffect`s based on overlap events and customizable application/removal policies.

---

## Key Features

- Intelligent Hit Logic
    - Prevents repeated hits through `TargetHitCooldown`, `LastHitActor`, and `LastHitTimes`.
    - Supports multi-hit projectiles or destroys after N hits via `DestroyOnHit`.
    - Uses Interfaces (e.g., `UNoCollision`, `UDontTriggerCollision`) to prevent unwanted collisions, such as a Gravity Ball pulling in other projectiles without triggering their effects and hardcoding
- Projectile vs Projectile Policy
    - Defines interaction behavior between projectiles: Ignore, Destroy, or Bounce.
- Target Relevancy Filtering
    - Filters targets dynamically using `TargetCollisionTypes` and `TargetEffectTypes` defined in `SpellParams`.
    - Supports precise definitions like “hit enemies but apply effects to self and allies.”
- GAS Integration and SpellParams
    - `SpellParams` carries ability context, including source/target ASC, relevant tags and custom float/vector/int… values.
    - Replicated relevant data allow client-side prediction and server-side effect application
    - Damage and effects are applied using custom context via `UEffectContextFunctions::ApplyDamageEffect`.
- Spline-Based Movement
    - Dynamic spline and speed scaling to allow the spline ends at the exact target location within spell’s range.
    - Smooth transition to projectile movement upon bounce or other external factors.

## Spell Index & Tag System

In ProjectW, all abilities and spells are categorized using a centralized tagging system that defines their type, damage class, pricing, and behavior. This modular setup is used across gameplay, UI, cooldown handling, and effect application.

Below is a visual snapshot of the internal ability data table used to configure and classify spells in the game:

![image.png](attachment:16941c9e-130b-4b09-8aee-7aaf356b6e3c:image.png)

---

## Example: Overlap Flow

```cpp
void ABaseProjectile::OnOverlap(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
	if (!OtherActor || OtherActor == this) return;
	if (ShouldIgnoreOverlap(OtherActor)) return;
	if (IsActorProjectile(OtherActor))
	{
		HandleProjectileCollision(OtherActor);
		return;
	}
	if (!CanHitTarget(OtherActor)) return;
	if (!SpellParams || !SpellParams->SourceASC) return;
	if (!CheckForCollisionTarget(OtherActor)) return;

	LastHitActor = OtherActor;
	LastHitTimes.FindOrAdd(OtherActor) = GetWorld()->GetTimeSeconds();
	HitCount++;

	if (HitCount == DestroyOnHit)
		Sphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);

	OnProjectileOverlap(OtherActor);
}
```

## Example: Spell Params

```cpp
UPROPERTY(BlueprintReadWrite)
TObjectPtr<UObject> SourceAvatar = nullptr;
UPROPERTY(BlueprintReadOnly, ReplicatedUsing=OnRep_UniqueId)
FUniqueNetIdRepl UniqueId = FUniqueNetIdRepl();
UPROPERTY(BlueprintReadWrite)
TObjectPtr<UAbilitySystemComponent> SourceASC = nullptr;
UPROPERTY(BlueprintReadWrite)
TObjectPtr<UAbilitySystemComponent> TargetASC = nullptr;
UPROPERTY(BlueprintReadWrite, Replicated)
TArray<ERelevancy> TargetCollisionTypes;
UPROPERTY(BlueprintReadWrite)
TArray<ERelevancy> TargetEffectTypes;
UPROPERTY(BlueprintReadWrite, Replicated)
float AbilityLevel = 0;

UPROPERTY(BlueprintReadWrite)
TObjectPtr<UMultiDataArray> More = nullptr;
```

`MultiDataArray` is a custom value holder with Key-Value types to allow any extra data required by the ability with ease.

## Example: Boomerang Spell Spline to Projectile Switch

![image.png](attachment:6a0818fa-1e0b-4a34-b1f7-c360f2173b65:image.png)

## Example: Firebolt Setup

![image.png](attachment:477202a3-0255-4710-8b0c-2f7e4b3218e1:image.png)

## Visual Showcase

![Shield.gif](attachment:4a075346-15c4-40ab-9be6-b12a7a1cb89c:Shield.gif)

![Frostbolt.gif](attachment:f7923346-3e0f-48bc-bc9c-a27b47e18a6f:Frostbolt.gif)

![Homing.gif](attachment:005b0e8e-fd71-4be6-86f7-7685e13c515d:Homing.gif)

![Blink.gif](attachment:23d0f89a-9230-445f-91a9-9243f4d01988:Blink.gif)
</details>

### 8. Structured Logging System
<details>
## Overview

ProjectW features a highly modular and developer friendly structured logging system designed to streamline debugging, profiling and gameplay analysis. It includes lightweight macro wrappers around Unreal’s logging and screen messaging systems, and a powerful Blueprint-exposed logging utility for serializing complex data structures into JSON at runtime.

This system allows fast debugging with contextual information such as function name, line number, player identity and categorized verbosity. This logs are configurable globally via a custom `UDebugConfig`.

## Core Objectives

- Simplify runtime logging with color-coded macros and context-rich messages.
- Support per-category logging through predefined `LogCategory`s.
- Allow conditional logging (debug-only, detailed-only).
- Enable logging complex `USTRUCT`s and `UObject` data as JSON.

## System Components

- Macro Layers
    - Display Macros:
        
        `log`, `warn`, and `error` display color-coded messages (white/yellow/red) on-screen using `AddOnScreenDebugMessage`.
        
        `logf`, `warnf`, `errorf` versions support formatting.
        
        `logkey`, `warnkey`, etc., allow persistent screen updates using key-based override.
        
        `logkeyf`, `warnkeyf`, etc., versions support key and formatting 
        
        and more.
        
    - Console Macros:
        
        `consolelog`, `consolewarn`, `consoleerror` output to the UE_LOG console using `LogProjectW` or custom subcategories (e.g., `LogCombat`, `LogNetwork`).
        
    - Contextual Logging:
        
        Macros like `logfunc`, `logfuncp`, and `logfuncmsgf` embed the class name, function name, and player identity into logs, assisting traceability during multiplayer debugging.
        
    - Conditionals:
        
        All macros honor developer config toggles like `bDebugFunc` and `bDebugExtraDetails`, ensuring debug logs are gated cleanly in packaged builds.
        

## Example: Runtime Context Macros

```cpp
#define CUR_CLASS_FUNC (FString(__FUNCTION__))

#define logfunc() checkenabled() UE_LOG(LogWFunc, Log, TEXT("%s"), *CUR_CLASS_FUNC)
#define logfuncp() checkenabled() UE_LOG(LogWFunc, Log, TEXT("Called by: %s - %s"), playername(), *CUR_CLASS_FUNC)
#define logfuncmsgf(Format, ...) checkenabled() UE_LOG(LogWFunc, Log, TEXT("%s >> %s"), *CUR_CLASS_FUNC, *FString::Printf(TEXT(Format), ##__VA_ARGS__))
#define logfuncpmsgf(Format, ...) checkenabled() UE_LOG(LogWFunc, Log, TEXT("Called by: %s - %s >> %s"), playername(), *CUR_CLASS_FUNC, *FString::Printf(TEXT(Format), ##__VA_ARGS__))
...

#define logfuncdpmsgf(Format, ...) checkenabled() checkifdetailed() UE_LOG(LogWFunc, Log, TEXT("Called by: %s - %s >> %s"), playername(), *CUR_CLASS_FUNC, *FString::Printf(TEXT(Format), ##__VA_ARGS__))
...
```

## Data Inspection in Blueprints

`UDebugLog::LogAsJson()` is a custom Blueprint-exposed thunk that serializes any supported property into a JSON-formatted string, logging the result to the designated category for easy inspection in logs.

It supports:

- Primitive types (`int`, `float`, `bool`, `FString`)
- `USTRUCT`s and `UObject`s (via `FJsonObjectConverter`)
- `TArray` of primitives or structs
- Custom conversions for types like `FSessionsSearchSetting` and `FSessionPropertyKeyPair`
- Optional `Tags` array to help group or filter logs

```cpp
FUNCTION(BlueprintCallable, CustomThunk, Category = "Debug", meta = (CustomStructureParam = "Property", AdvancedDisplay = "Prefix, Suffix, LogCategory, Tags", AutoCreateRefTerm="Tags"))
static void LogAsJson(const int32& Property, UObject* WorldContext, const FString& Prefix, const FString& Suffix, ELogCategory LogCategory, const TArray<FString>& Tags);
	
DECLARE_FUNCTION(execLogAsJson)
{
	Stack.StepCompiledIn<FProperty>(NULL);
	FProperty* Property = Stack.MostRecentProperty;
	void* ValuePtr = Stack.MostRecentPropertyAddress;

	P_GET_OBJECT(UObject, WorldContext);
	P_GET_PROPERTY(FStrProperty, Prefix);
	P_GET_PROPERTY(FStrProperty, Suffix);
	P_GET_PROPERTY(FByteProperty, LogCategory);
	P_GET_TARRAY(FString, Tags);

	P_FINISH;

	P_NATIVE_BEGIN;
		if(UWConfig::GetDebugConfig()->bDebugFunc) return;
	
		FString OutputString;
		FString PropertyName = Property->GetNameCPP();

		if (Property->IsA(FEnumProperty::StaticClass()))
		{
			FEnumProperty* EnumProperty = CastField<FEnumProperty>(Property);
			int64 EnumValue = EnumProperty->GetUnderlyingProperty()->GetSignedIntPropertyValue(ValuePtr);
			FString EnumName = EnumProperty->GetEnum()->GetNameStringByValue(EnumValue);
			OutputString = FString::Printf(TEXT("%s%s: %s%s"), *Prefix, *PropertyName, *EnumName, *Suffix);
		}
		else if ...
```

![image.png](attachment:c13ba4de-d5f3-45ca-b8e5-61f95c4ff87b:image.png)

![image.png](attachment:3cf0b0c6-fa73-4b18-bf76-7eecaef95ced:image.png)

## Use Cases

| Scenario | Recommended Macro |  |
| --- | --- | --- |
| Logging a damage calculation | `logfuncmsgf(TEXT("Final Damage: %f"), Value)` |  |
| Displaying real-time velocity | `logkeyf(1, TEXT("Velocity: %s"), *Velocity.ToString())` |  |
| Printing a struct on overlap (BP) | `LogAsJson(MyStruct, this, "Hit: ", "", ELogCategory::LogCombat, Tags)` |  |
| Tracing player-triggered action | `logfuncpmsgf(TEXT("Pressed button: %s"), *ButtonName)` |  |

The logging system in ProjectW significantly reduces the friction of debugging networked gameplay, tracking player input, and analyzing state changes across the game loop, without requiring verbose boilerplate or intrusive breakpoints.
</details>

### 9. Local & Global Timer Management (WTM)
<details>
## Overview

ProjectW implements a two-tiered timer strategy to meet both global and localized timing needs. The WTM (ProjectW Timer Manager) system provides centralized, key-driven global timers, while lightweight actor-local timer macros streamline simple, context-bound usage within gameplay classes.

## Design Goals

- Provide centralized and thread-safe control over global timers using named keys.
- Offer reusable delay logic.
- Enable safe and simple timer cancellation to prevent latent callbacks after object destruction.
- Support local, lifecycle-bound timers using simple macros with no boilerplate.
- Simplify creation of timers.

---

## Global Timer System (WTM)

The `WTM` static class allows fully decoupled and named control over one-shot and repeating timers. It supports standard behaviors like waiting until a condition is true, delaying execution, and automatically cleaning up after execution or cancellation.

### Key Features:

- Thread-safe via `FCriticalSection` and `FScopeLock`.
- Timer uniqueness via `FString` keys.
- Self-cancelling logic on execution.
- Supports delayed functions, conditional polling, and safe cleanup.

### Utility Functions:

- `Delay(Name, Seconds, Callback)`
- `DelayRepeat(Name, Seconds, Callback returning bool)`
- `Cancel(Name)`
- `CancelAll()`

## Example:

```cpp
WTM::Delay("LavaWarning", 3.0f, []()
{
	LogScreen("Lava expanding!");
});

WTM::DelayRepeat("CheckHealth", 1.0f, []() -> bool
{
	if (PlayerHealth < 10.f)
	{
		WarnScreen("Low HP!");
		return true; // Stop repeating
	}
	return false; // Keep repeating
});
```

## Local Timer Macros (Actor Scoped)

For Timers scoped to an actor’s lifecycle and don't require global tracking, ProjectW includes streamlined macros to reduce boilerplate. These use Unreal's `FTimerManager` internally and require access to `GetWorld()`.

### Macros:

```cpp
// Declare a timer handle in your header
DECLARE_TIMER(DashCooldown)

// One-shot timer in .cpp
SET_TIMER(DashCooldown, 2.0f, [this]() {
    bCanDash = true;
});

// Repeating timer
SET_TIMER_LOOP(SpawnLoop, 1.0f, [this]() {
    TrySpawnMinion();
});

// Safely clear timer
CLEAR_TIMER(DashCooldown)
```

### Benefits:

- Clean and concise syntax for simple actor or component bound timers.
- Automatically prevents invalid handle access.
- Ideal for gameplay systems where timers should die with their owning actor.

## Choosing Between WTM vs Local Timer Macros

| Feature | `WTM` (Global) | Local Timer Macros |
| --- | --- | --- |
| Timer lifetime | Independent, global | Bound to actor/component |
| Uniqueness | Named via `FString` keys | One per handle (per macro) |
| Cancellation | By key or bulk | Explicit per-instance |
| Scope | Game/system-wide | Class-local |
| Thread safety | Yes (via `FCriticalSection`) | No (assumes single-threaded use) |

## Conclusion

By combining the WTM system for globally coordinated delays and the local macros for actor specific control, ProjectW ensures that timer logic remains clear, maintainable, and optimized for both system-level and gameplay-level use cases.
</details>

### 10. Extended Sessions
<details>
## Overview

This includes everything from searching session and listing them, to accepting invites to join a session. It’s based on Advanced Steam Session plugin, extended for easy of use for ProjectW. Designed for listen-server multiplayer with Steam support.

## Design Goals

- Easy and modular setup for new data sessions has to hold, such as game mode, lobby names, password etc. and easily extendable to filter or show other stuff like starting gold, rounds etc. in the server list if required.
- Local filtering with cached data of servers.
- Easily integration with UI, main menu, server list.

---

## System Breakdown

While not heavily architectural, the session system relies on a few key components:

- Function Libraries: Reusable Blueprint nodes for easily handling the session data and its related variables.
- Game Instance: Core logic for managing session state, creating, finding, joining sessions etc.

## Example Functions

These Blueprint utilities simplify interaction with session data:

- `Make ... Search Property`: Used to safely generate session properties without relying on raw string keys
- `Make BPProperties From Session Properties`: Converts a structured session config into key-value pairs for adding to the session
- `Break BPResult Into Session Result`: Breaks the default `Session Result` and exposes custom session data in a structured and error-free way

### Functions in Use:

![image.png](attachment:88889595-38b1-4f6d-8535-fe944ce7c471:image.png)
