---
title: Unreal Engine - Game Framework
date: 2022-03-31 11:58:47 +07:00
---

# Game Flow Diagram
![GameFlowChart](/assets/img/posts/GameFlowChart.png)

**Initializing from start:**
* UGameEngine::Init
* UGameInstance::InitializeStandalone()
   * UGameInstance::Init() - CreateUOnlineSession and Register Delegates()
* UGameEngine::Start
* UGameInstance::StartGameInstance() =>
   * UEngine::LoadMap()
   * UWorld::InitializeActorsForPlay() - Call register components on all actor components in all levels. Note: Construction scripts are rerun in uncooked mode
      * UActorComponent::RegisterComponent() - Adding itself to its owner and inside owner’s world, possibly creating rendering/physics state
   * ABBGameModeBase::InitGame - Create the Game Session and register FGameDelegates (ex: PreCommitMapChangeDelegate, HandleDisconnectDelegate)
   * Ulevel::RouteActorInitialize() =>
      * Actor::PreInitializeComponents() - On all actors in the level
      * Iterate through Ulevel::Actors[] and call these functions on them one at a time
      * Actor::InitializeComponents() =>
         * UActorComponent::Activate() - Sets Component Tick To Be Enables & bIsActive = true
         * UActorComponent::InitializeComponent() - Place for components to Initialize themselves before BeginPlay (Actor or Component for anything in the world)
      * PostInitializeComponents() - Code that can run after gaurantee that all components have been initialized
      * Iterate through all Ulevel::ActorsToBeginPlay[] and call BeginPlay() to allows code to run with assumption that all other level actors have been PostInitializeComponents()
      * Not sure why this is here instead of the main call to BeginPlay(possibly for networked late joins?)
   * FWorldDelegates::OnWorldInitializedActors.Broadcast(OnActorInitParams)

**Begin play:**
* Uworld::BeginPlay()
* GameMode::StartPlay()
  * GameMode::StartMatch()
  * GameState::HandleBeginPlay()
    * AWorldSettings::NotifyBeginPlay()
    * Actor::BeginPlay(), for all actors
      * UActorComponent::RegisterAllComponentTickFunctions() - Allows components to register multiple tick functions (ex: Physics tick, cloth tick in skeletalmeshcomponent)
      * UActorComponent::BeginPlay()

**FEngineLoop::Tick():**
* UGameEngine::Tick()
* Uworld::Tick()
  * FTiskTaskManagerInterface::StartFrame()
  * Queues up all the TickFunctions according to their dependency graph & TickGroup
  * Ticking within each group is done by a dependency graph (AddTickPrerequisite) of tick functions of various objects during TickFunction registration
  * This function might change what TickGroup something runs in according to the prerequisite tick function’s tickgroup
  * Actor Components do not necessarily tick after their owner Actor
  * Calls RunTickGroup() for various tick groups which ticks components
  * FTickableGameObject::TickObjects() - ticks UObjects or anything that derives from FTickableGameObject (e.g. SceneCapturerCubes or LevelSequencePlayers )
* FTicker::GetCoreTicker().Tick(FApp::GetDeltaTime()) - Ticks all objects of type FTickerObjectBase. Ex: FHttpManager, FAvfMediaPlayer, FVoiceCapture, FSteamSocketSubsystem)
* This would be great place to add Objects that need to tick at the end of the frame that are engine/world agnostic
* Good possible place for our own UDP network ticking replication

**GameMode Flow:**
* InitGame()
* InitGameState()
* PostInitializeComponents() creates
    * AGameStateBase and calls InitGameState()
    * AGameNetworkManager which handles game-specific networking management (cheat detection, bandwidth management, etc)
* ChoosePlayerStart_Implementation()
* PostLogin()
* HandleStartingNewPlayer_Implementation()
* ChoosePlayerStart_Implementation()
* SetPlayerDefaults()
* RestartPlayerAtPlayerStart()
* StartPlay()
    * GameMode::StartMatch()
    * GameState::HandleBeginPlay()
        * AWorldSettings::NotifyBeginPlay()
        * Actor::BeginPlay(), for all actors
            * UActorComponent::RegisterAllComponentTickFunctions() - Allows components to register multiple tick functions (ex: Physics tick, cloth tick in skeletalmeshcomponent)
            * UActorComponent::BeginPlay()

**From:** https://bebylon.dev/ue4guide/gameplay-programming/master-engine-flow/master-engine-flow/

# Lifecycle Break Down
![ActorLifeCycle](/assets/img/posts/ActorLifeCycle.png)

**Load from Disk:**

This path occurs for any Actor that is already in the level, like when LoadMap occurs, or AddToWorld (from streaming or sub levels) is called.
1. Actors in a package/level are loaded from disk.
2. PostLoad - is called by serialized Actor after they have finished loading from disk. Any custom versioning and fixup behavior should go here. PostLoad is mutually exclusive with PostActorCreated.
3. InitializeActorsForPlay
4. RouteActorInitialize for any non-initialized Actors (covers seamless travel carry over).
    - PreInitializeComponents - Called before InitializeComponent is called on the Actor's components.
    - InitializeComponent - Helper function for the creation of each component defined on the Actor.
    - PostInitializeComponents - Called after the Actor's components have been initialized.
5. BeginPlay - Called when the level is started.

**Play in Editor:**

The Play in Editor path is mostly the same as Load from Disk, however the Actors are never loaded from disk, they are copied from the Editor.
1. Actors in the Editor are duplicated into a new World.
2. PostDuplicate is called.
3. InitializeActorsForPlay
4. RouteActorInitialize for any non-initialized Actors (covers seamless travel carry over).
    - PreInitializeComponents - Called before InitializeComponent is called on the Actor's components.
    - InitializeComponent - Helper function for the creation of each component defined on the Actor.
    - PostInitializeComponents - Called after the Actor's components have been initialized.
5. BeginPlay - Called when the level is started.

**Spawning:**

When spawning (instancing) an Actor, this is the path that will be followed.
1. SpawnActor called
2. PostSpawnInitialize
3. PostActorCreated - called for spawned Actors after its creation, constructor like behavior should go here. PostActorCreated is mutually exclusive with PostLoad.
4. ExecuteConstruction:
    - OnConstruction - The construction of the Actor, this is where Blueprint Actors have their components created and Blueprint variables are initialized.
5. PostActorConstruction:
    - PreInitializeComponents - Called before InitializeComponent is called on the Actor's components.
    - InitializeComponent - Helper function for the creation of each component defined on the Actor.
    - PostInitializeComponents - Called after the Actor's components have been initialized.
6. OnActorSpawned broadcast on UWorld.
7. BeginPlay is called.

**Deferred Spawn:**

An Actor can be Deferred Spawned by having any properties set to "Expose on Spawn."
1. SpawnActorDeferred - meant to spawn procedural Actors, allows additional setup before Blueprint construction script.
2. Everything in SpawnActor occurs, but after PostActorCreated the following occurs:
    - Do setup / call various "initialization functions" with a valid but incomplete Actor instance.
    - FinishSpawningActor - called to Finalize the Actor, picks up at ExecuteConstruction in the Spawn Actor line.

**During Gameplay:**

These are completely optional, as many Actors will not actually die during play.
- Destroy - is called manually by game any time an Actor is meant to be removed, but gameplay is still occurring. The Actor is marked pending kill and removed from Level's array of Actors.
- EndPlay - Called in several places to guarantee the life of the Actor is coming to an end. During play, Destroy will fire this, as well Level Transitions, and if a streaming level containing the Actor is unloaded. All the places EndPlay is called from:
    - Explicit call to Destroy.
    - Play in Editor Ended.
    - Level Transition (seamless travel or load map).
    - A streaming level containing the Actor is unloaded.
    - The lifetime of the Actor has expired.
    - Application shut down (All Actors are Destroyed).

Regardless of how this happens, the Actor will be marked RF_PendingKill so during the next garbage collection cycle it will be deallocated. Also, rather than checking for pending kill manually, consider using an FWeakObjectPtr<AActor> as it is cleaner.

OnDestroy - This is a legacy response to Destroy. You should probably move anything here to EndPlay as it is called by level transition and other game cleanup functions.

**Garbage Collection:**

Some time after an object has been marked for destruction, Garbage Collection will actually remove it from memory, freeing any resources it was using.

The following functions are called on the object during its destruction:
1. BeginDestroy - This is the object's chance to free up memory and handle other multithreaded resources (ie: graphics thread proxy objects). Most gameplay functionality related to being destroyed should have been handled earlier, in EndPlay.
2. IsReadyForFinishDestroy - The garbage collection process will call this function to determine whether or not the object is ready to be deallocated permanently. By returning false, this function can defer actual destruction of the object until the next garbage collection pass.
3. FinishDestroy - Finally, the object is really going to be destroyed, and this is another chance to free up internal data structures. This is the last call before memory is freed.

**From:** https://docs.unrealengine.com/en-US/ProgrammingAndScripting/ProgrammingWithCPP/UnrealArchitecture/Actors/ActorLifecycle/index.html

# UGameInstance
GameInstance has one instance that persists throughout the lifetime of the game. Traveling between maps and menus will maintain the same instance of this class. This class can be used to provide event hooks of handling network errors, loading of user data like game settings and generally features that are not relevant to only a specific level of your game.

**Accessing GameInstance:**
- **GetWorld()->GetGameInstance<T>();** - where T is the class type eg. GetGameInstance<UGameInstance>() or your own derived type.
- **Actor->GetGameInstance();**

**Personal Remarks:**
- Generally not used a whole lot early in your game project. Doesn’t do anything critical unless you’re diving deeper into the development (things like game sessions, demo playback or persisting some data between levels)

**From:** https://www.tomlooman.com/ue4-gameplay-framework/

# AGameMode
The primary class that specifies which classes to use (PlayerController, Pawn, HUD, GameState, PlayerState) and commonly used to specify game rules for modes such as ‘Capture the Flag’ where it could handle the flags, or to handle ‘wave spawns’ in a wave based shooter. Handles other key features like spawning the player as well.

GameMode is a sub-class of GameModeBase and contains a few more features originally used by Unreal Tournament such as MatchState and other shooter type features.

In multiplayer the GameMode class only exists on the server! This means no client ever has an instance. For singleplayer games this has no effect. To replicate functions and store data you need for your GameMode you should consider using GameState which does exist on all clients and is made for specifically this purpose.

**Accessing GameMode:** 
- **GetWorld()->GetAuthGameMode();** - GetWorld is available from any Actor instance.
- **GetGameState();** - return gamestate for replicating functions and/or variables
- **InitGame(…)** - initialize some of the game rules including those provided on the URL (eg. “MyMap?MaxPlayersPerTeam=2”) which can be passed in with loading levels in the game.

**From:** https://www.tomlooman.com/ue4-gameplay-framework/

# APlayerController
The primary class for a player, receiving the input of the user. A PlayerController itself is not visually shown in the environment instead, it will ‘Possess’ a Pawn instance which defines the visual and physical representation of that player in the world. During gameplay a player may possess a number of different pawns (eg. a vehicle, or a fresh copy of the Pawn when respawning) while the PlayerController instance remains the same throughout the duration of the level. This is important as there may also be moments where the PlayerController does not possess any pawn at all. This means things like the opening of menus should be added to the PlayerController and not the Pawn class.

In multiplayer games the PlayerController only exists on the owning client and the server. This means in a game of 4 players, the server has 4 player controllers, and each client only has 1. This is very important in understanding where to put certain variables as when all players require a player’s variable to be replicated it should never exist in the PlayerController but in the Pawn or even PlayerState (covered later) instead.

**Accessing PlayerControllers:** 
- **GetWorld()->GetPlayerControllerIterator();** - GetWorld is available in any Actor instance
- **PlayerState->GetOwner();** - owner of playerstate is of type PlayerController, you will need to cast it to PlayerController yourself.
- **Pawn->GetController();** - Only set when the pawn is currently ‘possessed’ (ie. controlled) by a PlayerController.
This class contains a PlayerCameraManager which handles view targets and camera transforms including camera shakes. Another important class the PlayerController handles is HUD (covered below) which is used for Canvas rendering (not used as much now that UMG is available) and can be great for managing some data you want to pass on to your UMG interface.

Whenever a new players joins the GameMode, a new PlayerController is created for this player via Login() in the GameModeBase class.

**From:** https://www.tomlooman.com/ue4-gameplay-framework/

# AIController
Similar in concept to the PlayerController class, but purely for AI agents instead of players. Will possess a Pawn (see below) much like the PlayerController and is the ‘brain’ of the agent. Where the Pawn is the visual representation of the agent, the AIController is the decision maker.

Behavior Trees run via this controller and so should any handling of perception data (what the AI can see and hear) and push the decisions into the Pawn to act.

**Accessing AIController:**
- AIControllers can be accessed in the same way as PlayerControllers from inside a Pawn (via “Get Controller”) and in Blueprint there is a “GetAIController” function available anywhere, the input is expected to be the Pawn that is controlled by AI.

**Spawning:** 
- AIControllers can be spawned automatically if desired by setting the variable “Auto Possess AI” in the Pawn class. Make sure you set “AI Controller Class” variable in the Pawn to your desired class when using this.

**From:** https://www.tomlooman.com/ue4-gameplay-framework/

# APlayerState
Container for variables to replicate between client/server for a specific player. Data container that doesn’t run much logic of his own. Since PlayerController is not available on all clients and Pawns are often destroyed as players die, the PlayerState is a good place to store data that must be replicated or persist between deaths.

**Accessing PlayerState:**
- Pawn has it as variable Pawn->PlayerState, also available in Controller->PlayerState. The PlayerState in Pawn is only assigned for as long as a Controller has possessed that Pawn, otherwise it’s a nullptr.

- A list of all current PlayerState instances (eg. all players currently in the match) is available via GameState->PlayerArray.

**Spawned by:**
- Class to spawn is assigned in GameMode (PlayerStateClass) and spawned by AController::InitPlayerState()

**From:** https://www.tomlooman.com/ue4-gameplay-framework/

# AHUD
The user interface class. Has a lot of ‘Canvas’ code which is the user interface drawing code that came before UMG – which took over most of today’s interface drawing.

Class only exists on the client. Not suitable for replication. Is owned by the PlayerController.

**Accessing HUD:**
- **PlayerController->GetHUD();** - Available in your local PlayerController

**Spawned by:**
- Spawned inside of PlayerController that owns the HUD via SpawnDefaultHUD (spawns generic AHUD), then overriden by the GameModeBase via InitializeHUDForPlayer with the HUD class specified in GameModeBase.

**From:** https://www.tomlooman.com/ue4-gameplay-framework/

# APawn
The physical and visual representation of what the player (or AI) is controlling. This can be a vehicle, a warrior, a turret, or anything that presents your game’s character. A common sub-class for Pawn is Character which implements a SkeletalMesh and more importantly a CharacterMovementComponent with MANY options for fine-tuning how your player can traverse the environment using common shooter type movement.

In multiplayer games each Pawn instance is replicated to other clients. This means in 4 player game, both the server and each of the clients all have 4 pawn instances. It is not uncommon to “kill” a Pawn instance when the player dies, and to spawn a fresh instance on player respawn. Keep this in mind for storing your data you want to keep beyond the single life of a player (or refrain from using this pattern altogether and keeping the pawn instance alive instead)

**Accessing Pawns:**
- **PlayerController->GetPawn();** - Only when the PC currently has a pawn possessed
- **GetWorld()->GetPawnIterator();** - GetWorld is available in any Actor instance, returns ALL pawns including AI you may have.

**Spawned by:** 
- GameModeBase spawns the Pawn via SpawnDefaultPawnAtTransform, the GameModeBase class also specifies which Pawn class to spawn.

**From:** https://www.tomlooman.com/ue4-gameplay-framework/

# AGameState
Similar to PlayerState, but for clients about GameMode information. Since GameMode instance doesn’t exist on clients, but only on server this is a useful container to replicate information like match time elapsed or team scores etc.

Comes in two variations, GameState and GameStateBase where GameState handles the extra variables required by GameMode (as opposed to GameModeBase)

**Accessing GameStateBase:**
- **World->GetGameState<T>();** - where T is the class to cast to eg. GetGameState<AGameState>()
- **MyGameMode->GetGameState();** - stored and available in your gamemode instance (only relevant on server which owns the only instance of GameMode), clients should use the above call instead.

**Personal Remarks:**
- Use GameStateBase over GameState unless your gamemode derives from GameMode instead of GameModeBase.

**From:** https://www.tomlooman.com/ue4-gameplay-framework/
